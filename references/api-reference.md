# 上报 API 参考（请求示例）

`{BASE}` = 部署的 API 基址（须含 `/api/v1`），例如 `http://agent.bitsunite.xyz/api/v1`。

## 公共请求头

```http
Authorization: Bearer workspace_sk_xxxxxxxxxxxxxxxx
Content-Type: application/json
```

若当前上下文中没有**正式** API Key（空、占位符、非 `workspace_sk_` 前缀），应先请用户在 Helm 控制台或 `config.md` 中提供完整密钥后再发请求；勿使用示例占位串调用。

## POST `{BASE}/report/heartbeat`

**Body**

| 字段 | 必填 | 说明 |
|------|------|------|
| `status` | 是 | `working` \| `idle` \| `sleeping` |
| `metadata` | 否 | 任意 JSON 对象（如 CPU、内存） |

```json
{
  "status": "working",
  "metadata": { "cpu_percent": 45.2, "memory_mb": 512 }
}
```

**说明**：该路由在服务端单独限流（约每分钟 10 次，以部署为准），与 `task/*` 不同。

---

## POST `{BASE}/report/task/start`

| 字段 | 必填 | 说明 |
|------|------|------|
| `task_id` | 是 | **须为 RFC 4122 合法 UUID 字符串**（与 PG `uuid` 主键一致；建议 v4） |
| `type` | 是 | 自定义类型，如 `chat`、`tool` |

**`task_id` 规则（与 `config.md` 一致）**

- 形态示例：`550e8400-e29b-41d4-a716-446655440000`（共 36 字符，含连字符）。
- **不要**传业务侧任意字符串（如 `task_001`）：会导致入库失败，常见为 **HTTP 500**。
- 每个用户可见任务：**生成新 UUID** → `task/start` → 执行 → `task/end`（同一 `task_id`）。
- **不要**对同一 `task_id` 再次 `task/start`（主键冲突，可能 500）。

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "chat"
}
```

**注意**：同一 `task_id` 重复 start 会触发主键冲突，请每次任务使用新 UUID。

---

## POST `{BASE}/report/task/end`

| 字段 | 必填 | 说明 |
|------|------|------|
| `task_id` | 是 | 与对应 `task/start` 相同 |
| `status` | 是 | `completed` \| `failed` |
| `token_used` | 否 | 整数 |
| `result` | 否 | JSON |
| `error` | 否 | 字符串，`failed` 时可填 |

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "token_used": 1234,
  "result": { "output": "..." }
}
```

失败示例：`"status": "failed"`，`"error": "timeout"`。

---

## POST `{BASE}/report/logs`

| 字段 | 必填 | 说明 |
|------|------|------|
| `logs` | 是 | 对象数组，元素为任意键值（如 `level`、`message`、`timestamp`） |

```json
{
  "logs": [
    { "level": "info", "message": "Starting task...", "timestamp": "2026-03-24T10:00:00Z" }
  ]
}
```

当前实现：校验通过后返回成功，**不做持久化**（以仓库最新代码为准）。

---

## 成功响应示例

```json
{ "code": 0, "message": "success" }
```

## 业务错误码（上报）

| code | 含义 |
|------|------|
| 3001 | API Key 无效 |
| 3002 | 任务不存在 |
| 3003 | 任务已结束，不可重复 end |
| 4000 | 请求参数不合法 |

## curl 示例

```bash
export BASE="http://agent.bitsunite.xyz/api/v1"
export KEY="workspace_sk_你的完整密钥"

curl -sS -X POST "$BASE/report/heartbeat" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"status":"working","metadata":{}}'

curl -sS -X POST "$BASE/report/task/start" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d "{\"task_id\":\"$(uuidgen | tr '[:upper:]' '[:lower:]')\",\"type\":\"chat\"}"
```
