# Stage 1: Reconnaissance

## 專案概覽

**名稱**: Manifest  
**定位**: 個人 AI Agent 的智慧 LLM 路由器  
**倉庫**: https://github.com/mnfst/manifest  
**官方文件**: https://manifest.build/docs  
**授權**: MIT  
**狀態**: Beta  

Manifest 坐落在 AI Agent（如 OpenClaw、Hermes）與 LLM providers（OpenAI、Anthropic、Google 等）之間，對每個請求進行評分，路由到「最便宜但足夠用的」模型。每次路由在 2ms 內完成。

---

## 技術棧

| 類別 | 技術 | 版本 | 說明 |
|------|------|------|------|
| Runtime | Node.js | 22.x | 伺服器執行環境 |
| Backend Framework | NestJS | 11.x | HTTP 框架 + DI Container |
| ORM | TypeORM | 0.3.x | 資料庫存取 |
| 資料庫 | PostgreSQL | 16.x | 主要資料庫（cloud mode） |
| Authentication | Better Auth | latest | email/password + OAuth |
| Frontend Framework | SolidJS | latest | SPA |
| 建置工具 | Vite | latest | Frontend 建置 |
| 圖表 | uPlot | latest | 時序圖表 |
| Monorepo | Turborepo + npm workspaces | 2.x | 多套件管理 |
| 語言 | TypeScript | 5.x | 嚴格模式 |
| 測試（後端） | Jest + Supertest | - | 單元測試 + E2E |
| 測試（前端） | Vitest | - | 元件測試 |
| CI/CD | GitHub Actions | - | ci.yml / docker.yml / release.yml |
| 容器化 | Docker + docker-compose | - | 單一服務部署 |
| 版本管理 | Changesets | - | 版本號 + CHANGELOG |
| Rate Limiting | express-rate-limit + NestJS Throttler | - | 登入保護 + 全域限流 |
| Security | Helmet + CSP | - | HTTP 安全標頭 |
| Cache | @nestjs/cache-manager | - | 記憶體快取 |

---

## 架構模式識別

**Monorepo** with 4 packages:
- `packages/shared` — 共享 TypeScript types + constants
- `packages/backend` — NestJS API server
- `packages/frontend` — SolidJS SPA
- `packages/manifest` — 版本 shell（無程式碼，僅存放 package.json/CHANGELOG）

**部署模式**: 單一服務。NestJS 同時服務 API 與靜態前端（production）。Vite dev server 代理 /api 與 /v1 到後端（development）。

---

## 目錄結構（3 層）

```
manifest/
├── .changeset/          # Changesets 版本管理設定
├── .github/
│   ├── assets/          # Logo + Screenshot 靜態資源
│   └── workflows/       # ci.yml / docker.yml / release.yml / codeql.yml
├── .husky/              # Git hooks（pre-commit lint）
├── docker/
│   ├── Dockerfile       # Multi-stage build (Node 22 Alpine)
│   ├── docker-compose.yml  # manifest + postgres + ollama services
│   ├── .env.example     # Docker 部署環境範本
│   ├── install.sh       # 一鍵安裝腳本
│   └── DOCKER_README.md # Self-hosting 完整指南
├── packages/
│   ├── backend/
│   │   ├── src/
│   │   │   ├── main.ts           # Bootstrap: Helmet, CORS, Better Auth mount
│   │   │   ├── app.module.ts     # Root module (guards, modules)
│   │   │   ├── config/           # app.config.ts — env var 載入
│   │   │   ├── auth/             # Better Auth 設定 + SessionGuard
│   │   │   ├── database/         # DatabaseModule + Migrations (47個)
│   │   │   ├── entities/         # 17個 TypeORM entities
│   │   │   ├── analytics/        # Dashboard 分析 API
│   │   │   ├── otlp/             # OTLP 入口（AgentKeyAuthGuard）
│   │   │   ├── routing/          # LLM 路由核心（proxy + resolve + tiers）
│   │   │   ├── scoring/          # 評分引擎（23維度）
│   │   │   ├── model-prices/     # OpenRouter 定價快取
│   │   │   ├── notifications/    # 告警規則 + Email
│   │   │   ├── sse/              # Server-Sent Events
│   │   │   ├── github/           # GitHub stars endpoint
│   │   │   ├── setup/            # 初始設定 wizard
│   │   │   ├── free-models/      # 免費模型資訊
│   │   │   └── common/           # guards, utils, DTOs, constants
│   │   ├── test/                 # E2E tests (supertest)
│   │   └── .env.example
│   ├── frontend/
│   │   ├── src/
│   │   │   ├── App.tsx           # Router 設定
│   │   │   ├── components/       # 60+ 共享 UI 元件
│   │   │   ├── pages/            # 路由頁面 (Login, Overview, Routing...)
│   │   │   ├── services/         # API client, auth client, formatters
│   │   │   ├── layouts/          # Layout 元件
│   │   │   └── styles/           # CSS 主題
│   │   └── public/
│   │       └── fonts/boxicons/   # Self-hosted 字型（no CDN）
│   ├── shared/
│   │   └── src/
│   │       ├── tiers.ts          # Tier 類型 + 描述
│   │       ├── specificity.ts    # SpecificityCategory 類型
│   │       ├── agent-type.ts     # AgentType 類型
│   │       ├── subscription/     # 訂閱制 provider 設定
│   │       └── index.ts          # 統一 export
│   └── manifest/
│       ├── package.json          # 版本號所在（canonical version）
│       └── CHANGELOG.md
├── scripts/
│   └── check-api-prefix.js      # 確保 API routes 使用 /api/ 前綴
├── CLAUDE.md                    # 開發指引（最詳盡的內部文件）
├── CONTRIBUTING.md              # 貢獻者指引
├── README.md                    # 專案介紹
└── turbo.json                   # Turborepo pipeline 設定
```

