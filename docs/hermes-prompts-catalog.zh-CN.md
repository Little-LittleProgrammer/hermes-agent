# Hermes Agent 提示词目录

本文列出 Hermes Agent 项目中会进入 LLM 请求、影响 LLM 行为，或作为辅助 LLM 请求模板使用的提示词。文档按提示词类别整理为中文说明，并保留关键英文片段，便于定位真实语义。

说明：

- “提示词”包括 system prompt、ephemeral system prompt、user prompt 模板、tool prompt、辅助 LLM prompt、skill prompt、平台提示、上下文注入模板。
- 技能库本身包含大量 `SKILL.md` 正文。本文列出技能提示词资产的类别、数量、载入条件和典型包装语，不全文复制 186 个技能/分类文档。
- 用户配置、项目上下文文件、记忆、频道提示、cron job prompt 等是运行时动态提示词，不在仓库中固定，本文列出其载入方式。

## 1. 主会话 System Prompt 层

```text
┌──────────────────────────────────────────────┐
│ 1. 基础身份                                  │
├──────────────────────────────────────────────┤
│ 2. Hermes 自身帮助提示                       │
├──────────────────────────────────────────────┤
│ 3. 工具相关行为提示                          │
├──────────────────────────────────────────────┤
│ 4. Nous 订阅能力提示                         │
├──────────────────────────────────────────────┤
│ 5. 工具使用强制提示                          │
├──────────────────────────────────────────────┤
│ 6. 模型族特定执行提示                        │
├──────────────────────────────────────────────┤
│ 7. 调用方 system_message                     │
├──────────────────────────────────────────────┤
│ 8. Memory / User 快照                        │
├──────────────────────────────────────────────┤
│ 9. 外部 Memory Provider 静态提示             │
├──────────────────────────────────────────────┤
│ 10. Skills 索引提示                          │
├──────────────────────────────────────────────┤
│ 11. 项目上下文文件                           │
├──────────────────────────────────────────────┤
│ 12. 会话元信息                               │
├──────────────────────────────────────────────┤
│ 13. 环境提示                                 │
├──────────────────────────────────────────────┤
│ 14. 平台提示                                 │
└──────────────────────────────────────────────┘
```

### 1.1 基础身份提示词

载入条件：

- 默认每个主会话载入。
- 如果存在并允许读取 `SOUL.md`，优先使用 `SOUL.md` 作为身份提示。
- 否则使用内置默认身份。

核心内容：

```text
You are Hermes Agent, an intelligent AI assistant created by Nous Research.
You are helpful, knowledgeable, and direct.
You assist users with a wide range of tasks including answering questions,
writing and editing code, analyzing information, creative work, and executing actions via your tools.
```

中文说明：

- 定义 Hermes 是 Nous Research 创建的智能助手。
- 要求有帮助、知识丰富、直接。
- 支持问答、写代码、分析信息、创意任务和通过工具执行动作。
- 要求清晰沟通、不确定时承认、优先真正有用而不是啰嗦。

### 1.2 Hermes 自身帮助提示

载入条件：

- 每个主会话基础 system prompt 中固定加入。

核心内容：

```text
If the user asks about configuring, setting up, or using Hermes Agent itself,
load the `hermes-agent` skill with skill_view(name='hermes-agent') before answering.
Docs: https://hermes-agent.nousresearch.com/docs
```

中文说明：

- 用户问 Hermes 本身的配置、安装或使用时，必须先加载 `hermes-agent` skill。
- 附带官方文档地址。

### 1.3 Memory 使用提示

载入条件：

- 当前工具集中存在 `memory` 工具时载入。

核心内容：

```text
You have persistent memory across sessions.
Save durable facts using the memory tool: user preferences, environment details,
tool quirks, and stable conventions.
Do NOT save task progress, session outcomes, completed-work logs, or temporary TODO state to memory.
Write memories as declarative facts, not instructions to yourself.
```

中文说明：

- 告诉模型有跨会话持久记忆。
- 只保存长期有用事实：用户偏好、环境细节、工具特性、稳定约定。
- 不保存任务进度、完成记录、临时 TODO。
- 记忆必须写成陈述事实，不写成命令式自我指令。

### 1.4 Session Search 使用提示

载入条件：

- 当前工具集中存在 `session_search` 时载入。

核心内容：

