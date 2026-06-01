# conversation_loop.py 详解

`conversation_loop.py` 是从 `run_agent.AIAgent` 中拆分出来的最大单块逻辑，约 3900 行，包含 `run_conversation` 函数。它驱动一次用户轮次从头到尾的完整执行：模型调用、工具分发、重试、降级、压缩、后处理钩子。

`AIAgent.run_conversation` 现在只是一个薄转发器，真正的逻辑在这里。

---

## 整体执行阶段

```
初始化
  └─ 预检压缩
       └─ 主循环（每次迭代）
            ├─ 消息准备
            ├─ API 调用（含重试/降级循环）
            ├─ 响应处理
            └─ 工具执行
后处理
  ├─ 轨迹保存
  ├─ 会话持久化
  ├─ 插件钩子
  └─ 后台记忆/技能审查
```

### 1. 初始化阶段

- 清理 Agent 状态，重置各类重试计数器
- 从 SQLite 恢复 todo 状态、记忆 nudge 计数器
- 构建或从会话 DB 恢复系统提示（用于前缀缓存）
- 一次性调用 `memory_manager.prefetch_all()`，结果缓存供本轮所有迭代复用
- 触发 `pre_llm_call` 插件钩子注入上下文

### 2. 预检压缩

进入主循环前，如果消息历史已超过阈值，先执行一次压缩，避免第一次 API 调用就因上下文溢出失败。

### 3. 主循环

条件：`api_call_count < max_iterations && iteration_budget.remaining > 0`

每次迭代按顺序执行：

1. **中断检查** — 检查 `agent._interrupt_requested`，若已设置则提前退出
2. **消耗迭代预算** — `iteration_budget.consume()`
3. **/steer 排水** — 注入待处理的用户实时指导消息
4. **消息准备**
   - 修复工具调用参数（损坏的 JSON）
   - 修复角色交替违规
   - 注入记忆上下文和插件上下文
   - 应用 Anthropic 提示缓存控制
   - 清理孤立工具结果
5. **API 调用**（见下文重试/降级章节）
6. **响应处理**
   - 验证工具调用名称和 JSON 参数
   - 执行工具调用（见下文工具分发章节）
   - 空响应恢复（重试、nudge、prefill）
   - 提取最终响应文本

### 4. 后处理阶段

主循环退出后：

- 保存轨迹（`trajectory.py`）
- 清理任务资源
- 持久化会话到 SQLite
- 生成文件变更验证页脚
- 触发插件钩子：`transform_llm_output` → `post_llm_call` → `on_session_end`
- 异步 fork 后台记忆/技能审查

---

## 重试与降级逻辑

### 触发重试的条件

| 条件 | 最大重试次数 |
|---|---|
| 响应为 None / choices 为空 | 立即尝试降级 |
| `finish_reason == "length"`（截断） | 压缩后重试 |
| `finish_reason == "incomplete"`（Codex） | 3 次 |
| 工具名称不在合法列表中 | 3 次 |
| 工具参数不是合法 JSON | 3 次 |
| 无内容且无推理（空响应） | 3 次，之后尝试降级 |
| 不完整的 REASONING_SCRATCHPAD | 2 次 |

### FailoverReason 及其处理方式

`error_classifier.py` 将 API 错误分类为 `FailoverReason`，`conversation_loop.py` 根据分类决定恢复策略：

| FailoverReason | 触发条件 | 处理方式 |
|---|---|---|
| `rate_limit` | HTTP 429 | 立即切换备用提供商，无备用则指数退避 |
| `billing` | HTTP 402 / 配额耗尽 | 刷新凭证或切换提供商 |
| `context_overflow` | 上下文超出窗口 | 压缩历史（最多 3 次） |
| `payload_too_large` | HTTP 413 | 压缩历史 |
| `image_too_large` | 图像超过 5MB | 缩小图像后重试 |
| `thinking_signature` | 思考块签名无效 | 剥离所有推理块后重试 |
| `invalid_encrypted_content` | Codex 加密推理失败 | 禁用回放后重试 |
| `multimodal_tool_content_unsupported` | 工具消息拒绝列表内容 | 从工具消息中剥离图像后重试 |
| `oauth_long_context_beta_forbidden` | Anthropic OAuth 不支持 1M | 禁用 1M beta 后重试 |
| `llama_cpp_grammar_pattern` | llama.cpp 拒绝正则表达式 | 从工具 schema 中剥离 pattern/format |
| `long_context_tier` | Anthropic 需要额外使用 | 将上下文压缩到 200K |
| `content_policy_blocked` | 安全过滤器拦截 | 尝试降级，无备用则返回错误 |

### 重试退避策略

- 普通错误：`jittered_backoff(retry_count, base_delay=2.0, max_delay=60.0)` 指数退避
- 速率限制：优先读取响应头中的 `Retry-After`
- 无效响应：不等待，立即尝试降级
- 退避期间每 0.2 秒检查一次中断请求

---

## 上下文压缩

### 触发时机

| 时机 | 触发条件 |
|---|---|
| 预检（进入主循环前） | 消息历史已超过阈值 |
| HTTP 413 响应 | payload 过大 |
| HTTP 400 + 大会话启发式 | 上下文溢出 |
| 工具执行后 | 基于实际 token 计数超阈值 |
| Anthropic 长上下文层 | 需要额外使用权限 |

