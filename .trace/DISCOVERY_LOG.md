# Manifest 探索紀錄與待解問題

> 撰寫日期：2026-04-20  
> 涵蓋版本：5.25.3（截至 2026-04）  
> 探索範圍：packages/backend、packages/frontend、packages/shared

---

## 1. Web Search 發現摘要

### 官方文件與社群資源

| 資源 | URL | 說明 |
|------|-----|------|
| 官方文件 | https://manifest.build/docs/introduction | Scoring 演算法、Tier 定義、Self-hosting 指南 |
| GitHub 倉庫 | https://github.com/mnfst/manifest | MIT 授權，Beta 狀態 |
| Docker Hub | https://hub.docker.com/r/manifestdotbuild/manifest | 主要發布管道（multi-arch amd64+arm64） |
| Discord 社群 | https://discord.gg/FepAked3W7 | 技術支援頻道 |
| GitHub Discussions | https://github.com/mnfst/manifest/discussions | 路線圖討論（如 #951 新增 Z.ai、MiniMax） |

### 關鍵架構決策的背景

- **23 維度評分**：官方文件確認 13 keyword + 10 structural，< 2ms，採用 Aho-Corasick 演算法。分數邊界（-0.1 / 0.08 / 0.35）來自程式碼，文件未公開校準依據。
- **Self-hosted vs Cloud 二元模式**：Self-hosted 時 Manifest 完全本地化，agent 訊息內容不離開本機；Cloud 模式只儲存 metadata（tokens、cost、tier）。
- **移除獨立 OTLP telemetry**（近期變動）：遙測資料現在只透過 routing proxy 收集，不再需要獨立的 OTLP 端點設定。
- **2 步驟 Onboarding**：舊版為多步驟引導，現已簡化為雲端與本地一致的 2 步驟流程。
- **Session Momentum 設計動機**：根據 DEV Community 文章，conversation depth 增加時 LLM 呼叫成本會因 context 累積而上升；momentum 機制確保短跟進訊息不被誤降至 simple tier，避免不必要的模型切換。

---

## 2. 既有文件與程式碼落差清單

| # | 文件說 | 程式碼實際情況 | 位置 |
|---|--------|--------------|------|
| 1 | CONTRIBUTING.md 建議以 `MANIFEST_MODE=local` 啟動後端（L87-91）| CLAUDE.md 明確說明 local mode 因多 instance SQLite 鎖定問題，開發期間應永遠使用 cloud mode（PostgreSQL） | `CONTRIBUTING.md:87-91` vs `CLAUDE.md` |
| 2 | CLAUDE.md API Endpoint 表格列出 17 個路由 | 程式碼中額外存在 `GET/POST /api/v1/setup/*`（`SetupController`）與 `GET /api/v1/public/*`（`PublicStatsController`，含 usage、free-models、provider-tokens、free-providers 四個子路由）未列入表格 | `packages/backend/src/setup/setup.controller.ts:6-28`、`packages/backend/src/public-stats/public-stats.controller.ts:53-111` |
| 3 | recon.md 記錄 `packages/backend/src/setup/` 與 `packages/backend/src/free-models/` 目錄存在 | CLAUDE.md 的 Project Structure 未列出 `setup/` 與 `free-models/` 模組 | `packages/backend/src/app.module.ts:26-27` |
| 4 | 環境變數文件（CLAUDE.md）列出 `MAILGUN_API_KEY` 與 `MAILGUN_DOMAIN` 作為 email 設定 | 程式碼實際支援 `EMAIL_PROVIDER`（resend / mailgun / sendgrid）+ `EMAIL_API_KEY` + `EMAIL_DOMAIN` 的統一介面；舊式 `MAILGUN_*` 變數為向下相容保留 | `packages/backend/src/notifications/services/email-providers/send-email.ts` |
| 5 | CLAUDE.md 說明 Analytics 使用 TypeORM QueryBuilder | `setup/setup.service.ts` 與 `database/database-seeder.service.ts` 仍直接使用 `dataSource.query()` 搭配 `$1, $2` 佔位符（raw SQL） | `packages/backend/src/setup/setup.service.ts:64-138`、`packages/backend/src/database/database-seeder.service.ts:77-113` |
| 6 | CLAUDE.md 說「定價快取不寫資料庫，純記憶體」| `public-stats/public-stats.controller.ts:42` 有 module-level `cachedFree` 變數用於快取 free models 回應，屬於額外的記憶體快取層，未在文件中說明 | `packages/backend/src/public-stats/public-stats.controller.ts:42` |

---

## 3. 程式碼中的 TODO / FIXME / HACK 彙整

搜尋結果：在 `packages/backend/src`、`packages/frontend/src`、`packages/shared/src` 中（排除 `.spec.` 測試檔）**未發現任何 TODO、FIXME、HACK、XXX 標注**。

