# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此目录中工作提供指引。

## 概览

`agent/` 目录是 Hermes 的核心运行时，包含对话循环、所有模型提供商适配器、工具分发、记忆、上下文压缩、技能基础设施以及可插拔提供商注册表。

## 关键文件

### 对话与循环

| 文件 | 作用 |
|---|---|
| `conversation_loop.py` | Agent 主对话循环（约 3900 行）：模型调用 → 工具分发 → 重试 → 降级 → 压缩 |
| `chat_completion_helpers.py` | chat-completions 代码路径的请求构建和降级激活 |
| `agent_init.py` | `AIAgent.__init__` 实现：提供商自动检测和凭证解析 |
| `agent_runtime_helpers.py` | 运行时辅助：轨迹转换、消息修复、凭证恢复、API 调试 |
| `system_prompt.py` | 系统提示组装，合并稳定层、上下文层和易变层 |
| `prompt_builder.py` | 将身份、平台提示、技能索引和上下文文件组装为系统提示 |

### 工具执行

| 文件 | 作用 |
|---|---|
| `tool_executor.py` | 工具调用执行，支持顺序和并发分发 |
| `tool_dispatch_helpers.py` | 并行度控制、多模态封装、变更追踪 |
| `tool_guardrails.py` | 循环护栏原语，追踪观察结果并返回决策 |
| `tool_result_classification.py` | 工具结果负载分类 |
| `iteration_budget.py` | 线程安全的迭代计数器，支持程序化工具调用的退款 |

### 上下文与记忆

| 文件 | 作用 |
|---|---|
| `context_compressor.py` | 使用辅助模型对中间轮次摘要，自动压缩上下文窗口 |
| `context_engine.py` | 可插拔上下文引擎的抽象基类 |
| `context_references.py` | 解析用户输入中的 `@` 引用（文件、目录、git、URL） |
| `conversation_compression.py` | 压缩启动检查、警告重放、图片尺寸恢复 |
| `memory_manager.py` | 编排记忆提供商（最多一个外部提供商） |
| `memory_provider.py` | 可插拔记忆提供商的抽象基类 |
| `background_review.py` | 异步 fork，在后台评估轮次并更新记忆/技能 |

### 模型提供商与适配器

| 文件 | 作用 |
|---|---|
| `anthropic_adapter.py` | Anthropic Messages API 适配器（OpenAI ↔ Anthropic 格式转换，OAuth） |
| `bedrock_adapter.py` | AWS Bedrock Converse API 适配器，支持动态模型发现和护栏 |
| `gemini_native_adapter.py` | 封装 Google AI Studio 原生 Gemini API 的 OpenAI 兼容外观 |
| `gemini_cloudcode_adapter.py` | 封装 Google Cloud Code Assist 的 OpenAI 兼容外观，含 OAuth PKCE |
| `copilot_acp_client.py` | 转发至 `copilot --acp` 的 OpenAI 兼容垫片 |
| `codex_runtime.py` | OpenAI Responses API（Codex）运行时 |
| `auxiliary_client.py` | 辅助任务（压缩、搜索、视觉）的共享路由器，含多提供商降级链 |
| `plugin_llm.py` | 为受信任插件暴露宿主 LLM 访问的插件 LLM 外观 |

### 凭证与限速

| 文件 | 作用 |
|---|---|
| `credential_pool.py` | 持久化多凭证池，支持同提供商故障转移、刷新和轮换 |
| `credential_sources.py` | 所有凭证来源（env、OAuth、device_code 等）的统一移除契约 |
| `credential_persistence.py` | 定义哪些凭证条目持久化到 `auth.json` |
| `rate_limit_tracker.py` | 追踪推理 API 响应中的 `x-ratelimit-*` 头 |
| `nous_rate_guard.py` | Nous Portal 的跨会话限速守卫，防止重试放大 |
| `account_usage.py` | 追踪用量窗口和快照，用于限速监控 |
| `error_classifier.py` | 分类 API 错误以实现智能故障转移（重试、轮换、降级、压缩） |
| `retry_utils.py` | 带抖动的指数退避，防止惊群效应 |

### 技能

