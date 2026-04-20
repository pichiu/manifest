# Stage 2.4: Extension Points

## 1. Provider 擴充

### 新增 LLM Provider（4 步驟）

Provider Registry 是單一真相來源：`common/constants/providers.ts`

```
步驟 1: PROVIDER_REGISTRY（providers.ts）
  → 加入 ProviderRegistryEntry { id, displayName, aliases, openRouterPrefixes }

步驟 2: FetcherConfig（model-discovery/provider-model-fetcher.service.ts）
  → 設定 /models API 端點、認證方式、解析邏輯

步驟 3: ProviderEndpoint（routing/proxy/provider-endpoints.ts）
  → 設定 chat/completions 端點 URL

步驟 4: ProviderDef（frontend/src/services/providers.ts）
  → 設定顯示名稱、icon、UI 說明
```

**不需要**：migration、entity 變更（provider 以字串儲存）

---

## 2. Specificity 類別擴充

### 新增任務類型（6 步驟）

```
步驟 1: SPECIFICITY_CATEGORIES（shared/src/specificity.ts）
  → 加入新類別 ID

步驟 2: DEFAULT_KEYWORDS（scoring/keywords.ts）
  → 加入對應 keyword 陣列（weight=0，只用於 specificity 偵測）

步驟 3: DEFAULT_CONFIG（scoring/config.ts）
  → 加入 { name, weight: 0, direction: 'up', keywords: ... }

步驟 4: DIMENSION_MAP（scoring/specificity-detector.ts）
  → 加入 category → dimensions 的映射

步驟 5（可選）: TOOL_NAME_PATTERNS（specificity-detector.ts）
  → 加入 tool 名稱前綴映射

步驟 6: SPECIFICITY_STAGES（frontend/src/services/providers.ts）
  → 加入 StageDef { category, label, description, icon }
```

`specificity_assignments` 表和 UI 元件自動支援新類別，**不需要 migration**。

---

## 3. 評分維度擴充

`scoring/config.ts` 的 `DEFAULT_CONFIG.dimensions` 陣列可直接新增：
```typescript
{ name: 'newDimension', weight: 0.03, direction: 'up', keywords: [...] }
```

或非 keyword 的 structural 維度：
```typescript
{ name: 'structuralDimension', weight: 0.02, direction: 'up' }
// 並在 scoring/index.ts 的 STRUCTURAL_SCORERS Map 加入對應 scorer 函式
```

---

## 4. Email Provider 擴充

**位置**: `notifications/services/email-providers/`

支援格式：unified (`EMAIL_PROVIDER` env var)
- 現有：`resend`, `mailgun`, `sendgrid`
- 新增：在 `send-email.ts` 加入新的 provider case

每個 provider 只需實作 `sendEmail({ to, subject, html, text })` 介面。

---

## 5. Agent Type 擴充

**位置**: `shared/src/agent-type.ts`

新增支援的 agent 框架：
1. 加入 `AGENT_TYPES` 常數陣列
2. 在 `frontend/src/components/FrameworkSnippets.tsx` 加入設定片段
3. 在 `frontend/src/components/AgentTypeGrid.tsx` 加入 UI card

---

## 6. API Endpoint 擴充

標準 NestJS 模組模式：
1. 建立 `entity/*.entity.ts`（若需要新資料表）
2. 建立 `new-feature/*.module.ts`, `*.controller.ts`, `*.service.ts`
3. 在 `database.module.ts` 加入 entity
4. 執行 `npm run migration:generate` 產生 migration
5. 在 `app.module.ts` imports 陣列加入新 module

**強制要求**：
- 所有 API routes 必須有 `/api/` 前綴（`scripts/check-api-prefix.js` 會在 CI 驗證）
- 需 100% 測試覆蓋率

---

## 7. Middleware / Guard 擴充

**現有 Guard Chain**（不能隨意更改順序）：
```
SessionGuard → ApiKeyGuard → ThrottlerGuard
```

`@Public()` 裝飾器跳過 Session 和 API Key 驗證。
`@SkipThrottle()` 裝飾器跳過速率限制。
`@UseGuards(AgentKeyAuthGuard)` 用於 proxy 端點（mnfst_* key 驗證）。

---

## 8. 自訂 Provider（Custom Provider）

使用者可透過 Dashboard 加入任何 OpenAI-compatible endpoint：
- Entity: `entities/custom-provider.entity.ts`
- Service: `routing/custom-provider/custom-provider.service.ts`
- 資料儲存在 `custom_providers` 表
- Routing 時自動識別並代理到自訂端點

---

## Hook / Event 擴充點

| 擴充點 | 位置 | 說明 |
|--------|------|------|
| IngestEventBus | `common/services/ingest-event-bus.service.ts` | RxJS Subject，訊息記錄後 emit |
| SSE Events | `sse/sse.service.ts` | 訂閱 IngestEventBus，推送到前端 |
| Cron Jobs | `notifications/` + `database/pricing-sync.service.ts` | NestJS @Cron 裝飾器 |