### 压缩流程

```
1. 估计请求 token（含工具 schema）
2. context_compressor.should_compress(tokens) 检查阈值
3. 调用 agent._compress_context()
   └─ 使用辅助模型（auxiliary_client）对中间轮次生成摘要
4. 最多执行 3 次压缩通过
5. 清除 conversation_history 强制新会话写入
6. 重置重试计数器，重新进入 API 调用
```

关键变量：`compression_attempts`（最多 3）、`context_compressor.last_prompt_tokens`。

---

## 工具分发

### 工具调用验证

收到模型响应后，先验证：
1. 工具名称是否在 `valid_tool_names` 中（不在则重试，最多 3 次）
2. 工具参数是否是合法 JSON（不合法则重试，最多 3 次）

### 执行

通过 `agent._execute_tool_calls(assistant_message, messages, effective_task_id, api_call_count)` 分发，并发度由 `tool_dispatch_helpers.py` 控制。

### 工具执行后的处理

1. **工具守卫检查** — 若 `tool_guardrails` 决定停止，直接返回最终响应
2. **迭代预算退款** — 若本次迭代仅调用了 `execute_code`，退还预算
3. **上下文压缩检查** — 基于实际 token 计数决定是否压缩
4. **增量会话保存** — 每次工具执行后保存进度

---

## 后台审查（Post-turn Hooks）

主循环结束、响应交付后，异步 fork 后台任务：

- **记忆审查**：每隔 `_memory_nudge_interval` 轮次触发，调用 `agent._spawn_background_review(review_memory=True)`
- **技能审查**：每隔 `_skill_nudge_interval` 工具迭代触发，调用 `agent._spawn_background_review(review_skills=True)`
- **外部记忆同步**：`agent._sync_external_memory_for_turn()` 同步已完成轮次并排队下一次预取

后台审查在独立线程中运行，不与用户任务竞争资源。

---

## Agent 对象上的关键状态变量

### 每轮重置的重试计数器

| 变量 | 含义 | 上限 |
|---|---|---|
| `_empty_content_retries` | 空响应重试 | 3 |
| `_thinking_prefill_retries` | 思考 prefill 重试 | 2 |
| `_invalid_tool_retries` | 无效工具名称重试 | 3 |
| `_invalid_json_retries` | 无效 JSON 参数重试 | 3 |
| `_incomplete_scratchpad_retries` | 不完整推理块重试 | 2 |
| `_codex_incomplete_retries` | Codex 不完整响应重试 | 3 |
| `_post_tool_empty_retried` | 工具后空响应 nudge | 1 次 |

### 压缩状态

| 变量 | 含义 |
|---|---|
| `compression_attempts` | 当前压缩次数（上限 3） |
| `context_compressor.last_prompt_tokens` | 上次 API 调用的实际 prompt token 数 |

### 工具执行状态

| 变量 | 含义 |
|---|---|
| `_last_content_with_tools` | 上一轮次的内容（用于空响应降级） |
| `_turn_failed_file_mutations` | 本轮失败的文件写入（用于页脚提示） |
| `_tool_guardrail_halt_decision` | 工具守卫停止决定 |

### 会话状态

| 变量 | 含义 |
|---|---|
| `_user_turn_count` | 用户轮次计数（用于记忆 nudge） |
| `_turns_since_memory` | 自上次记忆审查以来的轮次数 |
| `_iters_since_skill` | 自上次技能审查以来的工具迭代数 |
| `_interrupt_requested` | 用户中断标志 |
| `_current_task_id` | 本轮任务 ID（用于 VM 隔离） |
| `_response_was_previewed` | 流式传输中断恢复标志 |

---

## 关键辅助函数

| 函数 | 作用 |
|---|---|
| `_restore_or_build_system_prompt(agent, ...)` | 从会话 DB 恢复或新建系统提示，处理前缀缓存失效 |
| `classify_api_error(...)` | 将 API 错误分类为 `FailoverReason`（来自 `error_classifier.py`） |
| `agent._compress_context(...)` | 使用辅助模型压缩中间轮次 |
| `agent._execute_tool_calls(...)` | 分发并执行工具调用，追踪结果 |
| `agent._repair_message_sequence(messages)` | 修复角色交替违规 |
| `agent._sanitize_tool_call_arguments(...)` | 修复损坏的工具参数 |
| `agent._get_transport().normalize_response(...)` | 将提供商响应转换为统一的 `AssistantMessage` 格式 |
| `agent._spawn_background_review(...)` | 在后台线程中异步运行记忆/技能审查 |

---

## 性能优化点

- **系统提示缓存**：每会话构建一次，所有轮次复用（Anthropic 前缀缓存，输入 token 成本降低约 75%）
- **记忆预取缓存**：每轮次预取一次，本轮所有迭代复用
- **迭代预算退款**：仅调用 `execute_code` 时退还预算，避免纯代码执行消耗 Agent 步数
- **中断响应性**：退避期间每 0.2 秒检查一次中断，保证用户体验
- **增量保存**：每次工具执行后保存进度，防止长任务因崩溃丢失数据
