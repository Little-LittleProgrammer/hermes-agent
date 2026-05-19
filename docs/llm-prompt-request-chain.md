# Hermes Agent LLM 提示词链路

本文只整理 Hermes Agent 中和 LLM 请求提示词相关的内容：提示词来源、注入位置、进入哪个消息角色、是否持久化，以及哪些辅助 LLM 请求有独立提示词。

不逐字展开所有 `skills/**/SKILL.md` 正文，因为技能是运行时内容。本文只说明技能索引和完整技能内容在什么情况下进入提示词。

## 主会话提示词链路

主会话入口最终都会调用 `AIAgent.run_conversation()`：

- CLI/TUI/API/Gateway/Cron 收到用户输入。
- 创建或复用 `AIAgent`。
- 把用户消息追加到会话历史。
- 构建或复用缓存的基础 system prompt。
- 每次 API 调用前复制一份 API-only messages，临时注入动态上下文。
- 调用模型；如果模型返回工具调用，执行工具并把结果追加到历史，再进入下一轮。

关键文件：

- `run_agent.py`
- `agent/prompt_builder.py`
- `gateway/run.py`
- `gateway/session.py`
- `cron/scheduler.py`
- `agent/skill_commands.py`
- `tools/delegate_tool.py`

## 基础 System Prompt

`AIAgent._build_system_prompt()` 负责组装基础 system prompt，并缓存在 `_cached_system_prompt`。它通常在一个会话内保持稳定，压缩上下文后才会重建。

组装顺序如下。

1. 身份提示词
   - 来源：`agent/prompt_builder.py`
   - 优先读取 `HERMES_HOME/SOUL.md`。
   - 如果没有 `SOUL.md`，使用 `DEFAULT_AGENT_IDENTITY`。
   - `SOUL.md` 会先做 prompt injection 扫描，并做长度截断。

2. Hermes 自身帮助提示
   - 常量：`HERMES_AGENT_HELP_GUIDANCE`
   - 内容含义：当用户询问 Hermes Agent 的配置、安装、使用时，先加载 `hermes-agent` skill。

3. 工具相关行为提示
   - 只有对应工具在当前 tool surface 中存在时才注入。
   - `MEMORY_GUIDANCE`：指导如何保存长期记忆。
   - `SESSION_SEARCH_GUIDANCE`：用户提到历史会话时先用 `session_search`。
   - `SKILLS_GUIDANCE`：复杂任务、修复经验、技能过期时维护 skill。
   - `KANBAN_GUIDANCE`：Kanban worker 的任务执行协议。

4. Nous 订阅能力提示
   - 函数：`build_nous_subscription_prompt()`
   - 只有 Nous managed tools 启用且当前工具相关时才注入。
   - 告诉模型哪些能力已由 Nous 订阅提供，不要重复向用户索要 API key。

5. 工具使用强制提示
   - 常量：`TOOL_USE_ENFORCEMENT_GUIDANCE`
   - 注入条件由 `agent.tool_use_enforcement` 和模型名决定。
   - 作用：要求模型不要只说计划，必须直接调用工具完成任务。
   - Gemini/Gemma 额外注入 `GOOGLE_MODEL_OPERATIONAL_GUIDANCE`。
   - GPT/Codex 额外注入 `OPENAI_MODEL_EXECUTION_GUIDANCE`。

6. 调用方传入的 `system_message`
   - 来源：`run_conversation(system_message=...)`
   - 会进入缓存的基础 system prompt。

7. 内置 Memory/User 快照
   - 来源：`tools/memory_tool.py`
   - `MemoryStore.format_for_system_prompt("memory")`
   - `MemoryStore.format_for_system_prompt("user")`
   - 这是会话开始时冻结的记忆快照。
   - 记忆写入前会扫描 prompt injection / exfiltration 风险。

8. 外部 memory provider 的 system block
   - 来源：`agent/memory_manager.py`
   - 每个 provider 可提供 `system_prompt_block()`。
   - 这类静态 provider 指令进入 system prompt。
   - 动态召回内容不在这里注入，而是在每次 API 调用时追加到当前用户消息。

9. Skills 索引提示
   - 函数：`build_skills_system_prompt()`
   - 当 `skills_list`、`skill_view` 或 `skill_manage` 可用时注入。
   - 内容包括：
     - “回复前必须扫描技能列表”的强制说明。
     - 相关 skill 必须用 `skill_view(name)` 加载。
     - 可用 skill 的分类、名称、描述。
   - 这里只是索引，不是完整 `SKILL.md` 正文。

