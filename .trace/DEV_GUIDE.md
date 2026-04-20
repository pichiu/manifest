# Manifest 開發者上手指南

> 最後更新：2026-04-20｜版本：5.x

Manifest 是個人 AI Agent 的智慧 LLM 路由器，以 NestJS + SolidJS monorepo 實作，透過 23 維度評分在 2ms 內決定最合適的模型。本指南說明如何在本地建置開發環境、執行測試、以及提交 PR。

---

## 1. Prerequisites

| 工具 | 版本 | 說明 |
|------|------|------|
| Node.js | 22.x (LTS) | 執行環境 |
| npm | 10.x | 套件管理 |
| PostgreSQL | 16.x | 主要資料庫（建議用 Docker） |
| Docker | 任意近期版本 | 啟動 PostgreSQL container |

> **注意**：開發環境必須使用 PostgreSQL（`MANIFEST_MODE=cloud`，也是預設值）。不要使用 local/SQLite 模式，多個 Claude instance 並行時會發生 SQLite 鎖定衝突。

---

## 2. 本地開發環境建置

### 步驟 1：Clone 與安裝

```bash
git clone https://github.com/<your-username>/manifest.git
cd manifest
npm install
```

npm workspaces 會自動安裝 `packages/backend`、`packages/frontend`、`packages/shared`、`packages/manifest` 的所有相依套件。

### 步驟 2：用 Docker 啟動 PostgreSQL

每次開發 session 建議使用新建的資料庫，避免跨 session 的資料污染：

```bash
# 確認 postgres_db container 正在執行（若不存在則建立）
docker start postgres_db 2>/dev/null || \
  docker run -d --name postgres_db \
    -e POSTGRES_USER=myuser \
    -e POSTGRES_PASSWORD=mypassword \
    -e POSTGRES_DB=mydatabase \
    -p 5432:5432 \
    postgres:16

# 建立本次 session 專用的資料庫（唯一名稱避免衝突）
DB_NAME="manifest_dev_$(openssl rand -hex 4)"
docker exec postgres_db psql -U myuser -d postgres -c "CREATE DATABASE $DB_NAME;"
echo "使用資料庫：$DB_NAME"
```

### 步驟 3：設定 .env

```bash
cp packages/backend/.env.example packages/backend/.env
```

以下是最小化開發設定，將 `DATABASE_URL` 中的 `<DB_NAME>` 替換為上一步建立的名稱：

```env
PORT=3001
BIND_ADDRESS=127.0.0.1
NODE_ENV=development
BETTER_AUTH_SECRET=<執行 openssl rand -hex 32 產生>
DATABASE_URL=postgresql://myuser:mypassword@localhost:5432/<DB_NAME>
API_KEY=dev-api-key-12345
SEED_DATA=true
```

產生 `BETTER_AUTH_SECRET`：

```bash
openssl rand -hex 32
```

### 步驟 4：啟動 Backend 與 Frontend（各開一個終端）

**終端 1 — Backend**

```bash
cd packages/backend
NODE_OPTIONS='-r dotenv/config' npx nest start --watch
```

`NODE_OPTIONS='-r dotenv/config'` 是必要的，因為 `auth/auth.instance.ts` 在 TypeScript import 時（早於 NestJS `ConfigModule`）就讀取 `process.env`，必須確保 `.env` 已預先載入。

**終端 2 — Frontend**

```bash
cd packages/frontend
npx vite
```

Frontend 在 `http://localhost:3000` 啟動，所有 `/api/*` 和 `/v1/*` 請求會由 Vite proxy 轉發到 `http://localhost:3001`。

> **注意**：`npm run dev`（Turborepo）只啟動 frontend，backend 需如上方所示獨立啟動。

### 步驟 5：登入驗證

設定 `SEED_DATA=true` 後，啟動時會自動植入 demo 資料：

- **Dashboard URL**：`http://localhost:3000`
- **帳號**：`admin@manifest.build`
- **密碼**：`manifest`

Demo agent 名稱為 `demo-agent`，OTLP key 為 `dev-otlp-key-001`。

---

## 3. 測試策略與執行方式

### 3.1 單元測試（Jest）

Backend 各模組的 `.spec.ts` 檔案：

```bash
# 在根目錄執行
npm test --workspace=packages/backend

# 包含覆蓋率報告
cd packages/backend && npx jest --coverage
```

Shared 套件：

```bash
npm test --workspace=packages/shared
```

### 3.2 E2E 測試（Supertest + 真實 DB）

E2E 測試對真實 PostgreSQL 執行 HTTP 請求，需要一個乾淨的測試資料庫：

```bash
npm run test:e2e --workspace=packages/backend
# 等同於：cd packages/backend && jest --config ./test/jest-e2e.json --runInBand
```

