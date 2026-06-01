# 系统提示词构建架构

本文档说明 Hermes Agent 如何在每次会话中组装系统提示词（system prompt）。

核心代码：
- `agent/system_prompt.py` — 组装入口，定义三层结构
- `agent/prompt_builder.py` — 所有常量字符串、技能索引、上下文文件加载

---

## 三层结构

系统提示词由三个部分按顺序拼接（`\n\n` 分隔）：

```
stable  →  context  →  volatile
```

### 1. stable（稳定层）

会话期间不变，用于保持上游 prefix cache 命中。按以下顺序拼接：

| 顺序 | 内容 | 来源 |
|------|------|------|
| 1 | Agent 身份 | `~/.hermes/SOUL.md`（优先）或 `DEFAULT_AGENT_IDENTITY` 常量 |
| 2 | Hermes 帮助指引 | `HERMES_AGENT_HELP_GUIDANCE` 常量 |
| 3 | 任务完成指引 | `TASK_COMPLETION_GUIDANCE`（所有模型，可通过 config 关闭） |
| 4 | 工具行为指引 | 按工具是否存在条件注入：`MEMORY_GUIDANCE`、`SESSION_SEARCH_GUIDANCE`、`SKILLS_GUIDANCE`、`KANBAN_GUIDANCE` |
| 5 | 电脑控制指引 | `COMPUTER_USE_GUIDANCE`（仅当 `computer_use` 工具存在） |
| 6 | Nous 订阅块 | `build_nous_subscription_prompt()`（仅当 Nous 托管工具启用） |
| 7 | 工具调用强制指引 | `TOOL_USE_ENFORCEMENT_GUIDANCE`（按模型名匹配，可配置） |
| 7a | Google 模型指引 | `GOOGLE_MODEL_OPERATIONAL_GUIDANCE`（gemini/gemma） |
| 7b | OpenAI 模型指引 | `OPENAI_MODEL_EXECUTION_GUIDANCE`（gpt/codex/grok） |
| 8 | 技能索引 | `build_skills_system_prompt()`（扫描 `~/.hermes/skills/`） |
| 9 | Alibaba 模型 ID 修正 | 硬编码字符串（仅 alibaba provider，绕过 API bug） |
| 10 | 环境提示 | `build_environment_hints()`（OS、home、cwd、WSL、远程后端） |
| 11 | Python 工具链探测 | `get_environment_probe_line()`（非默认环境才输出，可关闭） |
| 12 | 活跃 profile 提示 | 硬编码字符串，说明当前 profile 路径 |
| 13 | 平台提示 | `PLATFORM_HINTS[platform]`（cli/telegram/discord/feishu 等） |

### 2. context（上下文层）

依赖当前工作目录，会话间可能不同：

| 顺序 | 内容 | 来源 |
|------|------|------|
| 1 | 调用方传入的 system_message | `run_conversation()` 参数 |
| 2 | 项目上下文文件 | 按优先级取第一个：`.hermes.md`/`HERMES.md`（向上找到 git root）→ `AGENTS.md` → `CLAUDE.md` → `.cursorrules` |

每个上下文文件上限 20,000 字符，超出则头尾截断（70%头 + 20%尾）。

所有上下文文件在注入前经过 `_scan_context_content()` 扫描，检测 prompt injection 模式，命中则替换为 `[BLOCKED: ...]` 占位符。

### 3. volatile（易变层）

每次会话/压缩后重建，不参与 prefix cache：

| 顺序 | 内容 | 来源 |
|------|------|------|
| 1 | 内置记忆快照 | `_memory_store.format_for_system_prompt("memory")` |
| 2 | 用户画像（USER.md） | `_memory_store.format_for_system_prompt("user")` |
| 3 | 外部记忆提供商块 | `_memory_manager.build_system_prompt()`（honcho/mem0 等） |
| 4 | 时间戳行 | `Conversation started: Monday, June 01, 2026`（日期精度，不含分钟） |

时间戳只精确到日期，避免每分钟使 prefix cache 失效。

---

## 技能索引构建（build_skills_system_prompt）

两层缓存：

1. **进程内 LRU**：key = (skills_dir, external_dirs, tools, toolsets, platform, disabled)
2. **磁盘快照**：`~/.hermes/.skills_prompt_snapshot.json`，通过 mtime/size manifest 验证有效性

冷路径：全量扫描 `~/.hermes/skills/` 下所有 `SKILL.md`，解析 frontmatter，过滤：
- 平台不兼容的技能（`platforms` 字段）
- 被禁用的技能
- 条件不满足的技能（`requires_tools`、`fallback_for_toolsets` 等）

外部技能目录（`skills.external_dirs` in config.yaml）也会扫描，本地技能优先（同名时跳过外部）。

---

## 工具调用强制指引的注入逻辑

由 `agent._tool_use_enforcement` 控制（来自 `config.yaml agent.tool_use_enforcement`）：

| 配置值 | 行为 |
|--------|------|
| `"auto"`（默认） | 匹配 `TOOL_USE_ENFORCEMENT_MODELS` 中的模型名子串 |
| `true` | 所有模型都注入 |
| `false` | 所有模型都不注入 |
| `[list]` | 自定义模型名子串列表 |

默认触发模型：`gpt`, `codex`, `gemini`, `gemma`, `grok`, `glm`, `qwen`, `deepseek`

---

## 环境提示构建（build_environment_hints）

- **本地后端**：输出 Host OS、用户 home、cwd；Windows 额外说明 bash shell
- **远程后端**（docker/modal/ssh/singularity/daytona）：抑制宿主机信息，改为在后端内执行探测命令（`uname -a && whoami && pwd`），输出后端的 OS/user/home/cwd
- **WSL**：追加 `/mnt/c/` 路径映射说明

探测结果按 `(env_type, cwd_hint)` 缓存在进程内，不写磁盘。

---

## 上下文文件优先级

```
.hermes.md / HERMES.md   ← 向上遍历到 git root
        ↓ 未找到
AGENTS.md / agents.md    ← 仅 cwd
        ↓ 未找到
CLAUDE.md / claude.md    ← 仅 cwd
        ↓ 未找到
.cursorrules + .cursor/rules/*.mdc  ← 仅 cwd
```

只取第一个命中的，不叠加。SOUL.md 独立于此优先级链，始终从 `~/.hermes/SOUL.md` 加载。

---

## 缓存策略

系统提示词在 `agent._cached_system_prompt` 上缓存，整个会话期间只构建一次。触发重建的唯一场景：**上下文压缩**（`invalidate_system_prompt()`）。

重建时同时从磁盘重新加载记忆，确保压缩后的提示词包含本次会话写入的新记忆。

---

## 本次会话的实际提示词

该会话（`session_20260601_142436_cca75a`）使用 `platform=cli`，因此 stable 层末尾注入了：

```
You are a CLI AI Agent. Try not to use markdown but simple text renderable inside a terminal. ...
```

volatile 层末尾为：

```
Conversation started: Monday, June 01, 2026
Model: claude-sonnet-4-6
Provider: custom
```

完整提示词见：`agent/docs/session_20260601_142436_cca75a_system_prompt.txt`