```text
When the user references something from a past conversation or you suspect
relevant cross-session context exists, use session_search to recall it before
asking them to repeat themselves.
```

中文说明：

- 用户提到历史会话或可能需要跨会话上下文时，先用 `session_search` 检索，不要直接让用户重复。

### 1.5 Skills 维护提示

载入条件：

- 当前工具集中存在 `skill_manage` 时载入。

核心内容：

```text
After completing a complex task (5+ tool calls), fixing a tricky error,
or discovering a non-trivial workflow, save the approach as a skill with skill_manage.
When using a skill and finding it outdated, incomplete, or wrong,
patch it immediately with skill_manage(action='patch').
```

中文说明：

- 完成复杂任务、修复难错、发现非平凡工作流后，应保存为 skill。
- 如果已加载的 skill 过时、不完整或错误，应立即 patch。

### 1.6 Kanban 任务执行提示

载入条件：

- 当前工具集中存在 `kanban_show` 时载入。
- 通常意味着 agent 被 Kanban dispatcher 作为 board worker 启动。

核心内容摘要：

```text
# Kanban task execution protocol
You have been assigned ONE task from the shared board.
Call kanban_show() first.
Work inside $HERMES_KANBAN_WORKSPACE.
Heartbeat on long operations.
Block on genuine ambiguity.
Complete with structured handoff.
If follow-up work appears, create it; don't do it.
```

中文说明：

- 规定 board worker 的生命周期。
- 要先查看任务、在指定 workspace 工作、长任务 heartbeat、遇到真实歧义 block、完成时结构化 handoff。
- 不允许越界做后续任务；要创建子任务交给合适角色。

### 1.7 Nous 订阅能力提示

载入条件：

- Nous managed tools 启用。
- 当前工具集中存在 web、browser、image、tts、terminal/process 等相关能力。

核心内容摘要：

```text
Nous subscription includes managed web tools, image generation, OpenAI TTS,
and browser automation by default.
When a Nous-managed feature is active, do not ask the user for API keys.
Do not mention subscription unless the user asks about it or it directly solves the current missing capability.
```

中文说明：

- 告诉模型哪些能力由 Nous 订阅托管。
- 已托管时不要向用户索要 Firecrawl、FAL、OpenAI TTS、Browser-Use 等 API key。

### 1.8 工具使用强制提示

载入条件：

- 当前存在可用工具。
- `agent.tool_use_enforcement` 配置允许。
- 默认对 `gpt`、`codex`、`gemini`、`gemma`、`grok` 等模型名匹配时载入。

核心内容：

```text
You MUST use your tools to take action.
Do not describe what you would do or plan to do without actually doing it.
When you say you will perform an action, you MUST immediately make the corresponding tool call.
Never end your turn with a promise of future action.
```

中文说明：

- 要求模型不能只描述计划。
- 说要做某事就必须立即调用工具。
- 每次回复要么调用工具推进，要么给最终结果。

### 1.9 OpenAI / Codex 模型执行纪律提示

载入条件：

- 工具使用强制提示已启用。
- 当前模型名包含 `gpt` 或 `codex`。

核心内容摘要：

```text
Use tools whenever they improve correctness, completeness, or grounding.
Keep calling tools until the task is complete and verified.
NEVER answer arithmetic, hashes, current time, system state, file contents,
git history, or current facts from memory; use tools.
If required context is missing, do NOT guess or hallucinate.
```

中文说明：

- 强化工具持久性、必要前置检查、验证和反幻觉。
- 明确列出必须用工具的任务：计算、哈希、时间、系统状态、文件内容、git、实时事实。

### 1.10 Google / Gemini / Gemma 操作提示

载入条件：

- 工具使用强制提示已启用。
- 当前模型名包含 `gemini` 或 `gemma`。

核心内容摘要：

```text
Always construct and use absolute file paths.
Verify file contents and project structure before making changes.
Never assume a library is available.
Keep explanatory text brief.
Make independent tool calls in parallel.
Use non-interactive flags.
Work autonomously until fully resolved.
```

中文说明：

- 针对 Google 模型补充操作规则。
- 强调绝对路径、先验证、依赖检查、简洁、并行工具调用、非交互命令、持续完成。

### 1.11 Skills 索引提示

载入条件：

- 当前工具集中存在 `skills_list`、`skill_view` 或 `skill_manage`。