`--runInBand` 強制序列執行，避免測試間的資料庫競態條件。

> **注意**：新增 TypeORM entity 到 `database.module.ts` 時，同樣需要加到 `packages/backend/test/helpers.ts` 的 entities 陣列，否則 E2E 測試會出現 `EntityMetadataNotFoundError`。

### 3.3 Frontend 測試（Vitest）

```bash
npm test --workspace=packages/frontend

# 包含覆蓋率報告
cd packages/frontend && npx vitest run --coverage
```

### 3.4 覆蓋率要求

**所有 PR 必須維持 100% 行覆蓋率。** 這是強制要求，不是建議。

- 新增的 service、guard、controller、utility → 必須有對應的 `.spec.ts` 且每一行都被覆蓋
- Error path 也必須被測試覆蓋
- PR 的 patch coverage 必須達到 100%
- CI 會透過 Codecov 自動檢查，低於標準的 PR 不會被 merge

一次全跑：

```bash
npm test --workspace=packages/shared
npm test --workspace=packages/backend
npm run test:e2e --workspace=packages/backend
npm test --workspace=packages/frontend
```

---

## 4. 常見踩坑與 Debugging

### 4.1 auth.instance.ts 在 NestJS 前初始化

**症狀**：`BETTER_AUTH_SECRET is not set` 或 Better Auth 初始化失敗，即使 `.env` 存在。

**原因**：`auth/auth.instance.ts` 在 TypeScript module 載入時就執行 `betterAuth()`，此時 NestJS 的 `ConfigModule` 尚未載入 `.env`。

**解法**：啟動 backend 時必須預載 dotenv：

```bash
# 正確
NODE_OPTIONS='-r dotenv/config' npx nest start --watch

# 錯誤（NestJS ConfigModule 太晚載入）
npx nest start --watch
```

### 4.2 Better Auth 的 trusted origins 設定

**症狀**：OAuth callback 失敗，或在 `:3000` 登入後 session 無效。

**原因**：Better Auth 的 OAuth callback URL 指向 `BETTER_AUTH_URL`（預設為 `http://localhost:3001`）。Social login 在 Vite dev server（`:3000`）上無法正常運作，必須透過 backend port（`:3001`）進行，即使用 production build。

**解法**：
- 開發時使用 email/password 登入即可（走 `:3000` 正常）
- 若要測試 OAuth，直接訪問 `http://localhost:3001`（production build）
- `FRONTEND_PORT` env var 可追加額外的 trusted origin port

### 4.3 Migration 必須手動產生（synchronize: false）

**症狀**：修改 entity 後資料庫 schema 沒有更新，或啟動時出現 column 不存在錯誤。

**原因**：`synchronize` 永久設定為 `false`，TypeORM 不會自動同步 schema。

**正確流程**：

```bash
cd packages/backend

# 1. 修改 entity 後，產生 migration
npm run migration:generate -- src/database/migrations/DescriptiveName

# 2. 在 database.module.ts 的 migrations 陣列中 import 新 migration

# 3. 執行 migration（dev 環境會在啟動時自動執行）
npm run migration:run

# 查看 migration 狀態
npm run migration:show
```

**重要**：每個 migration 檔案的 timestamp 必須唯一，不可複用或修改現有 timestamp。

### 4.4 Provider API key 加密導致無法直接讀 DB

**症狀**：在資料庫中查看 `user_providers.api_key_encrypted` 欄位，看到的是密文而非明文 API key。

**原因**：Provider API key 使用 AES-256-GCM 加密後才存入資料庫（`common/utils/crypto.util.ts`），加密 key 來自 `BETTER_AUTH_SECRET`。

**除錯方式**：透過 API 端點操作 provider 資料，不要直接操作資料庫欄位。Agent OTLP key 同樣用 scrypt KDF hash 後存於 `agent_api_keys.key_hash`。

### 4.5 CSP 規則（self-only，不能載入 CDN）

**症狀**：新增外部字型或 icon 庫後，瀏覽器 console 出現 CSP violation，資源無法載入。

**原因**：`main.ts` 中的 Helmet 設定了嚴格的 CSP，只允許 `'self'` 來源，禁止任何外部 CDN。

**正確做法**：

1. 下載字型/CSS 到 `packages/frontend/public/fonts/` 或 `public/` 目錄
2. 修改 CSS 中的 URL 為相對路徑（`./filename.woff`）
3. 在 `index.html` 用本地路徑引用（`<link href="/fonts/..." />`）
4. 不要修改 CSP 設定允許外部 domain

---

## 5. Contribution Workflow

### 5.1 Branching Model

```
main ← 所有 PR 的目標分支
  └── feature/your-feature-name
  └── fix/issue-description
  └── chore/internal-change
```

從 `main` 建立功能分支：