10. 项目上下文文件
    - 函数：`build_context_files_prompt()`
    - 未设置 `skip_context_files=True` 时注入。
    - 只加载一个项目上下文来源，优先级：
      1. `.hermes.md` / `HERMES.md`
      2. `AGENTS.md`
      3. `CLAUDE.md`
      4. `.cursorrules` / `.cursor/rules/*.mdc`
    - 每个文件会扫描 prompt injection，并限制长度。

11. 会话元信息
    - 会话开始时间。
    - 可选 session id。
    - 当前 model。
    - 当前 provider。
    - Alibaba provider 会额外注入真实模型身份提示，避免 API 返回固定模型名导致模型自报错误。

12. 环境与平台提示
    - `build_environment_hints()`：例如 WSL 路径说明。
    - `PLATFORM_HINTS`：按平台注入渲染、附件、消息风格提示。
    - 覆盖 CLI、Telegram、Discord、Slack、Signal、Email、Cron、SMS、Matrix、Feishu、WeCom、Yuanbao、API Server 等。
    - 插件平台也可以提供 `platform_hint`。

## 每次 API 调用临时注入的提示词

这些内容只进入本次 API 请求，不写入会话历史，也不进入 `_cached_system_prompt`。

1. `ephemeral_system_prompt`
   - 在 API 调用前追加到基础 system prompt 后面。
   - 用途：
     - 用户配置的 personality / system prompt。
     - Gateway 当前会话上下文。
     - 平台 channel prompt。
     - API Server 的 `instructions`。
     - Delegate 子代理任务提示。
   - 不持久化。

2. `prefill_messages`
   - 插入在 system prompt 之后、历史消息之前。
   - 用于少样本 priming、Cron/TUI 某些流程。
   - 不持久化。

3. 外部 Memory 动态召回
   - `MemoryManager.prefetch_all(original_user_message)` 在工具循环前执行一次。
   - 结果用 `build_memory_context_block()` 包起来。
   - 追加到当前用户消息，而不是 system prompt。

4. Plugin `pre_llm_call` 上下文
   - 插件 hook 可以返回字符串或 `{context: ...}`。
   - 追加到当前用户消息。
   - 不追加到 system prompt，目的是保持 system prefix 可缓存。

5. `/steer` 中途提示
   - 用户在模型运行中发送 steer 时，文本会被追加到最近的 tool 结果里。
   - 下一次 API 调用时模型会看到这段 “User guidance”。

6. Anthropic prompt cache 标记
   - `apply_anthropic_cache_control()` 可能给 system prompt 和最近消息添加 `cache_control`。
   - 这是缓存标记，不是新自然语言提示词。

## 入口侧提示词

### Gateway

Gateway 会构建 `combined_ephemeral`，作为 `ephemeral_system_prompt` 传给 `AIAgent`。

来源顺序：

1. 当前会话上下文
   - 函数：`build_session_context_prompt(context)`
   - 内容包括：
     - 当前平台、聊天、线程、用户。
     - 平台行为说明。
     - 已连接平台。
     - Home channels。
     - Cron/计划任务可投递目标。
   - 支持在安全平台上做 PII redaction。

2. Channel prompt
   - 函数：`resolve_channel_prompt()`
   - 来源：平台配置里的 `channel_prompts`。
   - Discord、Slack、Mattermost、Telegram、Yuanbao 等入口可传入。

3. 用户配置的 agent prompt
   - 来源：
     - `HERMES_EPHEMERAL_SYSTEM_PROMPT`
     - `agent.system_prompt`

Gateway 还可能在用户消息正文前注入提示性上下文：

- 回复引用：把 “正在回复哪条消息” 作为文本上下文加入。
- 文本文档附件：注入 `[Content of filename]: ...`。
- Sticker：注入 sticker 的视觉描述。
- 多用户会话：给用户消息加 `[sender name]` 前缀。
- Yuanbao 群聊：注入“机器人身份”和“只回答当前被 @ 的新消息”的 channel prompt。

### CLI / TUI

提示词来源：

- 用户配置的 personality/system prompt 进入 `ephemeral_system_prompt`。
- Slash skill 会通过 `agent/skill_commands.py` 转成一段用户消息。
- `@` context references 会被展开后注入用户输入。
- `/steer` 会在运行中注入下一轮 API 可见的 user guidance。

