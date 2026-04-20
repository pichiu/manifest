# Manifest — 專案總覽與速查

## 一句話總結

Manifest 是一個開源的智慧 LLM 路由器，專為個人 AI Agent 設計。它坐落在 Agent（OpenClaw、Hermes 等）與 LLM providers（OpenAI、Anthropic、Google 等）之間，透過 23 維度評分演算法（< 2ms），將每個請求自動路由到「最便宜但足夠勝任」的模型，節省 LLM 費用最多 70%，同時提供完整的 Dashboard 追蹤 tokens、費用與訊息記錄。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Node.js | 22.x | 伺服器執行環境 |
| Backend Framework | NestJS | 11.x | HTTP 框架 + DI Container |
| ORM | TypeORM | 0.3.x | 資料庫存取（PostgreSQL） |
| 資料庫 | PostgreSQL | 16.x | 主要資料庫 |
| Authentication | Better Auth | latest | Email/password + OAuth（Google/GitHub/Discord） |
| Frontend Framework | SolidJS | latest | 管理 Dashboard SPA |
| 前端建置 | Vite | latest | HMR + 打包 |
| 圖表 | uPlot | latest | 時序圖表（成本/token 趨勢） |
| Monorepo | Turborepo + npm workspaces | 2.x | packages 依賴管理 |
| 語言 | TypeScript | 5.x（strict） | 全端統一語言 |
| 測試（後端） | Jest + Supertest | - | 單元測試 + E2E |
| 測試（前端） | Vitest | - | 元件測試 |
| CI/CD | GitHub Actions | - | CI / Docker / Release |
| 容器化 | Docker + Compose | - | 單一服務部署 |
| 版本管理 | Changesets | - | CHANGELOG + 版本號 |
| Rate Limiting | express-rate-limit + NestJS Throttler | - | 登入保護 + 全域限流 |
| HTTP Security | Helmet + strict CSP | - | 安全標頭，禁止 CDN |
| 快取 | @nestjs/cache-manager（in-memory） | - | Dashboard 快取 |

---

## 關鍵指令速查

```bash
# 依賴安裝
npm install

# 開發伺服器（兩個終端）
cd packages/backend && NODE_OPTIONS='-r dotenv/config' npx nest start --watch
cd packages/frontend && npx vite

# 生產建置
npm run build                    # Turborepo: shared → backend → frontend

# 啟動生產伺服器
npm start                        # node packages/backend/dist/main.js

# 測試
npm test --workspace=packages/backend          # Jest 單元測試
npm run test:e2e --workspace=packages/backend  # E2E（需 PostgreSQL）
npm test --workspace=packages/frontend         # Vitest
npm test --workspace=packages/shared           # Jest

# Lint & Format
npm run lint
npm run format

# 資料庫 Migration
cd packages/backend
npm run migration:generate -- src/database/migrations/DescriptiveName
npm run migration:run
npm run migration:revert
npm run migration:show

# Docker（Self-hosted 一鍵安裝）
bash <(curl -sSL https://raw.githubusercontent.com/mnfst/manifest/main/docker/install.sh)

# Docker 手動啟動
cd docker && docker compose up -d

# 新增 Changeset
npx changeset   # 選 "manifest"，填寫描述
```

---

## 文件地圖

| 文件 | 說明 |
|------|------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | 系統架構、元件圖、設計決策、Sequence Diagram |
| [CODEBASE_MAP.md](CODEBASE_MAP.md) | 程式碼地圖、「我想改 X 看哪裡」速查表、模組依賴圖 |
| [DATA_MODEL.md](DATA_MODEL.md) | 資料模型、ER 圖、Entity 說明、Migration 機制 |
| [API_SURFACE.md](API_SURFACE.md) | 所有 API 端點、認證模型、Request/Response 範例 |
| [DEV_GUIDE.md](DEV_GUIDE.md) | 開發者上手指南、環境建置、測試策略、常見踩坑 |
| [DISCOVERY_LOG.md](DISCOVERY_LOG.md) | 探索紀錄、TODO/FIXME 彙整、落差分析、技術債 |

---

## 專案術語表

| 術語 | 定義 |
|------|------|
| **Tier** | 請求複雜度等級：simple / standard / complex / reasoning |
| **Specificity** | 任務類型路由（opt-in）：coding / web_browsing / data_analysis 等 9 類 |
| **Tenant** | 用戶的資料邊界。由 `user.id` 建立，`tenant.name = user.id` |
| **Agent** | 屬於 Tenant 的 AI Agent，有唯一 OTLP 入口 key（mnfst_*） |
| **AgentApiKey** | Agent 的 OTLP 入口 key，格式為 `mnfst_*` |
| **Momentum** | Session 動量：追蹤最近 5 個 tier，防止短跟進訊息降級 |
| **Heartbeat** | 含 `HEARTBEAT_OK` 的訊息，直接路由到 simple tier，不評分 |
| **Message** | `agent_messages` 表的一行記錄，即每次 LLM 呼叫的遙測資料 |
| **Scoring** | 23 維度評分演算法，決定請求屬於哪個 Tier |
| **Fallback** | 主要模型失敗時自動嘗試備用模型（每 tier 最多 5 個） |
| **OTLP** | OpenTelemetry Protocol，Manifest 用於接收 Agent 訊息的協議格式 |
| **Subscription provider** | 使用訂閱制（非 API key）的 provider（如 ChatGPT Plus、MiniMax Coding） |
| **Cloud mode** | 使用 PostgreSQL + Better Auth 的部署模式（預設） |
| **manifest/auto** | Agent 配置為 model 名稱時 Manifest 自動路由的 model ID |

---

## 快速上手路徑

1. 閱讀 [ARCHITECTURE.md](ARCHITECTURE.md) 理解整體架構（10 分鐘）
2. 閱讀 [DEV_GUIDE.md](DEV_GUIDE.md) 搭建開發環境（20 分鐘）
3. 閱讀 [CODEBASE_MAP.md](CODEBASE_MAP.md) 找到想修改的位置
4. 深入 [DATA_MODEL.md](DATA_MODEL.md) 了解資料結構
5. 參考 [API_SURFACE.md](API_SURFACE.md) 了解 API 介面