```bash
git checkout main && git pull
git checkout -b feature/your-feature-name
```

### 5.2 PR Convention

- 每個 PR 專注於單一變更
- PR 標題清楚說明改了什麼（使用現在式，e.g. "Add token cost breakdown to overview page"）
- PR description 包含：改了什麼、為什麼改、相關 issue（如有）
- PR 必須通過全部 CI checks 才能 merge

提交 PR 前的 checklist：

```bash
# 1. 測試全部通過
npm test --workspace=packages/shared
npm test --workspace=packages/backend
npm run test:e2e --workspace=packages/backend
npm test --workspace=packages/frontend

# 2. Production build 正常
npm run build

# 3. Lint 通過
cd packages/backend && npm run lint
```

### 5.3 Changeset 使用（只選 "manifest" 套件）

Changeset 用於記錄需要出現在 CHANGELOG 的變更。不是每個 PR 都需要，純內部/工具性的改動可以跳過。

```bash
npx changeset
# 互動流程：
#   → 選擇 "manifest"（唯一有效選項）
#   → 選擇 patch / minor / major
#   → 輸入一行摘要（這會成為 CHANGELOG entry）
```

**重要**：永遠選 `manifest` 套件。`manifest-backend`、`manifest-frontend`、`manifest-shared` 雖然在列表中，但它們被設定為 ignore，選了也不會生效。

產生的 `.changeset/*.md` 檔案需要跟程式碼一起 commit。

不需要 CHANGELOG entry 但想明確標記的：

```bash
npx changeset add --empty
```

### 5.4 CI Checks 說明

每個 PR 開啟後，GitHub Actions 會自動執行：

| Check | 說明 | 何時失敗 |
|-------|------|---------|
| `ci.yml` — tests | Jest + Vitest，含覆蓋率 | 測試失敗或覆蓋率下降 |
| `ci.yml` — lint | ESLint | 有 lint 錯誤 |
| `ci.yml` — typecheck | TypeScript strict mode | 型別錯誤 |
| `docker.yml` — build | Docker image 建置驗證 | Dockerfile 或 build 問題 |
| `codecov/patch` | 新增程式碼的覆蓋率 | patch coverage < 100% |
| `codecov/project` | 整體覆蓋率 | 整體覆蓋率下降超過 1% |
| `changeset-check` | 提醒是否需要 changeset | 僅警告，不封鎖 merge |

---

## 6. 開發模式 vs 生產模式的差異

| 項目 | 開發模式（`NODE_ENV=development`） | 生產模式（`NODE_ENV=production`） |
|------|----------------------------------|----------------------------------|
| CORS | 啟用，`CORS_ORIGIN` 指定的 origin | 停用 |
| Frontend serving | Vite dev server（`:3000`）proxy 到 backend（`:3001`） | NestJS `@nestjs/serve-static` 直接服務 `frontend/dist/` |
| Database migration | 開發/測試環境自動執行 | 需設定 `AUTO_MIGRATE=true` 或手動執行 |
| Agent key auth | Loopback IP（127.x.x.x）不需要有效的 `mnfst_*` key | 必須有有效的 Agent API key |
| Login rate limiting | 20 次 / 15 分鐘（`/api/auth/sign-in`） | 同左 |
| Trust proxy | 停用 | `trust proxy 1`（Nginx/Railway 後的 reverse proxy） |
| SEED_DATA | `true` → 植入 demo 資料 | 可選用於首次 self-hosting |

**本地 loopback auth** 的詳細說明：  
`AgentKeyAuthGuard` 在 loopback IP（`127.x.x.x`）收到的請求中，允許任何非 `mnfst_` 開頭的 Bearer token 通過（開發/測試用）。正式 production 環境中，必須提供透過 Dashboard 建立的有效 `mnfst_*` key。

---

## 7. 快速參考

### 常用指令

```bash
# 啟動（各開一個終端）
cd packages/backend && NODE_OPTIONS='-r dotenv/config' npx nest start --watch
cd packages/frontend && npx vite

# 全部測試
npm test --workspace=packages/shared && \
npm test --workspace=packages/backend && \
npm run test:e2e --workspace=packages/backend && \
npm test --workspace=packages/frontend

# Migration 管理
cd packages/backend
npm run migration:generate -- src/database/migrations/MigrationName
npm run migration:run
npm run migration:show
npm run migration:revert

# Production build
npm run build && npm start
```

### 資源連結

| 資源 | URL |
|------|-----|
| GitHub | https://github.com/mnfst/manifest |
| 官方文件 | https://manifest.build/docs |
| Docker Hub | https://hub.docker.com/r/manifestdotbuild/manifest |
| Discord | https://discord.gg/FepAked3W7 |
| GitHub Discussions | https://github.com/mnfst/manifest/discussions |
