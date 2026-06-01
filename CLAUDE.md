# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作提供指引。

## 环境搭建

```bash
uv venv venv --python 3.11 && source venv/bin/activate
uv pip install -e ".[all,dev]"
npm install  # 可选，用于浏览器工具
```

## 常用命令

```bash
# 运行全部测试（隔离环境，每文件独立进程，4 个 worker）
scripts/run_tests.sh

# 运行单个测试文件
scripts/run_tests.sh tests/test_retry_utils.py

# 运行单个测试函数
scripts/run_tests.sh tests/test_retry_utils.py -- -k test_backoff_is_exponential

# 代码检查
ruff check .

# 冒烟测试
hermes chat -q "test"
hermes doctor
```

## 架构概览

Hermes 是一个具备闭环自我学习能力的 AI Agent，能从经验中创建技能、在使用中持续改进，并支持多种部署目标。

**Agent 循环** — `run_agent.py`（AIAgent 类，约 4800 行）是核心，驱动对话循环：模型调用 → 工具分发 → 重试 → 降级 → 上下文压缩。`agent/conversation_loop.py` 和 `agent/chat_completion_helpers.py` 是从中拆分出的子系统。

**CLI / TUI** — `cli.py`（约 15k 行，HermesCLI）是基于 prompt_toolkit 的交互式终端界面，支持斜杠命令补全、多行编辑、流式工具输出和对话历史。

**工具** — `tools/` 包含 40+ 个工具（终端、文件操作、网页、视觉、浏览器、代码执行、委托、定时任务、技能）。工具通过 `tools/registry.py` 自注册。终端执行支持六种后端：local、Docker、SSH、Singularity、Modal、Daytona，通过配置选择。

**技能系统** — `skills/`（随每次安装打包）和 `optional-skills/`（官方但默认不启用）。技能是程序性记忆，Agent 可自主创建、调用和改进。兼容 agentskills.io 开放标准。

**Gateway** — `gateway/` 提供消息平台适配器（Telegram、Discord、Slack、WhatsApp、Signal、Matrix、飞书、钉钉、企业微信），含会话管理和定时任务调度。

**记忆与上下文** — `agent/context_compressor.py` 在接近上下文限制时自动摘要。`agent/memory_manager.py` 管理可插拔后端（honcho、mem0、supermemory 等）及 FTS5 全文检索。

**模型提供商** — `plugins/model-providers/` 通过 OpenRouter、Nous Portal、OpenAI、Anthropic、Mistral、Bedrock、Azure、NVIDIA NIM 等支持 200+ 个模型。提供商通过 `tools/lazy_deps.py` 懒加载，保持基础安装精简。

**配置** — `hermes_cli/config.py` 管理 YAML 配置（`cli-config.yaml`），支持环境变量覆盖，控制提供商选择、工具启用、记忆后端和终端后端。

## 目录说明

| 目录 | 作用 |
|---|---|
| `agent/` | Agent 核心运行时：上下文引擎、记忆管理、模型适配器、对话循环 |
| `tools/` | 40+ 个工具实现（终端、文件、网页、视觉、浏览器、委托、定时任务、技能），通过 `registry.py` 自注册 |
| `skills/` | 随安装打包的内置技能，按领域组织（devops、数据科学、创意等） |
| `optional-skills/` | 官方但默认不启用的技能（区块链、自主 Agent 等） |
| `gateway/` | 消息平台适配器：Telegram、Discord、Slack、WhatsApp、Signal、Matrix、飞书、钉钉、企业微信 |
| `hermes_cli/` | CLI 入口：认证、聊天、配置命令 |
| `tui_gateway/` | 交互式聊天会话的终端 UI 网关 |
| `ui-tui/` | TUI 渲染组件 |
| `plugins/` | 插件系统：模型提供商、浏览器、仪表盘、图像生成、看板 |
| `providers/` | 各服务/API 的提供商实现 |
| `optional-mcps/` | 可选 MCP 集成（Linear、n8n 等） |
| `apps/` | 桌面应用（Electron）和引导安装程序 |
| `web/` | Web 前端（Vite + TypeScript） |
| `cron/` | 定时任务定义和调度管理 |
| `docker/` | Docker 入口点和容器初始化脚本 |
| `scripts/` | 搭建、部署和维护脚本 |
| `tests/` | 测试套件 |
| `locales/` | 国际化翻译文件（YAML） |
| `docs/` | 文档源文件 |
| `nix/` | Nix 包管理器配置 |
| `packaging/` | 发行版配置（Homebrew 等） |
| `acp_adapter/` | Anthropic Cloud Platform 适配器，用于认证和编辑审批 |
| `plans/` | 功能设计和规划文档 |

## 技能 vs 工具的选择

行为特定于任务、可由用户自定义、或能从自我改进中受益时，添加**技能**。需要底层系统访问、必须在所有上下文中运行、或作为技能构建基础的原语时，添加**工具**。完整决策树见 `CONTRIBUTING.md`。