### API Server

提示词来源：

- Chat-compatible system messages。
- Responses-style `instructions`。

这些会映射到 `ephemeral_system_prompt`。

### Cron

`cron/scheduler.py::_build_job_prompt()` 会构建 Cron 任务的用户消息。

注入内容：

- 固定 Cron 运行提示：
  - 当前是 scheduled cron job。
  - 最终响应会自动投递。
  - 不要自己调用 `send_message` 投递。
  - 如果没有新内容，必须只返回 `[SILENT]`。
- Pre-run script 成功输出：`## Script Output`。
- Pre-run script 失败输出：`## Script Error`。
- 上游 job 输出：`## Output from job ...`。
- 配置的 skills：完整 skill 内容会被展开后放入 prompt。

Cron agent 还会获得平台提示 `cron`，说明没有用户在线，不能提问，需要自主完成。

## Skill 相关提示词

Skill 有三种进入 LLM 的方式。

1. System prompt 中的 skill 索引
   - 只包含分类、名称、描述和强制加载说明。
   - 不包含完整 `SKILL.md`。

2. 模型调用 `skill_view`
   - `skill_view(name)` 返回完整 skill 内容。
   - 作为 tool result 进入后续上下文。

3. Slash / preload / cron skill
   - `agent/skill_commands.py` 或 `cron/scheduler.py` 主动加载完整 skill。
   - 生成一段用户消息，开头说明：
     - 用户调用了这个 skill。
     - 用户希望模型遵守 skill 指令。
     - 完整 skill 内容在下方。
   - 还会附加：
     - Skill 目录绝对路径。
     - 配置值。
     - setup note。
     - supporting files 提示。

## 子代理提示词

文件：`tools/delegate_tool.py`

`delegate_task` 会创建新的 `AIAgent`，并把子任务提示放入子代理的 `ephemeral_system_prompt`。

子代理提示词包含：

- “你是一个 focused subagent”。
- 具体 delegated task。
- 可选 context。
- 可选 workspace path。
- 完成后需要汇报：
  - 做了什么。
  - 发现或完成了什么。
  - 创建或修改了哪些文件。
  - 遇到哪些问题。
- workspace 规则：不要假设仓库在 `/workspace/...`。

如果子代理角色是 orchestrator，还会额外注入：

- 什么时候可以继续 delegate。
- 什么时候不应该 delegate。
- 子代理深度限制。
- 最终需要综合下游结果再回复父代理。

子代理仍然会获得正常 Hermes 基础 system prompt；上述 delegated task 是额外的 ephemeral prompt。

## 辅助 LLM 请求提示词

以下请求通常不走主会话提示词链路，而是用 `call_llm()` / `async_call_llm()` 或单独创建 `AIAgent`。

### 上下文压缩

文件：`agent/context_compressor.py`

提示词要求模型作为 summarization agent 创建 context checkpoint。

核心要求：

- 输出会注入给另一个继续会话的 assistant。
- 不要回答会话里的问题。
- 只输出结构化 summary。
- 使用用户原本语言。
- 不保留 API key、token、password、secret、credential，统一替换为 `[REDACTED]`。

结构包括：

- `Active Task`
- `Completed Actions`
- `Active State`
- `In Progress`
- `Blocked`
- `Key Decisions`
- `Resolved Questions`
- `Pending User Asks`
- `Relevant Files`
- `Remaining Work`
- `Critical Context`

如果是迭代压缩，会把 previous summary 和 new turns 一起给模型，并要求更新摘要。

### Trajectory 压缩

文件：`trajectory_compressor.py`

提示词要求：

- 总结 agent conversation turns。
- 这个 summary 会替换原始 turns。
- 从中立视角描述 assistant 做了什么、学到了什么。
- 包含工具调用、搜索、文件操作、结果、决策、文件名、值、输出。
- 输出必须以 `[CONTEXT SUMMARY]:` 开头。

### 达到最大迭代次数时的总结

文件：`run_agent.py`

当达到最大 tool-calling iterations 时，会追加一条用户消息：

- 要求模型总结已经发现和完成的内容。
- 明确要求不要再调用工具。

随后使用当前 provider 再请求一次模型，且移除 tools。

### 会话标题生成

文件：`agent/title_generator.py`

System prompt 要求：

- 为第一轮 user/assistant exchange 生成 3-7 个词的短标题。
- 只返回标题。
- 不要引号、句末标点、前缀。