| 文件 | 作用 |
|---|---|
| `skill_commands.py` | 技能的共享斜杠命令辅助（CLI 和 gateway） |
| `skill_preprocessing.py` | SKILL.md 预处理：模板替换和内联 shell 展开 |
| `skill_utils.py` | 供 prompt_builder 和 skills_tool 使用的轻量技能元数据工具 |
| `skill_bundles.py` | 技能包，作为别名在单个斜杠命令下加载多个技能 |
| `curator.py` | 后台技能维护：自动转换生命周期状态，整合 Agent 创建的技能 |
| `curator_backup.py` | Curator 快照和回滚，含 tar.gz 备份和清单追踪 |

### 可插拔提供商注册表

每种媒体/服务类型遵循相同模式：`*_provider.py` 抽象基类 + `*_registry.py` 中央注册表。

| 注册表 | 提供商类型 |
|---|---|
| `browser_registry.py` | 云浏览器提供商（Browserbase、Browser Use、Firecrawl） |
| `image_gen_registry.py` | 图像生成提供商 |
| `transcription_registry.py` | 语音转文字提供商 |
| `tts_registry.py` | 文字转语音提供商 |
| `video_gen_registry.py` | 视频生成提供商 |
| `web_search_registry.py` | 网页搜索提供商 |

### 提示缓存与 Schema 辅助

| 文件 | 作用 |
|---|---|
| `prompt_caching.py` | Anthropic 提示缓存策略（system_and_3 布局，输入 token 成本降低约 75%） |
| `gemini_schema.py` | 将 OpenAI 工具 schema 转换为 Gemini 的 schema 子集 |
| `moonshot_schema.py` | 将 OpenAI 工具 schema 转换为 Moonshot 更严格的 schema 子集 |

### 工具类

| 文件 | 作用 |
|---|---|
| `message_sanitization.py` | 修复消息和工具负载中的问题字符 |
| `redact.py` | 基于正则的日志和工具输出脱敏，屏蔽 API 密钥和 token |
| `file_safety.py` | 工具和 ACP 垫片共用的写入禁止路径检测 |
| `display.py` | CLI 展示：进度条、kawaii 表情、工具预览格式化 |
| `i18n.py` | 静态用户可见消息的轻量国际化，回退到英文 |
| `markdown_tables.py` | 模型输出的 Markdown 表格 CJK/宽字符感知重对齐 |
| `think_scrubber.py` | 流式助手文本中推理/思考块的有状态清除器 |
| `stream_diag.py` | 每次尝试的流诊断：计数器、异常链、重试日志 |
| `async_utils.py` | 异步/同步桥接辅助，循环失败时正确清理 |
| `jiter_preload.py` | 尽力提前导入 OpenAI SDK 原生流解析器 |
| `process_bootstrap.py` | 进程级引导：懒加载 OpenAI、崩溃安全 stdio、HTTP 代理解析 |
| `model_metadata.py` | Models.dev 注册表集成，离线优先缓存 |
| `usage_pricing.py` | 按模型计算 token 成本 |
| `insights.py` | 会话洞察：token 消耗、成本、工具使用趋势 |
| `trajectory.py` | 会话录制工具 |
| `title_generator.py` | 从首轮对话异步生成简短会话标题 |
| `onboarding.py` | 每次安装仅显示一次的上下文首次引导提示 |
| `portal_tags.py` | Nous Portal 请求标签，用于产品归因 |
| `shell_hooks.py` | 从配置读取 shell 脚本钩子并注册回调 |
| `subdirectory_hints.py` | Agent 导航时的渐进式子目录提示发现 |
| `image_routing.py` | 路由用户附加图片（按模型能力选择原生或文本模式） |
| `lmstudio_reasoning.py` | 将 LM Studio 推理强度配置映射到 OpenAI 兼容词汇 |
| `manual_compression_feedback.py` | 手动压缩命令的用户可见摘要 |

## 子目录

### `lsp/`
写入后语义诊断的语言服务器协议层。关键文件：`manager.py`（服务编排）、`client.py`（异步 LSP 客户端）、`servers.py`（按语言的服务器注册表）、`reporter.py`（将诊断格式化为工具输出）。

### `secret_sources/`
外部密钥提供商集成。目前：`bitwarden.py`（通过 `bws` CLI 集成 Bitwarden Secrets Manager）。

### `transports/`
提供商特定的格式转换。每个 transport 实现 `base.py` 抽象基类：`convert_messages → convert_tools → build_kwargs → normalize_response`。已有 transport：`anthropic.py`、`bedrock.py`、`chat_completions.py`（覆盖约 16 个 OpenAI 兼容提供商）、`codex.py`、`codex_app_server.py`。
