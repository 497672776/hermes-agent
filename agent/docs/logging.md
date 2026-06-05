# 日志系统与 Dashboard 日志获取

## 概览

Hermes 的日志系统由 `hermes_logging.py` 统一管理，负责日志初始化、文件轮转、组件过滤和敏感信息脱敏。Dashboard 通过 HTTP 轮询 `/api/logs` 端点读取日志文件内容。

---

## 1. 日志初始化

**入口：** `hermes_logging.py:166` — `setup_logging()`

CLI、Gateway、GUI 启动时各自调用：

```python
# CLI / Dashboard 模式
setup_logging(mode="gui")

# Gateway 模式
setup_logging(hermes_home=_hermes_home, mode="gateway")
```

**日志目录：** `~/.hermes/logs/`（通过 `get_hermes_home()` 获取，支持 `HERMES_HOME` 环境变量覆盖）

---

## 2. 日志文件

每次启动会创建最多 4 个日志文件：

| 文件 | 级别 | 内容 |
|------|------|------|
| `agent.log` | INFO+ | 所有 agent/工具/会话活动（主日志） |
| `errors.log` | WARNING+ | 仅错误和警告 |
| `gateway.log` | INFO+ | 仅 gateway 相关事件（`mode="gateway"` 时创建） |
| `gui.log` | INFO+ | Dashboard/WebSocket/TUI 事件（`mode="gui"` 时创建） |

---

## 3. 日志轮转

**实现：** `hermes_logging.py:322` — `_ManagedRotatingFileHandler`（继承自 `RotatingFileHandler`）

- 默认每文件 5MB，保留 3 个备份
- 可通过 `~/.hermes/config.yaml` 配置：
  ```yaml
  logging:
    level: INFO
    max_size_mb: 5
    backup_count: 3
  ```
- 检测 inode 变化（外部 logrotate 或手动 mv），自动重新打开文件
- 在 NixOS 托管模式下确保文件权限为 0660（组可写）

---

## 4. 会话上下文

**实现：** `hermes_logging.py:45` — 线程本地存储

每条日志行自动附加 `[session_id]` 标签，通过自定义 `LogRecord` 工厂注入（`hermes_logging.py:94`）：

```python
set_session_context(session_id)   # 会话开始时设置
clear_session_context()           # 会话结束时清除
```

---

## 5. 组件过滤

**实现：** `hermes_logging.py:130` — `_ComponentFilter` 类

`COMPONENT_PREFIXES`（`hermes_logging.py:147`）定义各组件对应的 logger 前缀：

| 组件名 | Logger 前缀 |
|--------|------------|
| `gateway` | `gateway`, `hermes_plugins` |
| `agent` | `agent`, `run_agent`, `model_tools`, `batch_runner` |
| `tools` | `tools` |
| `cli` | `hermes_cli`, `cli` |
| `cron` | `cron` |
| `gui` | `hermes_cli.web_server`, `hermes_cli.pty_bridge`, `tui_gateway`, `uvicorn` |

`gateway.log` 和 `gui.log` 各自挂载对应的 `_ComponentFilter`，只写入匹配的日志记录。

---

## 6. 敏感信息脱敏

**实现：** `agent/redact.py:326` — `redact_sensitive_text()`

所有日志文件使用 `RedactingFormatter`（`agent/redact.py:488`），在写入磁盘前对每条记录执行脱敏，共 13 种模式：

1. 已知凭证前缀（`sk-`、`ghp_` 等）
2. 环境变量赋值（`OPENAI_API_KEY=***`）
3. JSON 字段（`"apiKey": "***"`）
4. Authorization 请求头（`Bearer ***`）
5. Telegram Bot Token
6. 私钥块（`[REDACTED PRIVATE KEY]`）
7. 数据库连接字符串（`postgres://user:***@host`）
8. JWT Token（`eyJ...`）
9. Form-urlencoded 请求体
10. E.164 电话号码（Signal、WhatsApp）

**性能优化：** 每个模式前置廉价子串预检，将典型日志行的扫描时间从 ~5.6µs 降至 ~1.8µs（-68%）。

可通过配置禁用：
```yaml
security:
  redact_secrets: false
```

---

## 7. CLI 日志访问

**命令：** `hermes logs [agent|errors|gateway|gui]`

**实现：** `hermes_cli/logs.py`

