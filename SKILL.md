---
name: agent-workspace-report-api
description: "Agent Workspace 上报 HTTP 契约：`/api/v1/report/*`，Bearer + `workspace_sk_` API Key。若无有效 Key，Agent 必须先向用户索取，不得空跑请求。含心跳、task/start、task/end、logs；task_id 须 UUID；仅 heartbeat 限流。排障 3001/3002/3003/4000。不涉及 JWT 管理接口。"
---

# Agent Workspace 上报 API

## 何时加载本 skill

- 实现或排查 **Agent 进程** 的 HTTP 上报（任意语言 / SDK）。
- 对照控制台下发的 **config.md** 核对路径、Header、字段。
- 处理上报错误码（尤其 **3001–3099**、**4000**）或 **401/429**。

## 不在范围内

- 用户登录、创建 Agent、拉取列表等 **JWT** 接口（`/api/v1/auth`、`/users`、`/agents`…）。
- 详细示例与 curl 见 `references/api-reference.md`。

## Base URL

- 以部署为准；生产示例：`http://agent.bitsunite.xyz/api/v1`（上报路径为 `{BASE}/report/...`）。
- 本地/其它环境以 `PUBLIC_API_URL`、`NEXT_PUBLIC_API_URL` 或 `config.md` 中的 **Endpoint** 为准。

## 认证

| 项 | 要求 |
|----|------|
| Header | `Authorization: Bearer <完整 API Key>` |
| Key 形态 | 必须以 `workspace_sk_` 开头（与中间件、数据库哈希校验一致） |
| 常见失败 | `code === 3001` → Key 无效或格式不对（HTTP **401**） |

所有上报接口均需：`Content-Type: application/json`。

## 缺少有效 API Key 时（Agent 必读）

在发起任意 `report` 请求**之前**，须确认已拿到**正式**接入密钥；若不具备，**必须先请用户补充**，禁止用空值、占位符或猜测值调用接口。

**视为「没有正式 API Key」的情况（须停手并提醒用户）**

- 环境变量 / 配置 / `config.md` 中 **未提供**、为空、仅空白。
- 仅有占位文案，例如：`your_api_key`、`xxx`、`...`、`<API_KEY>`、`changeme`、`example`、`请替换` 等。
- 字符串 **不以 `workspace_sk_` 开头**（长度明显不对、截断、缺前缀）。
- 用户明确表示尚未在 Helm（天枢）控制台创建 Agent 或未保存密钥。

**Agent 应怎么做**

1. **不要**直接调用上报接口试运气；也**不要**在对话里编造 Key。
2. 用简短中文说明：上报需要 Helm 控制台创建的 Agent **完整 API Key**（`workspace_sk_` 开头），可从创建成功页、`config.md` 或环境变量中读取。
3. 请用户**粘贴或配置**正式 Key 后再继续；若平台支持 AskUserQuestion 等交互，应用来收集密钥（敏感信息避免写入无关日志）。
4. 若请求已发出且返回 **401 / `code === 3001`**：同样视为 Key 无效或过期，**提醒用户核对 Key**（是否复制完整、是否在控制台重置过），勿重复盲重试。

**正式 Key 的最低检查**：非空 + 前缀 `workspace_sk_` + 总长度合理（明显过短视为无效）。

## 统一响应

- 成功：`code === 0`，`message` 多为 `"success"`；多数上报接口无业务 `data`。
- 失败：响应体含 `code`、`message`；同时看 HTTP 状态码。

## 接口一览

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/report/heartbeat` | 心跳（**单独限流**，见下） |
| POST | `/report/task/start` | 任务开始 |
| POST | `/report/task/end` | 任务结束 |
| POST | `/report/logs` | 日志（可选；当前实现校验后即成功，不落库） |

## 硬约束（与后端校验一致）

1. **`task_id`**：`task/start` 与 `task/end` 使用**同一** ID。数据库主键为 **UUID** 类型，须传 **合法 UUID 字符串**（建议 v4）；随意字符串可能导致创建失败或查询异常。
2. **任务生命周期**：任意一次业务任务都应 **先 `task/start` → 执行 → 再 `task/end`**；`end` 时任务须仍为「进行中」，否则 **3003**。
3. **`task/end`**：`status` 只能是 `completed` 或 `failed`；与 `task/start` 的 `task_id` 必须对应已存在且仍为 running 的任务，否则 **3002**（不存在）。
4. **心跳**：`status` 只能是 `working` | `idle` | `sleeping`。

## 限流（以当前路由为准）

- **仅** `POST /report/heartbeat` 套有限流（实现为 Redis，**约每分钟 10 次**量级，以部署配置为准）。
- `task/start`、`task/end`、`logs` 无额外 heartbeat 同组限流；若全链路 429，应降频 + 退避重试。

## 推荐调用顺序

1. 周期 **heartbeat**（产品侧常建议约 **30s**；注意与上述限流匹配，避免过密）。
2. 每个用户可见任务：**task/start** → 业务逻辑 → **task/end**（成功/失败均上报 `failed`/`completed`）。
3. 可选批量 **logs**。

## 错误码速查（上报域）

| code | 含义 |
|------|------|
| 3001 | API Key 无效 |
| 3002 | 任务不存在（`task/end` 的 `task_id` 无记录） |
| 3003 | 任务已结束，不可重复 end |
| 4000 | 请求体非法（缺字段、枚举不符、JSON 无效等） |

HTTP **401** 多与 3001 同时出现；结构校验失败多为 **400** + 4000。

## 与 Dashboard 的边界

控制台、统计、Agent 管理走 **用户 JWT**，与 Agent 的 **API Key** 上报链路分离；勿混用 Token 类型。
