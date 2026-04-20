# Manifest API 介面參考

> 文件版本：2026-04-20 ｜ 適用版本：Manifest Beta（PostgreSQL cloud mode）

Manifest 以 **OpenAI-compatible HTTP endpoint** 的形式坐落於 AI Agent 與 LLM provider 之間，同時提供 Dashboard 管理 API。本文件描述所有對外暴露的 HTTP 端點、認證機制、請求/回應格式，以及錯誤處理模式。

---

## 1. 認證模型

Manifest 有三種認證機制，依呼叫方身份選擇對應方式。

### 1.1 機制一：Better Auth Cookie Session（Dashboard 用戶）

Dashboard 前端用戶透過 Better Auth 登入後取得 cookie session。

- **登入端點**：`POST /api/auth/sign-in/email`
- **Cookie 名稱**：`better-auth.session_token`（HttpOnly，SameSite=Lax）
- **OAuth 支援**：Google、GitHub、Discord（需設定環境變數）
- **適用端點**：所有 `/api/v1/*` Dashboard 管理 API

### 1.2 機制二：X-API-Key Header（程式化存取）

使用 `API_KEY` 環境變數設定的靜態 key，用於 server-to-server 整合。

- **Header 名稱**：`X-API-Key`
- **比較方式**：timing-safe compare（防止 timing attack）
- **適用端點**：所有 `/api/v1/*` Dashboard 管理 API（與 session 互為備援）

### 1.3 機制三：Bearer mnfst_* Token（Agent OTLP/Proxy 入口）

每個 Agent 建立時自動生成的 `mnfst_` 開頭 token，專供 Agent 程序呼叫。

- **Header 格式**：`Authorization: Bearer mnfst_<token>`
- **儲存方式**：資料庫中以 scrypt KDF hash 儲存（原始 key 僅在建立時回傳一次）
- **快取**：`AgentKeyAuthGuard` 在記憶體快取有效 key 5 分鐘
- **適用端點**：`POST /v1/chat/completions`、`POST /api/v1/routing/resolve`、`POST /api/v1/routing/subscription-providers`、`GET /api/v1/agent/*`

### 認證流程圖

```mermaid
flowchart TD
    REQ([HTTP Request]) --> PUB{@Public\n裝飾器?}
    PUB -->|是| HANDLER([Route Handler])
    PUB -->|否| SG[SessionGuard\n檢查 Better Auth cookie]
    SG -->|session 有效| HANDLER
    SG -->|無 session| AKG[ApiKeyGuard\n檢查 X-API-Key header]
    AKG -->|key 有效| HANDLER
    AKG -->|無效| E401([401 Unauthorized])

    AGENT([Agent Request]) --> AGPUB[@Public + AgentKeyAuthGuard]
    AGPUB --> BEARER{Authorization:\nBearer mnfst_*}
    BEARER -->|hash 比對成功| CTX[注入 IngestionContext\nagentId / userId / tenantId]
    CTX --> HANDLER
    BEARER -->|無效| E401

    subgraph 速率限制
        HANDLER --> THROTTLE[ThrottlerGuard\n100 req/60s 全域]
        THROTTLE --> DONE([回應])
    end
```

---

## 2. 完整 API 端點表格