核心内容：

```text
## Skills (mandatory)
Before replying, scan the skills below.
If a skill matches or is even partially relevant to your task,
you MUST load it with skill_view(name) and follow its instructions.
Only proceed without loading a skill if genuinely none are relevant to the task.
```

中文说明：

- 提示模型回复前必须扫描技能索引。
- 相关或部分相关的技能必须加载。
- 这里注入的是技能目录索引，不是完整技能正文。

### 1.12 项目上下文文件提示

载入条件：

- 未跳过 context files。
- 当前工作目录或 Hermes home 中存在上下文文件。

可能来源：

- `SOUL.md`
- `.hermes.md`
- `HERMES.md`
- `AGENTS.md`
- `CLAUDE.md`
- `.cursorrules`
- `.cursor/rules/*.mdc`

中文说明：

- 这些文件的正文会作为项目或人格上下文注入。
- 会做注入扫描和长度截断。
- `.hermes.md` / `HERMES.md` / `AGENTS.md` / `CLAUDE.md` / Cursor rules 只取优先级命中的项目上下文。

### 1.13 会话元信息提示

载入条件：

- 每次构建基础 system prompt 时加入。

内容：

```text
Conversation started: <weekday, month day, year time>
Session ID: <session_id>
Model: <model>
Provider: <provider>
```

中文说明：

- 告诉模型会话开始时间、会话 ID、模型和 provider。
- Alibaba provider 还会补充真实模型 ID，避免模型自报被 API 返回值误导。

### 1.14 环境提示

载入条件：

- 检测到特殊执行环境时加入。

WSL 示例：

```text
You are running inside WSL (Windows Subsystem for Linux).
The Windows host filesystem is mounted under /mnt/.
When the user references Windows paths or desktop files, translate to the /mnt/c/ equivalent.
```

中文说明：

- 当前实现主要是 WSL 路径提示。
- 指导模型把 Windows 路径转换为 `/mnt/c/...`。

### 1.15 平台提示

载入条件：

- Agent 当前 `platform` 有对应平台提示。
- 插件平台也可提供 platform hint。

内置平台提示类别：

| 平台 | 提示重点 |
| --- | --- |
| CLI | 终端显示，少 Markdown，不输出 `MEDIA:/path` |
| Telegram | Markdown 支持、无表格语法、`MEDIA:/absolute/path` 发送媒体 |
| Discord | Discord 对话，可发送媒体附件 |
| Slack | Slack workspace，可上传媒体 |
| Signal | 纯文本为主，可发送媒体 |
| Email | 邮件风格，纯文本、清晰结构、不随意问候/署名 |
| Cron | 无用户在线，必须自主执行，最终响应自动投递 |
| SMS | 纯文本、简短、约 1600 字限制 |
| BlueBubbles | iMessage 风格，短消息，可媒体 |
| Mattermost | Markdown 支持，可媒体 |
| Matrix | Markdown 转 HTML，可媒体 |
| Feishu | Lark/飞书 Markdown，可媒体 |
| Weixin | 微信，紧凑聊天风格，可媒体 |
| WeCom | 企业微信，可媒体，AMR voice 限制 |
| QQBot | QQ，支持 Markdown 和 emoji，可媒体 |
| Yuanbao | 腾讯元宝，Markdown、媒体、原生贴纸工具 |
| API Server | 渲染层未知，纯文本、简短自然 |

Yuanbao 贴纸提示额外要求：

```text
When the user sends a sticker or asks you to send/reply-with a sticker,
you MUST use the sticker tools.
DO NOT draw sticker-like PNGs.
```

## 2. API 调用时临时提示词

### 2.1 Ephemeral System Prompt

载入条件：

- CLI/TUI/Gateway/API/Cron/Delegate 等入口传入临时 system prompt。
- 每次 API 调用时拼接到基础 system prompt 后面。
- 不写入会话历史，不进入缓存 system prompt。

来源：

- 用户配置的 `agent.system_prompt`。
- `HERMES_EPHEMERAL_SYSTEM_PROMPT`。
- Gateway 当前会话上下文。
- Channel prompt。
- API Server `instructions`。
- Delegate 子代理任务提示。
- Preloaded skills 包装提示。

### 2.2 Prefill Messages

载入条件：

