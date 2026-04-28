# 工具系统 — ToolRegistry

## 1. 设计理念

Hermes 的工具系统采用**自注册 + 自动发现 + 运行时调度**架构。三个核心原则：

1. **工具与核心解耦**：添加工具只需创建一个文件，无需修改核心代码
2. **声明式注册**：每个工具的 schema、handler、元数据由工具自己声明
3. **动态热更新**：MCP 工具可以在运行时注册/注销

## 2. 系统架构

```
tools/registry.py (ToolRegistry 单例)
  ↑ register()                      ↑ get_definitions()
tools/*.py (50+ 工具模块)           model_tools.py (编排层)
                                        ↑
                                   run_agent.py / cli.py (消费者)
```

### 2.1 导入链（避免循环导入）

```
tools/registry.py  ←── 零依赖（纯数据结构 + AST 分析）
       ↑
tools/*.py  ←── import registry; registry.register(...)
       ↑
model_tools.py  ←── 触发工具发现 + 提供公共 API
       ↑
run_agent.py, cli.py, gateway/run.py  ←── 消费者
```

## 3. ToolEntry — 工具元数据

```python
class ToolEntry:
    __slots__ = (
        "name",          # 工具唯一名称
        "toolset",       # 所属工具集
        "schema",        # JSON Schema 定义
        "handler",       # 执行函数
        "check_fn",      # 可用性检查（运行时）
        "requires_env",   # 依赖的环境变量
        "is_async",      # 是否异步处理
        "description",   # 描述
        "emoji",         # 图标
        "max_result_size_chars",  # 结果大小上限
    )
```

## 4. ToolRegistry — 核心注册中心

### 4.1 自动发现（AST 级别扫描）

```python
def discover_builtin_tools(tools_dir):
    """扫描 tools/ 目录，分析 AST 找到 registry.register(...) 调用的模块"""
    for path in tools_dir.glob("*.py"):
        if not path.name.startswith("__"):
            source = path.read_text()
            tree = ast.parse(source)
            if has_registry_register_call(tree):
                importlib.import_module(f"tools.{path.stem}")
```

**为什么用 AST 分析而不是直接 import？**

- 避免导入不完整的模块（缺少依赖时）
- 快速判断模块是否包含工具注册（一行 AST 遍历 vs 整个模块执行）
- 跳过纯工具库/辅助模块

### 4.2 注册方法

```python
def register(self, name, toolset, schema, handler, check_fn=None,
             requires_env=None, is_async=False, description="", ...):
```

**安全保护**：

- 同名工具拒绝覆盖（除非是 MCP→MCP 的合法刷新）
- 记录所有注册冲突的警告日志
- 线程安全锁：`threading.RLock()` 保护读写并发

### 4.3 注销方法

```python
def deregister(self, name):
    """MCP 动态工具发现调用 — 当服务器发送 tools/list_changed 时使用"""
    # 同时清理：
    #   1. tool entry
    #   2. toolset check（如果该 toolset 没有其他工具了）
    #   3. toolset alias（同样清理无引用的别名）
```

### 4.4 Schema 检索

```python
def get_definitions(self, tool_names, quiet=False):
    """返回 OpenAI 格式的工具 schema，只包含 check_fn 通过的工具"""
```

关键设计：每个工具的 `check_fn` 结果被缓存（同一次调用的同一 check_fn 只执行一次），避免重复检查。

### 4.5 工具调度

```python
def dispatch(self, name, args, **kwargs):
    """执行工具 - 自动处理同步/异步桥接"""
    entry = self.get_entry(name)
    if entry.is_async:
        return _run_async(entry.handler(args, **kwargs))
    return entry.handler(args, **kwargs)
```

所有异常被捕获并转为 JSON 错误格式返回，确保即使工具崩溃也不会中断整个对话。

## 5. 工具分类（按 toolset）

### 5.1 核心工具集

