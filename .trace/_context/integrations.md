# Stage 2.5: 外部整合

## 1. LLM Providers（核心整合）

**位置**: `routing/proxy/`

### OpenAI-compatible Providers（直接轉發）
- OpenAI、DeepSeek、Mistral、Moonshot、xAI、MiniMax、Qwen、Z.ai、OpenRouter
- 端點定義：`provider-endpoints.ts`
- 轉發方式：直接 HTTP PASS-THROUGH，request/response format 不變

### Anthropic（需要 adapter）
- **位置**: `routing/proxy/anthropic-adapter.ts`
- OpenAI messages format → Anthropic Messages API
- 主要差異：`system` role 分離、`content` 格式、`max_tokens` 必填
- Thinking tokens（Claude Extended Thinking）：特殊 `thinking` block 處理
- `ThoughtSignatureCache`：快取 tool_call 的 signature，跨請求傳遞

### Google Gemini（需要 adapter）
- **位置**: `routing/proxy/google-adapter.ts`
- OpenAI messages format → Gemini `generateContent` format
- `parts` 結構、`role: model` 轉換
- 串流格式轉換

### GitHub Copilot（OAuth flow）
- **位置**: `routing/oauth/copilot-device-auth.service.ts`
- Device Authorization Grant flow（適合無頭 CLI）
- 透過 `/api/v1/routing/copilot/auth/init` 啟動 Device Code flow
- `CopilotTokenService`：快取短期 access token

### OpenAI ChatGPT 訂閱（OAuth flow）
- **位置**: `routing/oauth/openai-oauth.service.ts`
- 支援 ChatGPT Plus/Pro/Team 訂閱（不需要 API key）
- Authorization Code flow

### MiniMax 訂閱
- **位置**: `routing/oauth/minimax-oauth.service.ts`
- MiniMax Coding Plan 訂閱
- 自訂 OAuth flow

---

## 2. OpenRouter（定價來源）

**位置**: `database/pricing-sync.service.ts`, `model-prices/model-pricing-cache.service.ts`

- 每日從 OpenRouter `/api/v1/models` 端點同步
- **不需要 API key**（public endpoint）
- 回傳所有 LLM 的定價資訊
- 用途：cost 計算、tier auto-assign
- 儲存方式：純記憶體（Map），不寫 DB

失敗處理：
- HTTP 錯誤 → 保留現有快取，記錄 warning log
- timeout 30 秒

---

## 3. Ollama（本地 LLM）

**位置**: `database/ollama-sync.service.ts`

- `OLLAMA_HOST` env var 設定端點（Docker compose 中預設 `http://ollama:11434`）
- 定期從 Ollama `/api/tags` 同步本地模型清單
- 支援 Ollama Cloud（不同端點）

---

## 4. Better Auth（身份驗證）

**位置**: `auth/auth.instance.ts`

- email/password 登入
- Google、GitHub、Discord OAuth
- 管理自己的 session tables（`user`, `session`, `account`, `verification` 等）
- 使用獨立 pg.Pool（與 TypeORM 不共享）

---

## 5. 電子郵件服務

**位置**: `notifications/services/email-providers/send-email.ts`

統一介面，支援三個 provider：

| Provider | 啟用條件 | API 方式 |
|---------|---------|---------|
| Resend | `EMAIL_PROVIDER=resend` + `EMAIL_API_KEY` | REST API |
| Mailgun | `EMAIL_PROVIDER=mailgun` 或舊式 `MAILGUN_API_KEY` | REST API |
| SendGrid | `EMAIL_PROVIDER=sendgrid` + `EMAIL_API_KEY` | REST API |

用途：
- Better Auth 的 email 驗證和密碼重設
- threshold alert 通知

---

## 6. GitHub Stars API

**位置**: `github/github.controller.ts`

- `GET /api/v1/github/stars` → 查詢 GitHub API 取得 star 數
- @Public() 端點（無需認證）
- 快取結果

---

## 7. Railway 部署

**位置**: `railway.toml`

```toml
[build]
builder = "dockerfile"
dockerfilePath = "docker/Dockerfile"

[deploy]
startCommand = "node packages/backend/dist/main.js"
healthcheckPath = "/api/v1/health"
```

---

## 失敗處理策略彙整

| 整合點 | 失敗處理 |
|--------|---------|
| LLM Provider | Fallback 到下一個模型（最多 5 個），失敗碼白名單觸發 |
| OpenRouter 定價同步 | 保留舊快取，記錄 warning |
| Ollama 同步 | 靜默失敗，下次同步重試 |
| Email 發送 | 記錄 error log，不阻塞主流程（async void） |
| GitHub Stars | 快取過期後查詢失敗 → 返回 null |
| Better Auth | 由 Better Auth 框架自行處理 |
| DB 連線 | TypeORM 連接池（max 20），PG healthcheck |