- 配置了 prefill messages 文件或入口传入 prefill messages。
- 每次 API 调用插入在 system prompt 之后、会话历史之前。
- 不写入会话历史。

中文说明：

- 用于 few-shot、格式 priming、assistant 预填等。

### 2.3 外部 Memory 动态召回

载入条件：

- 外部 memory provider 启用。
- 当前用户消息触发 prefetch。

注入方式：

- 召回结果被包装为 memory context block。
- 追加到当前 user message，而不是 system prompt。

### 2.4 Plugin pre_llm_call 上下文

载入条件：

- 插件实现了 `pre_llm_call` hook 并返回上下文。

注入方式：

- 追加到当前 user message。
- 不追加到 system prompt，避免破坏 prompt cache 前缀。

### 2.5 Steer 中途提示

载入条件：

- 用户在模型运行中发送 steer 指令。
- 且存在可注入的最近 tool result。

注入内容：

```text
User guidance: <steer text>
```

中文说明：

- 下一次 API 调用时模型会看到该用户引导。

## 3. Gateway / 通讯入口提示词

### 3.1 当前会话上下文提示

载入条件：

- Gateway 收到消息并创建 agent 时。

模板结构：

```text
## Current Session Context

**Source:** <platform> (<chat/user/thread description>)
**Channel Topic:** <topic>
**User:** <user_name>
**Session type:** Multi-user session — messages are prefixed with [sender name].
**Connected Platforms:** local, telegram/slack/...
**Home Channels (default destinations):**
**Delivery options for scheduled tasks:**
```

平台专项内容：

- Slack：说明没有 Slack 专用 API，不要承诺搜索频道历史、pin 消息、列用户等。
- Discord：如果 Discord 工具不可用，说明只能读直接发来的消息并回复；如果可用，注入 guild/channel/thread/message IDs。
- BlueBubbles：要求 iMessage 风格，短句，空行拆 bubble。
- Yuanbao：说明可用 `send_message` 给元宝 DM 或 group。

### 3.2 Channel Prompt

载入条件：

- 平台配置中存在当前 channel/thread 对应的 `channel_prompts`。

中文说明：

- 作为 ephemeral prompt 拼入当前会话。
- 用于某个频道的专属规则、角色或输出约束。

### 3.3 Reply / Attachment / Sticker 注入

载入条件：

- 用户回复某条消息、上传可读文本附件、发送 sticker 或多用户上下文。

典型注入：

```text
[Content of filename]:
<text content>

[Replying to: "..."]

[sender name] <message>

Sticker description: <description>
```

中文说明：

- 这些通常进入 user message，而不是 system prompt。

### 3.4 Yuanbao 群聊定向提示

载入条件：

- Yuanbao 群聊消息触发 @bot guard。

核心内容：

```text
You are handling a Yuanbao group chat message.
- Your identity: user_id=<id>, @-mention name in this group=<name>
- Lines in history prefixed with [nickname|user_id] are observed group context
  and are not necessarily addressed to you.
- Treat only the current new message as a request explicitly directed at you.
```

中文说明：

- 防止模型把群历史里其他人的消息都当成对自己的请求。

## 4. Cron 提示词

### 4.1 Cron 平台提示

载入条件：

- Agent platform 为 cron。

核心内容：

```text
You are running as a scheduled cron job.
There is no user present.
You cannot ask questions, request clarification, or wait for follow-up.
Execute the task fully and autonomously.
```

### 4.2 Cron Job Prompt 包装

载入条件：

- Cron job 执行时构建用户 prompt。

固定前缀：

```text
[IMPORTANT: You are running as a scheduled cron job.
DELIVERY: Your final response will be automatically delivered to the user.
Do NOT use send_message or try to deliver the output yourself.
SILENT: If there is genuinely nothing new to report, respond with exactly "[SILENT]".]
```

可选注入：

```text
## Script Output
The following data was collected by a pre-run script. Use it as context for your analysis.

## Script Error
The data-collection script failed. Report this to the user.

## Output from job '<job_id>'
The following is the most recent output from a preceding cron job.
```

### 4.3 Cron Skill 包装提示

载入条件：

- Cron job 配置了 skills。

核心内容：

```text
[IMPORTANT: The user has invoked the "<skill_name>" skill,
indicating they want you to follow its instructions.
The full skill content is loaded below.]
```

如果 skill 缺失：

