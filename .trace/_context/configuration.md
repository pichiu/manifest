# Stage 2.6: 設定與環境

## 設定載入機制

```
process.env（Node.js process）
  ↑
dotenv（-r dotenv/config 或 dotenv/config require）
  ↑  
packages/backend/.env 或 docker/.env
```

⚠️ **重要**: `auth/auth.instance.ts` 在 TypeScript import time 讀取 `process.env`，早於 NestJS `ConfigModule`。因此：
- `.env` 必須在 Node 啟動時就已載入（`NODE_OPTIONS='-r dotenv/config'`）
- NestJS 的 `ConfigService` 讀取的是**同一個** `process.env`，不需要 dotenv 再次載入

---

## 環境變數完整清單

### 必填

| 變數 | 說明 | 範例 |
|------|------|------|
| `BETTER_AUTH_SECRET` | Session 簽名密鑰（min 32 chars） | `openssl rand -hex 32` |
| `DATABASE_URL` | PostgreSQL 連線字串 | `postgresql://user:pass@host:5432/db` |

### 伺服器設定

| 變數 | 預設 | 說明 |
|------|------|------|
| `PORT` | `3001` | 監聽 port |
| `BIND_ADDRESS` | `127.0.0.1` | 監聽位址（Docker/Railway 用 `0.0.0.0`） |
| `NODE_ENV` | `development` | 環境（影響 CORS、auth 行為） |
| `CORS_ORIGIN` | `http://localhost:3000` | 允許的 CORS origin |
| `BETTER_AUTH_URL` | `http://localhost:PORT` | Better Auth base URL（OAuth callback 用） |
| `FRONTEND_PORT` | - | 額外的可信任 origin port |

### 資料庫

| 變數 | 預設 | 說明 |
|------|------|------|
| `AUTO_MIGRATE` | `false`（dev/test 自動執行） | 啟動時自動執行 migration |
| `DB_POOL_MAX` | `20` | PostgreSQL 連線池大小 |

### 速率限制

| 變數 | 預設 | 說明 |
|------|------|------|
| `THROTTLE_TTL` | `60000` | 限流窗口（ms） |
| `THROTTLE_LIMIT` | `100` | 窗口內最大請求數 |

### 認證 API（可選）

| 變數 | 說明 |
|------|------|
| `API_KEY` | X-API-Key header 認證密鑰 |

### 電子郵件（可選，選其一）

| 變數 | 說明 |
|------|------|
| `EMAIL_PROVIDER` | `resend` / `mailgun` / `sendgrid` |
| `EMAIL_API_KEY` | Email provider API key |
| `EMAIL_DOMAIN` | 發送 domain（Mailgun 需要） |
| `EMAIL_FROM` | 寄件人位址 |

### OAuth（可選，皆需 ID + SECRET）

| 變數 | 說明 |
|------|------|
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google OAuth |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | GitHub OAuth |
| `DISCORD_CLIENT_ID` / `DISCORD_CLIENT_SECRET` | Discord OAuth |

### 資料植入

| 變數 | 說明 |
|------|------|
| `SEED_DATA` | `true` → 植入 demo 資料（admin@manifest.build / manifest） |

### 部署

| 變數 | 說明 |
|------|------|
| `MANIFEST_EMBEDDED` | `true` → 跳過 bootstrap（嵌入模式） |
| `MANIFEST_FRONTEND_DIR` | 自訂前端靜態檔目錄 |
| `FRAME_ANCESTORS` | CSP frame-ancestors（逗號分隔） |
| `OLLAMA_HOST` | Ollama 服務位址（Docker: `http://ollama:11434`） |

---

## 優先順序與覆寫

```
1. process.env（最高優先）
2. .env 檔案（dotenv 載入）
3. 程式預設值（app.config.ts 中的 ?? 預設）
```

---

## Migration 管理

**工具**: TypeORM CLI
**設定**: `packages/backend/src/database/datasource.ts`

| 指令 | 說明 |
|------|------|
| `npm run migration:generate -- src/database/migrations/Name` | 自動從 entity 差異產生 |
| `npm run migration:run` | 執行 pending migrations |
| `npm run migration:revert` | 回滾上一個 |
| `npm run migration:show` | 顯示狀態 |

**規則**：
- `synchronize: false`（永不自動同步）
- 所有 schema 變更必須透過 migration
- 新 migration 必須在 `database.module.ts` 的 `migrations` 陣列中 import
- Timestamp 必須唯一，不可複用

---

## Feature Flags

目前無正式 feature flag 系統。透過環境變數控制功能：
- `SEED_DATA=true` → 啟用 demo 資料
- `AUTO_MIGRATE=true` → 啟用自動 migration
- `NODE_ENV=development` → 啟用 CORS、dev loopback auth

---

## Secrets 管理

| Secret | 儲存方式 | 說明 |
|--------|---------|------|
| BETTER_AUTH_SECRET | env var | session 簽名 |
| DATABASE_URL | env var | 含密碼 |
| Provider API Keys | DB（AES-256-GCM 加密） | `user_providers.api_key_encrypted` |
| Agent OTLP Keys | DB（scrypt hash） | `agent_api_keys.key_hash` |
| Email provider keys | DB（AES-256-GCM 加密） | `email_provider_configs.key_encrypted` |

AES-256-GCM 加密工具：`common/utils/crypto.util.ts`  
scrypt KDF：`common/utils/hash.util.ts`
