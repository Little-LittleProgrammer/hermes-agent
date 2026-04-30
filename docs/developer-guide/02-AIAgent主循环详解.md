# AIAgent 主循环详解

## 1. 概述

`run_agent.py` 是整个 Hermes Agent 的"心脏"，文件约 680K。`AIAgent` 类实现了完整的**自动工具调用循环（agentic loop）**，采用 **ReAct 模式**：思考 → 行动 → 观察 → 思考 → ...，直到模型决定给出最终答案。

它不是一次简单的 LLM API 调用，而是一个**自驱动的问题解决引擎**。

## 2. 核心类：AIAgent

### 2.1 初始化：组装全部依赖

```python
class AIAgent:
    def __init__(
        self,
        model,           # 模型名称
        base_url,        # API 端点
        api_key,         # API 密钥
        toolsets,        # 启用的工具集
        task_id,         # 会话任务 ID
        system_prompt,   # 额外系统提示词
        max_turns,       # 最大工具调用轮数
        ...
    ):
```

初始化做了几件关键的事：

| 步骤 | 说明 |
|---|---|
| **凭证管理** | 通过 `CredentialPool` 创建 OpenAI 客户端，支持多凭证池故障转移 |
| **传输适配器选择** | 根据 `api_mode` 选择对应的 transport（anthropic/chat_completions/bedrock/codex） |
| **工具定义加载** | 调用 `get_tool_definitions()` 筛选可用的工具 schema |
| **记忆管理初始化** | 创建 `MemoryManager`；每轮对话开始时再调用 `prefetch_all()` |
| **提示词组装** | 构建完整的系统提示词（身份+技能+平台+记忆+上下文文件） |
| **上下文长度检测** | `fetch_model_metadata()` 探测模型的上下文窗口限制 |

### 2.2 数据流：一个完整的对话轮次

```
run_conversation(user_message)
  │
  ├─ 前置处理
  │   ├── 1. subdirectory_hints.on_pre_turn()          # 目录提示追踪
  │   ├── 2. memory_manager.queue_prefetch_all(msg)    # 记忆预取
  │   ├── 3. 将 user message 转为 OpenAI 格式消息
  │   └── 4. 检查上下文窗口是否超出（触发压缩）
  │
  ├─ 主循环 (agentic loop)
  │   ├── 5. prompt_builder 组装完整消息列表
  │   ├── 6. transport.build_kwargs() 转为提供商格式
  │   ├── 7. 调用 LLM API（流式）
  │   ├── 8. transport.normalize_response() 统一化响应
  │   ├── 9. 检查是否有 tool_calls
  │   │   ├── 有 → registry.dispatch() 执行工具 → 结果回传 → 回到步骤5
  │   │   └── 无 → 退出循环，返回最终文本
  │   └── 10. enforce_turn_budget() 检查轮数上限
  │
  └─ 后置处理
      ├── 11. memory_manager.sync_all(msg, response)   # 同步记忆
      ├── 12. title_generator 生成会话标题
      └── 13. 返回最终 response
```

### 2.3 关键设计：流式处理与中断

AIAgent 支持真正的流式响应：

```
SSE 流 → 逐 chunk 解析 → 实时发送到 stream_consumer → 分发到各平台
         │
         └─ 累积到完整消息 → 解析 tool_calls → 执行工具 → 继续循环
```

**中断支持**：用户可以通过 `/interrupt` 或平台特定方式（如 Telegram inline button）中断正在执行的工具或 LLM 调用。中断检查点散布在循环的每个关键步骤：

- LLM 调用前：检查 `self._interrupt_requested`
- 工具执行前/后
- 每轮循环开始

### 2.4 错误恢复与故障转移

```
LLM API 调用失败
  │
  ├─ classify_api_error() 分类错误类型
  │   ├── rate_limit → jittered_backoff + 切换凭证池
  │   ├── context_too_long → 触发自动压缩
  │   ├── server_error → 重试（指数退避）
  │   └── auth_error → 标记凭证失效，切换下一个
  │
  └─ 重试次数耗尽 → 返回友好错误给用户
```

错误分类器（`agent/error_classifier.py`）能识别 20+ 种不同的 API 错误模式，每种都有对应的恢复策略。

## 3. AIAgent 的状态管理

### 3.1 每个实例持有的状态

| 状态 | 类型 | 生命周期 |
|---|---|---|
| `self.messages` | `List[Dict]` | 整个会话的对话历史 |
| `self.todo_store` | `TodoStore` | 当前会话的任务列表 |
| `self._memory_manager` | `MemoryManager` | 记忆提供商的编排器 |
| `self.conversation_id` | `str` | 会话唯一标识 |
| `self.turn_count` | `int` | 当前工具调用轮数 |
| `self._completion_model` | `str` | 实际使用的模型名称 |

### 3.2 工具调用循环（最核心逻辑）

```python
while self.turn_count < self.max_turns:
    # 1. 修复/整理消息与工具调用结构
    self._sanitize_tool_call_arguments(messages, logger=logger)
    
    # 2. 格式转换 + 调用
    transport = self._get_transport()
    kwargs = transport.build_kwargs(messages, tools, model)
    
    # 3. 流式调用 LLM
    response = self._interruptible_streaming_api_call(kwargs)
    
    # 4. 解析响应
    normalized = transport.normalize_response(response)
    
    # 5. 检查是否完成
    if normalized.text and not normalized.tool_calls:
        return normalized.text  # LLM 认为任务完成
    
    # 6. 执行工具
    self._execute_tool_calls(assistant_message, messages, task_id)
    
    self.turn_count += 1
```

上面是压缩后的主路径示意；真实实现还包含压缩重试、截断 tool call 修复、provider fallback、budget grace call、并发工具调用、轨迹保存等分支。

## 4. 辅助层（Auxiliary Client）

`agent/auxiliary_client.py`（约 152K）提供了独立的 LLM 客户端，用于**非主对话**的辅助任务：

- 上下文压缩摘要生成
- 标题生成
- 智能命令审批（判断命令是否危险）
- 错误分析
- 子代理的独立 LLM 调用

它与主 AIAgent 解耦，使用独立的客户端实例，避免污染主对话的状态。

## 5. 性能优化点

| 优化 | 说明 |
|---|---|
| **提示缓存** | Anthropic 的 prompt caching 减少重复 prompt 的 token 消耗 |
| **持久事件循环** | `_get_tool_loop()` 维护持久化的 asyncio 事件循环，避免每次创建/销毁 |
| **客户端 LRU 缓存** | Gateway 中对 AIAgent 实例做 LRU 缓存，避免频繁重建（上限 128 个，空闲 TTL 1 小时） |
| **工具 schema 懒加载** | 只有当前启用的 toolset 的工具才会生成 schema |

## 6. 关键文件索引

| 文件 | 大小 | 角色 |
|---|---|---|
| `run_agent.py` | ~680K | AIAgent 主类 + 主循环 |
| `agent/auxiliary_client.py` | ~152K | 辅助 LLM 调用（摘要、标题等） |
| `agent/error_classifier.py` | ~37K | API 错误分类与恢复策略 |
| `agent/retry_utils.py` | ~2K | 重试工具（退避、抖动） |
| `agent/subdirectory_hints.py` | ~8K | 子目录提示追踪 |