```text
[IMPORTANT: The following skill(s) were listed for this job but could not be found and were skipped...
Start your response with a brief notice...]
```

## 5. Skill 提示词资产

### 5.1 技能索引

载入条件：

- Skills 工具可用。

内容：

- 技能分类描述。
- 技能名称。
- 技能 frontmatter description。
- 必须加载相关技能的规则。

### 5.2 完整 Skill 内容

载入条件：

- 模型调用 `skill_view`。
- 用户通过 slash command 显式调用技能。
- CLI/TUI 预加载技能。
- Cron job 绑定技能。

Slash / preload 包装提示：

```text
[IMPORTANT: The user has invoked the "<skill_name>" skill,
indicating they want you to follow its instructions.
The full skill content is loaded below.]
```

Preloaded skill 包装提示：

```text
[IMPORTANT: The "<skill_name>" skill has been preloaded.
Treat its instructions as active guidance for the duration of this session.]
```

附加内容：

- Skill directory。
- Skill config。
- Setup note。
- Supporting files 列表。

### 5.3 技能资产规模

当前仓库内技能提示词资产数量：

- `skills/` 和 `optional-skills/` 下共 186 个 `SKILL.md` / `DESCRIPTION.md` 文件。
- 这些文件本身都是可被加载进 LLM 上下文的提示词资产。

按目录统计：

| 目录 | 数量 |
| --- | ---: |
| skills/apple | 5 |
| skills/autonomous-ai-agents | 5 |
| skills/creative | 20 |
| skills/data-science | 2 |
| skills/devops | 3 |
| skills/github | 7 |
| skills/gaming | 3 |
| skills/media | 6 |
| skills/mlops | 20 |
| skills/productivity | 10 |
| skills/research | 6 |
| skills/software-development | 11 |
| optional-skills/mlops | 25 |
| optional-skills/productivity | 6 |
| optional-skills/research | 8 |
| optional-skills/creative | 5 |
| optional-skills/security | 4 |
| 其他 skills / optional-skills 目录 | 45 |

## 6. 子代理提示词

载入条件：

- 主 agent 调用 delegate/subagent 能力。

Leaf 子代理核心提示：

```text
You are a focused subagent working on a specific delegated task.

YOUR TASK:
<goal>

CONTEXT:
<context>

WORKSPACE PATH:
<workspace_path>

Complete this task using the tools available to you.
When finished, provide a clear, concise summary of:
- What you did
- What you found or accomplished
- Any files you created or modified
- Any issues encountered
```

重要规则：

```text
Never assume a repository lives at /workspace/... unless explicitly given.
If no exact local path is provided, discover it first.
```

Orchestrator 子代理额外提示：

```text
You have access to delegate_task and CAN spawn your own subagents.
WHEN to delegate: independent subtasks that can run in parallel.
WHEN NOT to delegate: single-step mechanical work, trivial tasks,
or re-delegating your entire assigned goal to one worker.
Coordinate your workers' results and synthesize them before reporting back.
```

## 7. 后台 Memory / Skill Review 提示词

### 7.1 Memory Review

载入条件：

- 达到 memory review nudge 间隔。
- 当前会话完成后后台 review agent 启动。

核心内容：

```text
Review the conversation above and consider saving to memory if appropriate.
Focus on:
1. Has the user revealed things about themselves?
2. Has the user expressed expectations about how you should behave?
If something stands out, save it using the memory tool.
If nothing is worth saving, just say 'Nothing to save.' and stop.
```

### 7.2 Skill Review

载入条件：

- 达到 skill creation/update nudge 间隔。
- 或复杂任务、技能使用后触发后台 review。

核心内容摘要：

```text
Review the conversation above and update the skill library.
Be ACTIVE.
Target shape: CLASS-LEVEL skills with rich SKILL.md and references/.
Signals:
- User corrected style, tone, format, verbosity, workflow, or approach.
- Non-trivial technique, fix, workaround, debugging path emerged.
- A loaded skill was wrong, missing, or outdated.
Preference order:
1. Update currently-loaded skill.
2. Update existing umbrella skill.
3. Add support file.
4. Create new class-level umbrella skill.
```

### 7.3 Combined Memory + Skill Review

载入条件：

- Memory review 和 Skill review 同时触发。

核心内容：

