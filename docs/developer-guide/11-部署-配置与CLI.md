# 部署、配置与 CLI

## 1. 多种运行模式

Hermes Agent 支持从本地终端到无服务器计算的完整部署谱系：

```
本地开发 ←→ 容器化 ←→ 远程 SSH ←→ 无服务器 ←→ HPC 集群
  CLI       Docker    SSH 跳板    Modal/       Slurm
                                 Daytona
```

### 1.1 模式对比

| 模式 | 启动方式 | 适用场景 |
|---|---|---|
| **CLI 交互** | `hermes` | 本地开发、日常使用 |
| **单次查询** | `hermes -q "问题"` | 脚本集成、批处理 |
| **Gateway** | `python -m gateway.run` | 生产部署、多平台接入 |
| **TUI** | `hermes --tui` | Ink 终端 UI；Python 侧由 `tui_gateway` 提供 JSON-RPC 后端 |
| **ACP** | `python -m acp_adapter` | IDE 集成 |
| **Cron** | `python -m cron.scheduler` | 定时任务专用 |
| **Docker** | `docker-compose up` | 容器化部署 |
| **Modal** | 通过 Modal CLI | 无服务器 GPU |
| **Daytona** | 通过 Daytona CLI | 云开发环境 |
| **SSH** | `hermes --ssh user@host` | 远程机器执行 |

## 2. CLI 架构 (cli.py ~600K+)

### 2.1 TUI 组件

```python
# cli.py 使用 prompt_toolkit 构建完整的终端 UI
from prompt_toolkit import Application, Layout
from prompt_toolkit.widgets import TextArea
from prompt_toolkit.layout import HSplit, Window

class HermesCLI:
    def __init__(self):
        self.app = Application(
            layout=Layout(
                HSplit([
                    Window(content=FormattedTextControl(...)),  # 输出区域
                    ConditionalContainer(CompletionsMenu(...)),  # 自动补全
                    TextArea(...),                               # 输入区域
                ])
            ),
            key_bindings=self._build_keymap(),
        )
```

### 2.2 关键 CLI 功能

| 功能 | 实现位置 |
|---|---|
| **多彩输出** | `hermes_cli/colors.py` + Rich 库 |
| **皮肤引擎** | `hermes_cli/skin_engine.py` — 可定制的终端外观 |
| **命令补全** | prompt_toolkit completions |
| **对话历史** | FileHistory（跨会话持久化） |
| **流式渲染** | 实时显示 LLM 的增量输出 |
| **工具执行状态** | KawaiiSpinner + 动画显示工具调用进度 |
| **ASCII Banner** | `hermes_cli/banner.py` — 启动艺术字 |

### 2.3 Slash 命令注册表 (hermes_cli/commands.py)

```python
COMMAND_REGISTRY = [
    CommandDef("new", "Start a new session", "Session", aliases=("reset",)),
    CommandDef("model", "Switch model for this session", "Configuration"),
    CommandDef("skills", "Manage skills", "Tools & Skills",
               subcommands=("list", "install", "enable", "disable")),
    ...
]

def resolve_command(name: str) -> CommandDef | None:
    """解析 canonical name、alias 和带斜杠输入。"""
```

`COMMAND_REGISTRY` 是会话内 slash commands 的单一来源。CLI help、gateway dispatch、Telegram BotCommand 菜单、Slack 子命令映射和 autocomplete 都从这里派生。`SUBCOMMANDS` 仍存在，但只是从 `CommandDef.subcommands`/`args_hint` 自动生成的补全辅助表，不是新增命令的入口。

### 2.4 命令补全 (hermes_cli/completion.py)

支持 bash/zsh/fish 的 shell 自动补全生成。

## 3. 配置系统

### 3.1 配置层次

```
环境变量 (最高优先级)
     ↓
~/.hermes/config.yaml (用户配置)
     ↓
项目根目录配置 (项目级)
     ↓
默认值 (代码内置)
```

### 3.2 配置文件结构 (~/.hermes/config.yaml)

```yaml
# 模型默认值
model:
  provider: anthropic
  model: claude-sonnet-4-6
  base_url: ""
  api_key: ""

# 工具集配置
toolsets:
  enabled: ["web", "terminal", "file_ops", ...]
  disabled: []

# 技能配置
skills:
  enabled: []
  disabled: []
  guard_agent_created: false

# 记忆配置
memory:
  provider: builtin  # builtin | honcho
  honcho:
    api_key: ""

# MCP 配置
mcp:
  servers: {}

# 网关配置
gateway:
  enabled: false
  bind: "0.0.0.0"
  telegram:
    enabled: false
    token: ""
  discord:
    enabled: false
    token: ""
  # ...更多平台配置

# 安全配置
security:
  auto_approve: false
  require_confirmation: true
  allowed_commands: []
  denied_commands: []

# 日志配置
logging:
  level: INFO
  file: ~/.hermes/logs/agent.log

# 插件配置
plugins:
  enabled: []
  disabled: []
```

### 3.3 CLI 配置管理 (hermes_cli/config.py)