| 函数 | 行号 | 作用 |
|------|------|------|
| `tail_log()` | 140 | 读取并展示日志，支持过滤 |
| `_read_tail()` | 251 | 高效读取末尾 N 条匹配行 |
| `_read_last_n_lines()` | 280 | 文件 ≤1MB 全读，更大则从末尾分块读取 |
| `_follow_log()` | 336 | 实时轮询新内容（类似 `tail -f`） |
| `list_logs()` | 360 | 列出可用日志文件及大小 |

**过滤选项：**
- `--level`：DEBUG / INFO / WARNING / ERROR / CRITICAL
- `--session`：按会话 ID 子串过滤
- `--since`：相对时间（`1h`、`30m`、`2d`）
- `--component`：gateway / agent / tools / cli / cron
- `-n` / `--lines`：显示行数
- `-f` / `--follow`：实时跟踪

---

## 8. Dashboard 日志 API

**端点：** `GET /api/logs`（`hermes_cli/web_server.py:3728`）

**请求参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `file` | string | — | 日志文件名：`agent`、`errors`、`gateway`、`gui` |
| `lines` | int | 100 | 返回行数，最大 500（搜索时最大 2000） |
| `level` | string | ALL | 按级别过滤 |
| `component` | string | all | 按组件过滤 |
| `search` | string | — | 大小写不敏感的子串搜索 |

**响应结构（TypeScript）：**
```typescript
interface LogsResponse {
  file: string;
  lines: string[];
}
```

**实现逻辑：**
1. 调用 `hermes_cli.logs._read_tail()` 读取文件末尾
2. 通过 `COMPONENT_PREFIXES` 按组件过滤
3. 按 `search` 子串后过滤（大小写不敏感）
4. 文件不存在时返回空数组

---

## 9. Dashboard 前端日志页面

**文件：** `web/src/pages/LogsPage.tsx`

**功能：**
- 文件选择：agent、errors、gateway
- 级别过滤：ALL、DEBUG、INFO、WARNING、ERROR
- 组件过滤：all、gateway、agent、tools、cli、cron
- 行数选择：50、100、200、500
- 自动刷新（5 秒轮询）
- 颜色编码：ERROR/CRITICAL=红，WARNING=黄，DEBUG=灰，INFO=默认
- 刷新后自动滚动到底部

**API 调用：**
```typescript
api.getLogs({ file, lines: lineCount, level, component })
  .then((resp) => setLines(resp.lines))
```

日志**不通过 WebSocket 推送**，而是 HTTP 轮询（5 秒间隔）。

---

## 10. 数据流全景

```
应用代码（agent、gateway、CLI、plugins）
  ↓ logging.getLogger("component.subcomponent")

Root Logger
  ↓ 4 个 RotatingFileHandler

_ManagedRotatingFileHandler（自定义子类）
  ├─ agent.log   (INFO+，全量)
  ├─ errors.log  (WARNING+，全量)
  ├─ gateway.log (INFO+，仅 gateway.* / hermes_plugins.*)
  └─ gui.log     (INFO+，仅 web_server / pty_bridge / tui_gateway)

RedactingFormatter（每个 handler 独立挂载）
  ↓ redact_sensitive_text() — 13 种脱敏模式

磁盘：~/.hermes/logs/*.log（轮转，组可写）

读取路径 A — CLI：
  hermes logs [file] [--level] [--component] [-f]
  ↓ _read_tail() / _follow_log()

读取路径 B — Dashboard：
  GET /api/logs?file=...&level=...&component=...&search=...
  ↓ _read_tail()
  ↓ HTTP 响应 → LogsPage.tsx（5 秒轮询）
```

---

## 11. 关键文件索引

| 文件 | 行号 | 作用 |
|------|------|------|
| `hermes_logging.py` | 166 | `setup_logging()` 入口 |
| `hermes_logging.py` | 45, 76 | 线程本地会话 ID |
| `hermes_logging.py` | 130, 147 | `_ComponentFilter`、`COMPONENT_PREFIXES` |
| `hermes_logging.py` | 322 | `_ManagedRotatingFileHandler` |
| `agent/redact.py` | 326 | `redact_sensitive_text()` |
| `agent/redact.py` | 488 | `RedactingFormatter` |
| `hermes_cli/logs.py` | 140 | `tail_log()` |
| `hermes_cli/logs.py` | 280 | `_read_last_n_lines()` |
| `hermes_cli/logs.py` | 336 | `_follow_log()` |
| `hermes_cli/web_server.py` | 3728 | `GET /api/logs` 端点 |
| `web/src/pages/LogsPage.tsx` | 1 | React 日志页面 |
| `web/src/lib/api.ts` | — | `getLogs()` API 客户端 |