User prompt 包含截断后的 User 与 Assistant 片段。

### Session Search 摘要

文件：`tools/session_search_tool.py`

System prompt 要求模型回顾过去会话 transcript，围绕 search topic 总结：

- 用户当时想做什么。
- 做了哪些动作以及结果。
- 关键决策、方案、结论。
- 重要命令、文件、URL、技术细节。
- 未解决事项。

User prompt 包含：

- Search topic。
- Session source。
- Session date。
- Conversation transcript。

### Web / Browser 内容摘要

文件：

- `tools/web_tools.py`
- `tools/browser_tool.py`

提示词类型：

- Web full-document prompt：作为内容分析专家，保留关键摘录、事实、代码片段、行动信息，并输出 markdown summary。
- Web chunk prompt：只处理大文档的一段，提取关键事实，不写介绍/结论。
- Synthesis prompt：把多个 section summaries 合成一个去重后的综合 summary。
- Browser snapshot prompt：从 accessibility snapshot 中提取和用户任务相关的内容，并保留交互元素 ref ID。

### Vision 提示词

文件：

- `tools/vision_tools.py`
- `tools/browser_camofox.py`
- `gateway/sticker_cache.py`

提示词类型：

- `vision_analyze`：完整描述并解释图片，然后回答用户问题。
- Browser screenshot：分析浏览器截图并回答问题。
- Sticker：用 1-2 句话客观描述 sticker 的角色、动作、情绪。

### Mixture of Agents

文件：`tools/mixture_of_agents_tool.py`

提示词类型：

- Reference models：直接收到原始 user prompt。
- Aggregator：收到多个模型响应，要求批判性评估偏差和错误，综合成一个高质量回答。

### Smart Approval

文件：`tools/approval.py`

提示词要求模型作为 security reviewer 判断一个命令：

- `APPROVE`：明确安全。
- `DENY`：确实危险。
- `ESCALATE`：不确定。

输出必须是三者之一。

### 后台 Memory / Skill Review

文件：`run_agent.py`

后台 review 会创建单独 `AIAgent`，读取对话并决定是否写 memory 或维护 skills。

提示词包括：

- `_MEMORY_REVIEW_PROMPT`
  - 查找用户身份、偏好、个人细节、行为期待。
  - 值得保存就调用 memory tool。
  - 没有则说 `Nothing to save.`。

- `_SKILL_REVIEW_PROMPT`
  - 主动更新 skill library。
  - 把可复用工作流、修复经验、用户纠正写入 class-level skills。
  - 优先 patch 当前加载过的 skill，其次 patch umbrella skill，再写 support file，最后才创建新 skill。

- `_COMBINED_REVIEW_PROMPT`
  - 同时处理 memory 和 skills。

### Curator

文件：`agent/curator.py`

`CURATOR_REVIEW_PROMPT` 要求模型作为 Hermes background skill curator：

- 做 class-level skill consolidation。
- 不碰 bundled / hub-installed skills。
- 不删除 skill，最多 archive。
- 跳过 pinned skill。
- 按内容重叠判断是否合并，不按 usage count。
- 目标是减少一堆窄的一次性 skill，形成 umbrella skills。

### Feishu Comment Agent

文件：`gateway/platforms/feishu_comment.py`

评论事件可创建独立 `AIAgent`。

提示词来源：

- 当前评论 prompt。
- 历史 comment conversation。
- 线程内注入的 Lark client 只影响工具执行，不是自然语言提示词。

### MCP Sampling

文件：`tools/mcp_tool.py`

MCP server 可以发起 sampling request。

提示词来源：

- MCP server 提供的 messages。
- Hermes 只是转发给 `call_llm()`。
- 这类 prompt 应视为外部输入。

## Prompt 安全处理

和提示词直接相关的安全处理：

- Context files：`agent/prompt_builder.py::_scan_context_content()`
- Memory entries：`tools/memory_tool.py`
- Cron prompts：`tools/cronjob_tools.py`
- Skills：`tools/skills_tool.py`、`tools/skills_guard.py`
- MCP tool descriptions：`tools/mcp_tool.py`
- Web/browser extraction：发送给辅助 LLM 前会 redaction。
- `@` context references：注入前有 token 限制。

这些处理降低风险，但不把外部文本变成可信指令。整体设计上，Hermes 把稳定 system prompt、临时 system prompt、动态召回、用户消息、tool result 分层处理，便于控制提示词来源和缓存边界。