雖然沒有明確的技術債標記，下列程式碼模式值得注意：

| 嚴重程度 | 描述 | 位置 |
|---------|------|------|
| 中 | Email provider 使用 `console.warn/error/log` 而非 NestJS `Logger` | `notifications/services/email-providers/sendgrid.provider.ts:4-6`、`resend.provider.ts`、`mailgun.provider.ts` |
| 中 | `setup.service.ts` 使用 `eslint-disable-next-line @typescript-eslint/no-require-imports` 抑制 lint 規則，表示存在動態 `require()` | `packages/backend/src/setup/setup.service.ts:122` |
| 低 | `auth/auth.instance.ts` 同樣有 `eslint-disable-next-line @typescript-eslint/no-require-imports`，為 dynamic import 繞過 ESM 限制 | `packages/backend/src/auth/auth.instance.ts:15` |
| 低 | `sql-dialect.ts` 中遇到未知 interval 字串時使用 `console.warn`（fallback 到 24 小時），屬於 defensive coding 但缺少告警機制 | `packages/backend/src/common/utils/sql-dialect.ts:45` |

---

## 4. 未解答的技術疑問

### Q1：Scoring 維度 weight 如何校準？有 A/B testing 數據嗎？

`scoring/config.ts` 中的 weight 值（如 `formalLogic: 0.07`、`simpleIndicators: 0.08`）直接硬編碼，程式碼中沒有任何校準注解或實驗數據。Tier 邊界（-0.1 / 0.08 / 0.35）亦同。目前無法從程式碼判斷這些數值是否來自 empirical testing、手動調整或直覺設計。

### Q2：Session Momentum 的 30 分鐘 TTL 是如何決定的？

`session-momentum.service.ts:10` 的 `TTL_MS = 30 * 60 * 1000` 為 magic number，程式碼與文件中皆無設計說明。這是否可透過環境變數覆寫？對不同 agent 使用情境（如長時間暫停後繼續對話）是否有不同的最佳值？

### Q3：為何 Better Auth 使用獨立的 `pg.Pool` 而非共用 TypeORM 的連接池？

`auth/auth.instance.ts:19` 建立了獨立的 `new Pool({ connectionString: databaseUrl })`。Better Auth 框架需要直接操作資料庫（管理自己的 session tables），與 TypeORM 的 DataSource 不相容。這意味著在高負載時，兩個連接池（TypeORM + Better Auth）合計可能遠超過 `DB_POOL_MAX`（預設 20）的限制。是否有總體連接數的管控機制？

### Q4：Specificity 路由啟用後，Complexity Scoring 是否完全被繞過？

根據 `data_flow.md` 描述，specificity 檢查優先於 complexity scoring。但當 agent 同時啟用多個 specificity categories，且某條訊息命中多個 category 時，解析順序為何？程式碼中 `resolve.service.ts` 的多 category 衝突處理邏輯需進一步確認。

### Q5：記憶體快取在多 instance（水平擴展）場景下的行為？

`session-momentum.service.ts`、`routing-cache.service.ts`、`proxy-rate-limiter.ts` 等多個服務都使用 in-memory `Map`。在 Docker Swarm 或 Kubernetes 多 replica 部署時，不同 instance 的快取彼此不同步。目前的部署文件（`railway.toml`）是否假設單一 instance？官方有無 Redis 整合的路線圖？

### Q6：`ThoughtSignatureCache` 的跨請求持久化策略是什麼？

`routing/proxy/thought-signature-cache.ts` 使用 in-memory Map 快取 Claude Extended Thinking 的 tool_call signature。如果 Node.js process 重啟，這些 signature 會遺失。對於長對話中的 thinking blocks，重啟後是否會造成 API 錯誤？

### Q7：`ProxyMessageDedup` 的 write lock 機制是否完整處理競態條件？

`proxy-message-dedup.ts:23` 的 `successWriteLocks` 使用 Promise-based write lock 防止重複 INSERT。但這只在單一 Node.js process 內有效，無法跨 instance 防止競態。若使用多 replica 部署，是否依賴資料庫的 unique constraint 作為最終防線？

---

## 5. 已知技術債

### 需要重構的區域

| 區域 | 問題描述 | 影響 |
|------|---------|------|
| `setup/setup.service.ts` | 直接使用 `dataSource.query()` + raw SQL（`$1, $2` 佔位符）；與其他模組改用 QueryBuilder 的方向不一致 | 維護性差，跨 DB 方言支援弱 |
| `database/database-seeder.service.ts` | 同上，直接使用 raw SQL 操作 `"user"` 表（Better Auth 管轄的表）| 與 Better Auth schema 耦合，升級 Better Auth 時有破壞風險 |
| Email provider logging | 三個 email provider（Mailgun、Resend、SendGrid）使用 `console.log/warn/error` 而非 NestJS `Logger`，日誌無法整合到 NestJS 的結構化日誌系統 | 生產環境可觀測性差 |
| `public-stats/public-stats.controller.ts` | module-level `cachedFree` 變數在 NestJS DI 框架外管理快取狀態，違反 NestJS 的依賴注入慣例 | 測試困難，快取無法透過 DI 替換 |