- 同时判断“用户是谁/偏好是什么”和值得写入 memory 的内容。
- 同时判断“这个类别任务以后怎么做”并更新 skill。
- 没有有效信号时才输出 `Nothing to save.`。

## 8. Curator 提示词

载入条件：

- 技能 curator 后台维护流程运行。

核心内容摘要：

```text
You are running as Hermes' background skill CURATOR.
This is an UMBRELLA-BUILDING consolidation pass.
The goal is a library of CLASS-LEVEL instructions and experiential knowledge.
Do NOT touch bundled or hub-installed skills.
Do NOT delete any skill. Archiving is the maximum destructive action.
Do NOT touch pinned skills.
Judge overlap on CONTENT, not use_count.
```

中文说明：

- Curator 不是普通审计，而是技能库整理/合并流程。
- 目标是减少一次性窄 skill，形成可复用 umbrella skill。

## 9. 压缩 / 摘要提示词

### 9.1 主会话上下文压缩

载入条件：

- 会话过长。
- 手动或自动触发 context compression。

核心 preamble：

```text
You are a summarization agent creating a context checkpoint.
Your output will be injected as reference material for a DIFFERENT assistant.
Do NOT respond to any questions or requests in the conversation.
Only output the structured summary.
Write the summary in the same language the user was using.
NEVER include API keys, tokens, passwords, secrets, credentials, or connection strings.
```

结构模板：

```text
## Active Task
## Goal
## Constraints & Preferences
## Completed Actions
## Active State
## In Progress
## Blocked
## Key Decisions
## Resolved Questions
## Pending User Asks
## Relevant Files
## Remaining Work
## Critical Context
```

迭代压缩额外提示：

```text
You are updating a context compaction summary.
PRESERVE all existing information that is still relevant.
ADD new completed actions.
Update Active Task to reflect the user's most recent unfulfilled request.
```

Focus 压缩额外提示：

```text
FOCUS TOPIC: "<topic>"
Prioritise preserving all information related to the focus topic.
For unrelated content, summarise more aggressively.
```

### 9.2 Trajectory 压缩

载入条件：

- 轨迹压缩器压缩 agent turns。

核心内容：

```text
Summarize the following agent conversation turns concisely.
This summary will replace these turns in the conversation history.
Write the summary from a neutral perspective describing what the assistant did and learned.
Write only the summary, starting with "[CONTEXT SUMMARY]:" prefix.
```

### 9.3 最大迭代数总结

载入条件：

- 工具调用循环达到最大迭代数。

核心内容：

```text
You've reached the maximum number of tool-calling iterations allowed.
Please provide a final response summarizing what you've found and accomplished so far,
without calling any more tools.
```

## 10. 标题 / 历史搜索 / Web / Vision 辅助提示词

### 10.1 会话标题生成

载入条件：

- 首轮 user/assistant exchange 后自动生成 session title。

核心内容：

```text
Generate a short, descriptive title (3-7 words) for a conversation.
Return ONLY the title text, nothing else.
No quotes, no punctuation at the end, no prefixes.
```

### 10.2 Session Search 摘要

载入条件：

- 用户搜索历史会话。
- 检索到候选会话后需要辅助 LLM 总结。

System prompt：

```text
You are reviewing a past conversation transcript to help recall what happened.
Summarize the conversation with a focus on the search topic.
Include what the user wanted, actions taken, outcomes, decisions,
commands, files, URLs, technical details, and unresolved items.
```

User prompt 模板：

```text
Search topic: <query>
Session source: <source>
Session date: <started>

CONVERSATION TRANSCRIPT:
<conversation_text>

Summarize this conversation with focus on: <query>
```

### 10.3 Web 内容摘要

载入条件：

- Web extract 结果过长，需要辅助 LLM 摘要。

完整文档 system prompt：

```text
You are an expert content analyst.
Process web content and create a comprehensive yet concise summary
that preserves all important information while dramatically reducing bulk.
```

分块 system prompt：

```text
You are an expert content analyst processing a SECTION of a larger document.
Extract and summarize the key information from THIS SECTION ONLY.
Do NOT write introductions or conclusions.
Preserve facts, figures, data points, quotes, code snippets, and specific details.
```

综合摘要 system prompt：

```text
You synthesize multiple summaries into one cohesive, comprehensive summary.
Be thorough but concise.
```