```python
def load_config():
    """从多个来源加载合并配置"""
    config = default_config()
    
    # 1. 项目根目录
    project_config = Path.cwd() / ".hermes.yaml"
    if project_config.exists():
        merge(config, yaml.safe_load(project_config.read_text()))
    
    # 2. 用户配置
    user_config = get_config_path() / "config.yaml"
    if user_config.exists():
        merge(config, yaml.safe_load(user_config.read_text()))
    
    # 3. 环境变量覆盖
    apply_env_overrides(config)
    
    return config
```

## 4. 环境变量加载 (hermes_cli/env_loader.py)

```python
def load_hermes_dotenv(hermes_home, project_env):
    """加载 .env 文件链"""
    paths = []
    
    # 1. ~/.hermes/.env (用户全局环境变量)
    user_env = hermes_home / ".env"
    if user_env.exists():
        load_dotenv(user_env)
        paths.append(user_env)
    
    # 2. 项目 .env (开发回退)
    if project_env.exists():
        load_dotenv(project_env, override=False)  # 不覆盖已有值
        paths.append(project_env)
    
    return paths
```

## 5. 存储系统 (hermes_constants.py)

### 5.1 目录结构

```
~/.hermes/                    # Hermes 主目录
  ├── config.yaml              # 用户配置
  ├── .env                     # 环境变量
  ├── logs/                    # 日志
  │     ├── agent.log
  │     ├── errors.log
  │     └── gateway.log
  ├── memory/                  # 记忆存储
  │     ├── user/
  │     ├── project/
  │     ├── feedback/
  │     └── reference/
  ├── skills/                  # 用户创建的技能
  ├── skills-hub/              # Skills Hub 安装的技能
  ├── plugins/                 # 用户插件
  ├── sessions.db              # 网关会话 (SQLite)
  ├── history                  # CLI 历史
  │     └── hermes_history
  ├── backups/                 # 备份
  ├── crash_reports/           # 崩溃报告
  └── .versions/               # 版本管理
```

### 5.2 路径 API

```python
# hermes_constants.py
def get_hermes_home() -> Path:
    """返回当前 profile 的 Hermes home；默认 ~/.hermes，可由 HERMES_HOME 覆盖。"""

def display_hermes_home() -> str:
    """返回用户可读路径，用于日志/提示文案。"""

def get_default_hermes_root() -> Path:
    """返回 profile 管理根目录；profile 模式下不是当前 profile home。"""
```

代码路径必须用 `get_hermes_home()`，用户可见文案用 `display_hermes_home()`。不要在代码里硬编码 `Path.home() / ".hermes"` 或字符串 `~/.hermes`，否则会破坏 profile 隔离。

## 6. 健康检查 (hermes_cli/doctor.py)

`hermes doctor` 命令检查系统状态：

```python
def run_diagnostics():
    checks = []
    
    # 环境检查
    checks.append(check_python_version(">=3.11"))
    checks.append(check_disk_space(hermes_home))
    
    # 连接性检查
    checks.append(check_api_connectivity())
    checks.append(check_internet_connection())
    
    # 工具检查
    checks.append(check_toolset_requirements())  # 哪些 toolset 可用
    checks.append(check_environment_variables())
    
    # 权限检查
    checks.append(check_file_permissions())
    
    # 配置检查
    checks.append(validate_config_syntax())
    
    return format_report(checks)
```

## 7. 备份与恢复 (hermes_cli/backup.py)

```python
def create_backup():
    """备份当前 HERMES_HOME 到 tar.gz 归档"""
    backup_path = HERMES_HOME / "backups" / f"hermes-{timestamp}.tar.gz"
    # 打包: config.yaml, memory/, skills/, sessions.db, history
    # 排除: logs/, .versions/, __pycache__/

def restore_backup(backup_path):
    """从备份恢复"""
    # 1. 解压到临时目录
    # 2. 验证完整性
    # 3. 替换当前 HERMES_HOME 内容
    # 4. 保留当前版本作为 .backup 后缀
```

## 8. 关键文件索引

| 文件/目录 | 大小 | 角色 |
|---|---|---|
| `cli.py` | ~600K | CLI 主入口 + TUI |
| `hermes_cli/commands.py` | ~ | 子命令系统 |
| `hermes_cli/config.py` | ~ | 配置管理 |
| `hermes_cli/env_loader.py` | ~ | 环境变量加载 |
| `hermes_cli/doctor.py` | ~ | 健康检查 |
| `hermes_cli/backup.py` | ~ | 备份恢复 |
| `hermes_cli/setup.py` | ~ | 初始化向导 |
| `hermes_cli/web_server.py` | ~ | Web 服务器 |
| `hermes_cli/oneshot.py` | ~ | 单次查询模式 |
| `hermes_cli/banner.py` | ~ | ASCII 艺术 Banner |
| `hermes_cli/skin_engine.py` | ~ | 皮肤引擎 |
| `hermes_cli/main.py` | ~ | CLI 入口点 |
| `hermes_constants.py` | ~ | 全局常量 |
| `hermes_state.py` | ~ | 全局状态管理 |
| `hermes_time.py` | ~ | 时间工具 |
| `hermes_logging.py` | ~ | 日志配置 |
| `Dockerfile` | ~ | Docker 镜像 |
| `docker-compose.yml` | ~ | Docker 编排 |
| `pyproject.toml` | ~ | 项目元数据 + 依赖 |