| 分类 | Toolset | 主要工具 |
|---|---|---|
| **终端** | `terminal` | `terminal`（6 种后端）, `process` |
| **文件** | `file_ops` | `read_file`, `write_file`, `patch`, `search_files` |
| **浏览器** | `browser` | `browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type` 等 |
| **Web** | `web` | `web_search`（4 种引擎）, `web_extract` |
| **消息** | `messaging` | `send_message`（跨平台消息推送） |
| **视觉** | `vision` | `vision_analyze`（图片分析） |
| **代码** | `code` | `execute_code`（沙箱代码执行） |
| **委派** | `delegate` | `delegate_task`（子代理委派） |
| **技能** | `skills` | `skills_list`, `skill_view`, `skill_manage` |
| **计划** | `planning` | `todo`（任务管理） |
| **记忆** | `memory` | `memory`（记忆读写）, `session_search` |
| **交互** | `interaction` | `clarify`（向用户提问）, `interrupt` |
| **定时** | `cron` | `cronjob`（定时任务管理） |
| **MCP** | `mcp-*` | 动态注册的外部工具 |

### 5.2 非工具集的"工具模块"

有些 `tools/` 下的文件不是工具，而是**工具基础设施**：

- `process_registry.py` — 进程管理器（所有工具共享）
- `file_state.py` — 文件操作状态缓存
- `tool_result_storage.py` — 工具结果持久化
- `approval.py` — 危险命令审批
- `budget_config.py` — Token/轮数预算配置

## 6. 工具开发模式

### 6.1 创建新工具的标准模板

```python
# tools/my_new_tool.py
import json
from tools.registry import registry, tool_error, tool_result

def _check_availability():
    """运行时检查 - 例如检查 API key 是否存在"""
    import os
    return bool(os.environ.get("MY_API_KEY"))

def my_tool_handler(args, **kwargs):
    param = args.get("param_name", "")
    try:
        # 工具逻辑
        result = do_something(param)
        return tool_result({"output": result})
    except Exception as e:
        return tool_error(str(e))

registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema={
        "name": "my_tool",
        "description": "描述工具做什么，何时使用，何时不使用",
        "parameters": {
            "type": "object",
            "properties": {
                "param_name": {
                    "type": "string",
                    "description": "参数说明"
                }
            },
            "required": ["param_name"]
        }
    },
    handler=my_tool_handler,
    check_fn=_check_availability,
    requires_env=["MY_API_KEY"],
    description="Short description for UI",
    emoji="🔧",
)
```

### 6.2 关键要点

- **schema 质量决定调用质量**：LLM 靠 schema description 来判断何时调用工具，写清楚"何时使用"和"何时不使用"
- **handler 返回 JSON 字符串**：统一使用 `tool_result()` 和 `tool_error()` 辅助函数
- **check_fn 不能抛异常**：会被静默吞掉并视为不可用
- **is_async=True** 标记异步 handler，dispatch 自动桥接

## 7. 关键文件索引

| 文件 | 大小 | 角色 |
|---|---|---|
| `tools/registry.py` | ~19K | ToolRegistry 单例 + 工具发现 |
| `model_tools.py` | ~26K | 编排层 + 异步桥接 + 公共 API |
| `toolsets.py` | ~30K | 工具集定义与解析 |
| `tools/terminal_tool.py` | ~94K | 终端工具（最大工具文件） |
| `tools/browser_tool.py` | ~116K | 浏览器自动化 |
| `tools/web_tools.py` | ~88K | Web 搜索与抓取 |
| `tools/file_tools.py` | ~52K | 文件操作 |
| `tools/mcp_tool.py` | ~127K | MCP 客户端（动态工具） |
| `tools/skills_hub.py` | ~119K | 技能市场（安装/卸载） |
| `tools/delegate_tool.py` | ~107K | 子代理委派 |
| `tools/approval.py` | ~52K | 命令审批系统 |
| `tools/todo_tool.py` | ~10K | 任务管理 |