### 2.1 公開端點（無需認證）

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/v1/health` | 健康檢查，回傳 `{ status: "ok" }` |
| ALL | `/api/auth/*` | Better Auth（登入/註冊/OAuth/session 管理） |
| GET | `/api/v1/setup/status` | 初始設定狀態（是否需要建立第一個 admin） |
| POST | `/api/v1/setup/admin` | 建立第一個 admin 帳號（setup wizard 用） |
| GET | `/api/v1/github/stars` | GitHub star 數（快取） |
| GET | `/api/v1/public-stats` | 公開統計數據 |

### 2.2 Dashboard API（Session 或 X-API-Key）

#### 概覽與分析

| Method | Path | Query Params | 說明 |
|--------|------|-------------|------|
| GET | `/api/v1/overview` | `range`, `agent_name` | Dashboard 概覽（token/cost/message 摘要 + 時序資料） |
| GET | `/api/v1/tokens` | `range`, `agent_name` | Token 用量分析 |
| GET | `/api/v1/costs` | `range`, `agent_name` | 費用分析 |
| GET | `/api/v1/messages` | `range`, `provider`, `service_type`, `cost_min`, `cost_max`, `limit`, `cursor`, `agent_name` | 分頁 Message log |
| GET | `/api/v1/messages/:id/details` | — | Message 詳細資訊 |
| PATCH | `/api/v1/messages/:id/feedback` | — | 設定 Message 評分（Body: `rating`, `tags`, `details`） |
| DELETE | `/api/v1/messages/:id/feedback` | — | 清除 Message 評分 |
| GET | `/api/v1/security` | — | Security score + 事件列表 |
| GET | `/api/v1/model-prices` | — | 所有模型定價（從 OpenRouter 快取） |
| GET | `/api/v1/events` | — | SSE 即時事件流（僅支援 Session 認證） |

#### Agent 管理

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/v1/agents` | Agent 列表（含 sparkline 資料） |
| POST | `/api/v1/agents` | 建立 Agent + 生成 API Key |
| GET | `/api/v1/agents/:agentName/key` | 取得 Agent API Key（keyPrefix + 可能的完整 key） |
| POST | `/api/v1/agents/:agentName/rotate-key` | 輪換 Agent API Key |
| PATCH | `/api/v1/agents/:agentName` | 重新命名 Agent / 更新分類 |
| DELETE | `/api/v1/agents/:agentName` | 刪除 Agent |

#### 路由設定（Routing Config）

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/v1/routing/:agentName/status` | 路由狀態（是否可路由，原因） |
| GET | `/api/v1/routing/:agentName/providers` | 已連接的 provider 列表 |
| POST | `/api/v1/routing/:agentName/providers` | 連接 / 更新 provider |
| DELETE | `/api/v1/routing/:agentName/providers/:provider` | 移除 provider |
| POST | `/api/v1/routing/:agentName/providers/deactivate-all` | 停用所有 provider |
| GET | `/api/v1/routing/:agentName/tiers` | 取得 Tier 設定 |
| PUT | `/api/v1/routing/:agentName/tiers/:tier` | 設定 Tier Primary Model |
| DELETE | `/api/v1/routing/:agentName/tiers/:tier` | 清除 Tier Override |
| POST | `/api/v1/routing/:agentName/tiers/reset-all` | 重設所有 Tier |
| GET | `/api/v1/routing/:agentName/tiers/:tier/fallbacks` | 取得 Tier Fallback 列表 |
| PUT | `/api/v1/routing/:agentName/tiers/:tier/fallbacks` | 設定 Tier Fallback 模型 |
| DELETE | `/api/v1/routing/:agentName/tiers/:tier/fallbacks` | 清除 Tier Fallback |
| GET | `/api/v1/routing/:agentName/specificity` | 取得 Specificity 設定 |
| PUT | `/api/v1/routing/:agentName/specificity/:category` | 設定 Specificity Primary Model |
| POST | `/api/v1/routing/:agentName/specificity/:category/toggle` | 啟用/停用 Specificity Category |
| DELETE | `/api/v1/routing/:agentName/specificity/:category` | 清除 Specificity Override |
| PUT | `/api/v1/routing/:agentName/specificity/:category/fallbacks` | 設定 Specificity Fallback |
| DELETE | `/api/v1/routing/:agentName/specificity/:category/fallbacks` | 清除 Specificity Fallback |
| POST | `/api/v1/routing/:agentName/specificity/reset-all` | 重設所有 Specificity |
| GET | `/api/v1/routing/:agentName/available-models` | 已連接 provider 的可用模型列表 |
| POST | `/api/v1/routing/:agentName/refresh-models` | 重新探索所有 provider 的模型 |
| GET | `/api/v1/routing/pricing-health` | OpenRouter 定價快取狀態 |
| POST | `/api/v1/routing/pricing/refresh` | 手動刷新定價快取 |
| POST | `/api/v1/routing/ollama/sync` | 手動同步 Ollama 本地模型 |

#### Notification（告警規則）

| Method | Path | 說明 |
|--------|------|------|
| GET/POST | `/api/v1/notifications` | 列出/建立告警規則（Query: `agent_name`） |
| PATCH/DELETE | `/api/v1/notifications/:id` | 更新/刪除告警規則 |
| GET | `/api/v1/notifications/logs` | 通知發送 log（Query: `agent_name`） |
| GET/POST/DELETE | `/api/v1/notifications/email-provider` | Email Provider CRUD |
| POST | `/api/v1/notifications/email-provider/test` | 測試 Email Provider（未存 or 已存） |
| GET/POST | `/api/v1/notifications/notification-email` | 取得/設定通知收件地址 |
| POST | `/api/v1/notifications/trigger-check` | 手動觸發 threshold 檢查 |

### 2.3 Agent 入口（Bearer mnfst_* Token）

| Method | Path | 說明 |
|--------|------|------|
| POST | `/v1/chat/completions` | LLM Proxy，完整 OpenAI-compatible 介面 |
| POST | `/api/v1/routing/resolve` | 僅解析路由決策（不實際呼叫 LLM） |
| POST | `/api/v1/routing/subscription-providers` | 註冊訂閱制 provider（ChatGPT Plus 等） |
| GET | `/api/v1/agent/usage` | 當前 Agent 的 Token 用量（Query: `range`） |
| GET | `/api/v1/agent/costs` | 當前 Agent 的費用資料（Query: `range`） |

---

## 3. 核心端點詳細說明

### 3.1 `POST /v1/chat/completions`（LLM Proxy）

這是 Manifest 最核心的端點。Agent 以標準 OpenAI 格式發送請求，Manifest 在 2ms 內評分並路由至最適合的 LLM。

**認證**：`Authorization: Bearer mnfst_<token>`

**Request Body**（OpenAI Chat Completions 格式）：
```json
{
  "model": "auto",
  "messages": [
    { "role": "user", "content": "幫我寫一個 Python quicksort" }
  ],
  "stream": false,
  "max_tokens": 2048
}
```

**特殊 Request Headers**：

| Header | 說明 |
|--------|------|
| `x-session-key` | 跨請求的 session 識別（用於 Anthropic thinking block 快取），預設 `"default"` |
| `x-manifest-specificity` | 強制使用特定 specificity category（覆蓋自動偵測） |
| `traceparent` | W3C Trace Context，用於 trace ID 擷取 |

**Response Headers（X-Manifest-* 元資料）**：

| Header | 說明 |
|--------|------|
| `X-Manifest-Tier` | 路由決策的 tier（`simple`/`standard`/`complex`/`reasoning`） |
| `X-Manifest-Model` | 實際使用的模型 ID |
| `X-Manifest-Provider` | 實際使用的 provider |
| `X-Manifest-Confidence` | 路由置信度（0–1） |
| `X-Manifest-Reason` | 路由原因（`scored`/`specificity`/`heartbeat` 等） |
| `X-Manifest-Specificity` | Specificity category（若 specificity 路由生效） |
| `X-Manifest-Fallback-From` | 若發生 fallback，此為原始模型 ID |
| `X-Manifest-Fallback-Index` | Fallback 順序（0-based） |
| `X-Manifest-Fallback-Exhausted` | `"true"` 表示所有 fallback 均失敗 |

**成功 Response**（非 stream）：
```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "created": 1745000000,
  "model": "claude-3-5-sonnet-20241022",
  "choices": [{
    "index": 0,
    "message": { "role": "assistant", "content": "..." },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 45,
    "completion_tokens": 312,
    "total_tokens": 357
  }
}
```

**Streaming**：設定 `"stream": true` 後回傳 `text/event-stream`，格式與 OpenAI SSE 相同。

---

### 3.2 `GET /api/v1/overview`

Dashboard 概覽，回傳多個聚合指標。

**Query Params**：

| 參數 | 類型 | 預設 | 說明 |
|------|------|------|------|
| `range` | string | `24h` | 時間範圍：`1h`/`24h`/`7d`/`30d` |
| `agent_name` | string | — | 過濾特定 Agent（可選） |

**Response**：
```json
{
  "summary": {
    "tokens_today": 125430,
    "cost_today": 0.0847,
    "messages": 42,
    "services_hit": { "total": 0, "healthy": 0, "issues": 0 }
  },
  "token_usage": [...],
  "cost_usage": [...],
  "message_usage": [...],
  "cost_by_model": [
    { "model": "claude-3-5-sonnet", "cost": 0.05, "provider": "anthropic" }
  ],
  "recent_activity": [...],
  "active_skills": [...],
  "has_data": true,
  "has_providers": true
}
```

> 回應有 `UserCacheInterceptor` 快取（依用戶），TTL = `DASHBOARD_CACHE_TTL_MS`。

---

### 3.3 `GET /api/v1/messages`

分頁查詢 Message log，支援多維度過濾。

**Query Params**：

| 參數 | 類型 | 說明 |
|------|------|------|
| `range` | string | 時間範圍（`1h`/`24h`/`7d`/`30d`） |
| `provider` | string | 過濾 LLM provider |
| `service_type` | string | 過濾 service type |
| `cost_min` | number | 最低費用過濾 |
| `cost_max` | number | 最高費用過濾 |
| `limit` | number | 每頁筆數，最大 200，預設 50 |
| `cursor` | string | 游標分頁（上一頁回傳的 `next_cursor`） |
| `agent_name` | string | 過濾特定 Agent |

**Response**：
```json
{
  "messages": [
    {
      "id": "uuid",
      "model": "claude-3-5-sonnet-20241022",
      "provider": "anthropic",
      "routing_tier": "complex",
      "routing_reason": "scored",
      "specificity_category": "coding",
      "auth_type": "api_key",
      "fallback_from_model": null,
      "prompt_tokens": 45,
      "completion_tokens": 312,
      "cost": 0.00187,
      "created_at": "2026-04-20T10:00:00Z"
    }
  ],
  "next_cursor": "cursor_string_or_null",
  "total": 128
}
```

---

### 3.4 `POST /api/v1/agents`

建立新 Agent，同時生成 `mnfst_*` API Key。

**Request Body**：
```json
{
  "name": "my-coding-agent",
  "agent_category": "coding",
  "agent_platform": "openclaw"
}
```

- `name`：必填，自動 slugify（轉小寫、空白換 `-`）
- `agent_category`：選填，Agent 任務類型
- `agent_platform`：選填（`openclaw`/`hermes`/`openai-sdk` 等）

**Response**：
```json
{
  "agent": {
    "id": "uuid",
    "name": "my-coding-agent",
    "display_name": "my-coding-agent",
    "agent_category": "coding",
    "agent_platform": "openclaw"
  },
  "apiKey": "mnfst_xxxxxxxxxxxxxxxxxxxx"
}
```

> `apiKey` 為明文，**僅回傳一次**。請立即儲存，後續無法取回完整 key（只能取得前綴）。

---

### 3.5 Routing Config CRUD

#### 連接 Provider

```
POST /api/v1/routing/:agentName/providers
```

**Request Body**：
```json
{
  "provider": "anthropic",
  "apiKey": "sk-ant-xxx",
  "authType": "api_key",
  "region": null
}
```

- `authType` 可為 `api_key`（預設）或 `subscription`
- `region` 僅支援 Alibaba/Qwen provider（值：`auto`/`singapore`/`us`/`beijing`）

**Response**：`{ "id": "uuid", "provider": "anthropic", "auth_type": "api_key", "is_active": true, "region": null }`

#### 設定 Tier Primary Model

```
PUT /api/v1/routing/:agentName/tiers/:tier
```

tier 值：`simple` / `standard` / `complex` / `reasoning`

**Request Body**：
```json
{
  "model": "claude-3-5-haiku-20241022",
  "provider": "anthropic",
  "authType": "api_key"
}
```

#### 設定 Specificity Override

```
PUT /api/v1/routing/:agentName/specificity/:category
```

category 值：`coding` / `web_browsing` / `data_analysis` / `image_generation` / `video_generation` / `social_media` / `email_management` / `calendar_management` / `trading`

**Request Body**：
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "provider": "anthropic",
  "authType": "api_key"
}
```

#### 啟用/停用 Specificity Category

```
POST /api/v1/routing/:agentName/specificity/:category/toggle
```

**Request Body**：`{ "active": true }`

---

## 4. Error Handling Pattern

### 4.1 Friendly Response（Provider 未設定或 Limit 超出）

當 Agent 尚未設定 provider、或超過 token/cost 上限時，Manifest **不回傳 HTTP error**，而是以標準 OpenAI chat completion 格式回傳一則引導訊息（`HTTP 200`），讓 Agent 可以直接展示給用戶，不會崩潰：

```json
{
  "id": "chatcmpl-manifest-<uuid>",
  "object": "chat.completion",
  "model": "manifest",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "🦚 請至 Manifest Dashboard 設定 LLM provider..."
    },
    "finish_reason": "stop"
  }],
  "usage": { "prompt_tokens": 0, "completion_tokens": 0, "total_tokens": 0 }
}
```

Streaming 模式下同樣以 SSE 格式（`text/event-stream`）回傳此訊息。

### 4.2 Provider Error Response

當 upstream LLM provider 回傳錯誤（且 fallback 亦已耗盡）時，回應包含：

```json
{
  "error": {
    "message": "...",
    "type": "fallback_exhausted",
    "status": 503,
    "primary_model": "gpt-4o",
    "primary_provider": "openai",
    "attempted_fallbacks": [
      { "model": "gpt-4o-mini", "provider": "openai", "status": 503 }
    ]
  }
}
```

同時設定 response header `X-Manifest-Fallback-Exhausted: true`。

### 4.3 Rate Limit Error（429）

超過速率限制時，回傳標準 HTTP 429，**不轉換為 friendly response**：

```json
{
  "error": {
    "message": "Too Many Requests",
    "type": "proxy_error"
  }
}
```

### 4.4 伺服器端錯誤（5xx）

伺服器內部錯誤：

```
[🦚 Manifest] Something broke on our end. Try again in a moment.
```

此訊息以 friendly response 格式（HTTP 200）回傳，避免破壞 Agent 的訊息流。

---

## 5. Rate Limiting & Pagination

### 5.1 Rate Limiting

| 層級 | 規則 | 適用範圍 |
|------|------|---------|
| 全域 ThrottlerGuard | 100 req / 60s / 用戶 | 所有 Dashboard API |
| 登入保護 | 20 req / 15min / IP | `POST /api/auth/sign-in` |
| Proxy Rate Limiter | 依 userId 與 IP 雙重限流 | `POST /v1/chat/completions` |

Proxy 端的速率限制透過 `ProxyRateLimiter` 服務管理，同時追蹤 concurrent slot 數（防止同一 Agent 發送大量並行請求）。

### 5.2 Pagination（游標分頁）

`GET /api/v1/messages` 使用**游標分頁（cursor-based pagination）**，不支援 offset：

- 首次請求：不帶 `cursor`
- 後續頁：帶上一頁回傳的 `next_cursor`（`null` 表示最後一頁）
- `limit` 最大值為 200，預設 50
- 排序固定為 `created_at DESC`

### 5.3 Dashboard 快取

Dashboard 分析 API 啟用 `UserCacheInterceptor`（cache key = userId），TTL = `DASHBOARD_CACHE_TTL_MS`（記憶體快取）。Agent 建立/刪除時手動 invalidate 相關 key。

---

## 6. 常見請求範例

### 設定 Agent 並發送第一個請求

```bash
# 1. 建立 Agent → 取得 apiKey: "mnfst_xxx"
curl -X POST http://localhost:3001/api/v1/agents \
  -H "X-API-Key: dev-api-key-manifest-001" -H "Content-Type: application/json" \
  -d '{ "name": "my-agent" }'

# 2. 連接 Provider
curl -X POST http://localhost:3001/api/v1/routing/my-agent/providers \
  -H "X-API-Key: dev-api-key-manifest-001" -H "Content-Type: application/json" \
  -d '{ "provider": "anthropic", "apiKey": "sk-ant-xxx" }'

# 3. 透過 Proxy 發送請求
curl -X POST http://localhost:3001/v1/chat/completions \
  -H "Authorization: Bearer mnfst_xxx" -H "Content-Type: application/json" \
  -d '{ "model": "auto", "messages": [{ "role": "user", "content": "Hello!" }] }'
```

### 查詢路由決策（不呼叫 LLM）

```bash
curl -X POST http://localhost:3001/api/v1/routing/resolve \
  -H "Authorization: Bearer mnfst_xxx" -H "Content-Type: application/json" \
  -d '{ "messages": [{ "role": "user", "content": "Write a React component" }], "recentTiers": ["standard"], "specificity": null }'
# → 回傳 { tier, model, provider, reason, confidence, ... }
```