### 10.4 Browser Snapshot 摘要

载入条件：

- 浏览器页面 snapshot 过长，需要抽取和用户任务相关的信息。

核心内容：

```text
You are a content extractor for a browser automation agent.
Given the page snapshot, extract and summarize the most relevant information.
Focus on interactive elements, relevant text content, and navigation structure.
Keep ref IDs for interactive elements.
```

### 10.5 Vision Analyze

载入条件：

- 调用视觉分析工具分析图片。

核心包装：

```text
Fully describe and explain everything about this image,
then answer the following question:

<question>
```

### 10.6 Browser Screenshot Vision

载入条件：

- 浏览器截图需要视觉模型分析。

核心内容：

```text
Analyze this browser screenshot and answer: <question>
```

### 10.7 Sticker Vision

载入条件：

- Telegram 等平台收到 sticker，且缓存中没有描述，需要视觉模型生成描述。

核心内容：

```text
Describe this sticker in 1-2 sentences.
Focus on what it depicts -- character, action, emotion.
Be concise and objective.
```

### 10.8 TUI / CLI 附图预分析

载入条件：

- 用户在 TUI/CLI 入口附加本地图片。
- 入口先用视觉模型生成图片描述，再把描述拼回用户消息。

核心内容：

```text
Describe everything visible in this image in thorough detail.
Include any text, code, data, objects, people, layout, colors,
and any other notable visual information.
```

注入到用户消息的包装：

```text
[The user attached an image:
<vision description>]
[You can examine it with vision_analyze using image_url: <path>]
```

### 10.9 Feishu 评论 Agent Prompt

载入条件：

- 飞书文档评论事件中 @mention 了 agent。
- 评论可分为局部引用评论和整篇文档评论。

局部评论 prompt 结构：

```text
The user added a reply in "<doc_title>".
Current user comment text: "<target_reply_text>"
Original comment text: "<root_comment_text>"
Quoted content: "<quote_text>"
This comment mentioned you (@mention is for routing, not task content).
Document link: <doc_url>
Current commented document:
- file_type=<file_type>
- file_token=<file_token>
- comment_id=<comment_id>
Current comment card timeline:
...
```

整篇文档评论 prompt 结构：

```text
The user added a comment in "<doc_title>".
Current user comment text: "<comment_text>"
This is a whole-document comment.
This comment mentioned you (@mention is for routing, not task content).
Document link: <doc_url>
Current commented document:
- file_type=<file_type>
- file_token=<file_token>
Whole-document comment timeline:
...
```

中文说明：

- 该 prompt 把文档标题、链接、评论文本、引用内容、评论时间线和当前文档标识传给 agent。
- 明确说明 @mention 只是路由，不是任务内容本身。

### 10.10 Webhook Prompt Template

载入条件：

- Webhook 平台收到某条动态路由事件。
- route 配置中存在 `prompt` 模板。

注入方式：

```text
<route_config.prompt 渲染 payload 后的文本>
```

如果 route 配置了 skills，则会把第一个匹配 skill 的完整调用包装作为 prompt：

```text
[IMPORTANT: The user has invoked the "<skill_name>" skill,
indicating they want you to follow its instructions.
The full skill content is loaded below.]

<skill content>

The user has provided the following instruction alongside the skill invocation:
<rendered webhook prompt>
```

## 11. 多模型聚合 / 审批 / 安全提示词

### 11.1 Mixture of Agents 聚合提示

载入条件：

- 调用 mixture_of_agents 工具。
- 多个 reference model 已返回答案。

Aggregator system prompt：

```text
You have been provided with a set of responses from various open-source models
to the latest user query.
Your task is to synthesize these responses into a single, high-quality response.
Critically evaluate the information, recognizing that some may be biased or incorrect.
Do not simply replicate the given answers.
```

### 11.2 Smart Approval 命令审批提示

载入条件：

- 终端命令被模式匹配标记为潜在危险。
- 启用智能审批。

核心内容：

```text
You are a security reviewer for an AI coding agent.
A terminal command was flagged by pattern matching as potentially dangerous.

Assess the ACTUAL risk of this command.
APPROVE if clearly safe.
DENY if genuinely dangerous.
ESCALATE if uncertain.

Respond with exactly one word: APPROVE, DENY, or ESCALATE.
```

## 12. API / Provider 协议提示词形态