### 可能的效能瓶頸

| 瓶頸點 | 說明 | 嚴重程度 |
|--------|------|---------|
| 多 in-memory Map 快取 | `routing-cache.service.ts`、`session-momentum.service.ts`、`proxy-rate-limiter.ts`、`think-block-cache.ts` 等多個 in-memory Map 無共享快取後端，水平擴展時失效 | 高（若需多 replica） |
| Better Auth 獨立連接池 | 與 TypeORM 各自維護連接池，高並發時可能超過 PostgreSQL 最大連接數 | 中 |
| OpenRouter 定價快取 | 純記憶體 Map，啟動時同步、每日更新。若 OpenRouter API 在同步窗口內不可用，cost 計算可能使用過期資料 | 低（有保留舊快取的 fallback） |
| `ProxyMessageDedup` write lock | 使用 Promise chain 序列化寫入，高吞吐場景（streaming + OTLP 並發）可能形成排隊瓶頸 | 低（單 instance 場景） |

---

## 6. 需要進一步調查的區域

1. **`setup/` 模組的完整用途**：`SetupController` 提供 `GET /api/v1/setup/status` 與 `POST /api/v1/setup/admin`，應為首次部署的管理員帳號建立流程。此模組在文件中完全缺失，需確認它如何與 `SEED_DATA` 機制互動，以及兩者是否互斥。

2. **`free-models/` 與 `public-stats/` 模組的關係**：`PublicStatsController` 的 `/api/v1/public/free-models` 端點注入了 `FreeModelsService`，但 `FreeModel` interface 定義在 `public-stats.service.ts`。模組邊界不清晰，需釐清 `free-models/` 模組的獨立職責。

3. **Anthropic Extended Thinking（思考 tokens）的計費邏輯**：`thinking-block-cache.ts` 快取思考 blocks，但 `proxy-message-recorder.ts` 中的 cost 計算是否正確處理 thinking tokens 的定價（Anthropic 對 thinking tokens 有特殊定價規則）？

4. **`x-manifest-specificity` header 的驗證**：specificity-detector 支援 agent 透過 header 強制指定任務類型，但此 header 的驗證是否有白名單限制？是否允許任意字串繞過 specificity routing 邏輯？

5. **Fallback 觸發後的 `fallback_from_model` 欄位填寫**：`CLAUDE.md` 提及 `selectMessageRowColumns()` 包含 `fallback_from_model`，需確認 fallback 鏈（primary → fallback 1 → fallback 2）中，每一層的 `fallback_from_model` 欄位是否正確記錄前一層的模型名稱。

---

## 7. 建議向維護者確認的問題清單

1. **Scoring weight 校準**：dimensions 的 weight 值（如 `formalLogic: 0.07`）是否有對應的 benchmark dataset 或 A/B testing 結果？是否計劃支援 per-agent 的 scoring config 覆寫？

2. **Session Momentum TTL**：30 分鐘 TTL 是否可透過環境變數配置？對於 long-running session（如整日使用的 coding agent）是否有更長 TTL 的需求？

3. **多 instance 部署支援**：目前所有快取均為 in-memory，官方路線圖是否有 Redis adapter 計劃？Railway 部署是否官方只支援單 replica？

4. **Better Auth 連接池上限**：TypeORM pool（預設 20）+ Better Auth pool 的合計連接數是否有上限設定？高並發生產環境下的推薦 `DB_POOL_MAX` 值為何？

5. **`setup/` 模組的生命週期**：首次部署後 `POST /api/v1/setup/admin` 是否應被關閉？是否有防止重複建立 admin 的冪等保護？（程式碼中使用了 `pg_advisory_lock`，但鎖定後的行為需確認。）

6. **Anthropic Extended Thinking 計費**：thinking tokens 是否納入 cost 計算？`model-pricing-cache` 是否從 OpenRouter 取得 thinking tokens 的定價？

7. **`CONTRIBUTING.md` 的 local mode 指引**：對於多 Claude Code instance 的開發環境，`CONTRIBUTING.md:87-91` 的 `MANIFEST_MODE=local` 指引是否應更新以反映 CLAUDE.md 的建議（永遠使用 cloud mode）？

---

> ⚠️ 以上部分資訊（特別是 Web Search 來源）無法在本機程式碼中直接驗證，標記為 ⚠️ 的項目需進一步確認。