---

## 既有文件掃描

### 發現的文件

| 文件 | 路徑 | 內容摘要 |
|------|------|---------|
| README.md | 根目錄 | 專案介紹、Quick start、Provider 表格、與 OpenRouter 比較 |
| CONTRIBUTING.md | 根目錄 | 技術棧、Prerequisites、Getting Started、測試方式、Workflow |
| CLAUDE.md | 根目錄 | **最詳盡的內部文件**，包含架構說明、API 端點表格、環境變數、Migration 流程 |
| docker/DOCKER_README.md | docker/ | Self-hosting 完整指南（Docker 部署） |
| packages/manifest/README.md | packages/manifest/ | 說明版本 shell 的用途 |

### 文件與程式碼落差

1. **CONTRIBUTING.md 提到 local mode（sqlite）**，但 CLAUDE.md 強調「永遠使用 cloud mode（PostgreSQL）」— 兩份文件均正確，但針對不同受眾（貢獻者 vs 部署者）有不同建議

2. **CONTRIBUTING.md L90**: 範例指令使用 `MANIFEST_MODE=local`，但 CLAUDE.md 說明 local mode 因 SQLite 鎖定問題不建議多 instance 使用 — ⚠️ 可能造成貢獻者困惑

3. **CLAUDE.md 的 API Endpoint 表格**中包含 `/api/v1/events` 但未列出所有 public-stats 和 setup 端點（程式碼有，但文件未提及）

4. **README.md** 提到 `packages/manifest` 為 CLI，但現在已是 Docker-only — README 前段已更新，但 npm 頁面仍有舊套件（⚠️ 外部資源與當前版本不一致）

---

## 規模評估

- **總檔案數**: 863
- **TypeScript 原始碼**: 719 個 `.ts`/`.tsx` 檔案
- **遷移檔案**: 47 個 TypeORM migrations
- **測試覆蓋**: 每個模組幾乎都有對應 `.spec.ts`
- **判斷**: 中型 monorepo，需完整 trace

---

## 核心業務邏輯摘要

Manifest 的價值在於：
1. **智慧評分路由**：23 維度評分（13 keyword-based + 10 structural），2ms 內完成
2. **四層 Tier 體系**：simple / standard / complex / reasoning，各層可設定 primary + 5 個 fallback 模型
3. **Specificity 路由（opt-in）**：9 個任務類型（coding / web_browsing / data_analysis / image_generation / video_generation / social_media / email_management / calendar_management / trading），覆寫 complexity tier
4. **Session Momentum**：追蹤最近 5 個 tier，避免短跟進訊息降級
5. **AES-256-GCM 加密** provider API keys 儲存於資料庫
6. **OpenAI-compatible proxy**：完整支援 streaming + fallback + Anthropic adapter + Google adapter