### 12.1 Chat Completions

载入条件：

- 使用 OpenAI-compatible chat completions。

提示词形态：

- system message 承载基础 system prompt。
- user/assistant/tool messages 承载会话历史和工具结果。
- GPT-5 / Codex 等模型可能把 system role 改为 developer role。

### 12.2 OpenAI Responses / Codex

载入条件：

- 使用 Responses API 或 Codex Responses 兼容路径。

提示词形态：

- system prompt 被提取为 `instructions`。
- 其他消息转成 Responses API input items。
- reasoning 配置可能进入请求字段。

### 12.3 Anthropic Messages

载入条件：

- 使用 Anthropic Messages 协议。

提示词形态：

- system message 被提取为独立 `system` 参数。
- multimodal 内容、tool calls、thinking blocks 会转换为 Anthropic 格式。

### 12.4 Bedrock Converse

载入条件：

- 使用 AWS Bedrock Converse。

提示词形态：

- system prompt 转成 Bedrock system blocks。
- tool calls 转成 `toolUse`。
- tool results 转成 user role 中的 `toolResult`。

## 13. 动态外部提示词来源

这些提示词不固定在仓库中，但运行时会进入 LLM 请求。

| 来源 | 载入条件 | 注入位置 |
| --- | --- | --- |
| 用户输入 | 每轮用户消息 | user message |
| `agent.system_prompt` | 用户配置了 personality/system prompt | ephemeral system |
| `HERMES_EPHEMERAL_SYSTEM_PROMPT` | 环境变量存在 | ephemeral system |
| prefill messages 文件 | 配置了 prefill 文件 | system 后、历史前 |
| Memory/User 文件 | memory/user profile 启用 | system prompt |
| 外部 memory provider 召回 | provider 启用且 prefetch 命中 | 当前 user message |
| 项目上下文文件 | 当前工作目录存在相关文件 | system prompt |
| channel prompt | 平台配置命中 channel/thread | ephemeral system |
| cron job prompt | cron job 执行 | user prompt |
| cron script output | job 配置 script 且有输出 | user prompt 前缀 |
| webhook prompt template | webhook route 配置 prompt | user prompt |
| MCP sampling messages | MCP server 发起 sampling | 独立 LLM 请求 |
| 工具结果 | 模型调用工具后 | tool message |
| 图片/附件/文档提取文本 | 平台收到附件并可解析 | user message 前缀 |

## 14. Prompt 安全相关提示/规则

载入或生效位置：

- 上下文文件读取前。
- Memory 写入前。
- Cron prompt 创建/更新时。
- Skill 读取/安装/管理时。
- MCP tool description 加载时。
- Web/browser 内容发给辅助 LLM 前。

规则摘要：

- 扫描 `ignore previous instructions`、`disregard rules`、`system prompt override` 等注入模式。
- 扫描不可见 unicode。
- 扫描读取或外传 secret 的模式。
- 对网页/browser 内容做敏感信息 redaction。
- 长内容截断。

中文说明：

- 这些不是直接给模型看的主要提示词，但会决定哪些外部文本能进入提示词链路。

## 15. 总结：Hermes 提示词资产全景

```text
固定主提示词
  - 基础身份
  - Hermes 帮助提示
  - Memory / Session Search / Skills / Kanban 指导
  - 工具强制使用
  - OpenAI / Google 模型族指导
  - 平台提示
  - 环境提示

动态主提示词
  - 用户配置 system prompt
  - Gateway session context
  - Channel prompt
  - Project context files
  - Memory/User 快照
  - 外部 memory 召回
  - Prefill messages
  - 用户输入、附件、引用、群聊 sender label

技能提示词资产
  - Skills 索引
  - 186 个 SKILL.md / DESCRIPTION.md
  - Slash/preload/cron skill 包装提示

辅助 LLM 提示词
  - 上下文压缩
  - Trajectory 压缩
  - 最大迭代数总结
  - 标题生成
  - Session search 摘要
  - Web extract 摘要
  - Browser snapshot 摘要
  - Vision / Sticker 描述
  - Mixture of Agents 聚合
  - Smart Approval
  - Memory / Skill review
  - Curator

协议转换提示词形态
  - Chat Completions system/developer messages
  - Responses API instructions
  - Anthropic system 参数
  - Bedrock system blocks
```
