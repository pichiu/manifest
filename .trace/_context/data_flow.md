# Stage 2.2: Request / Data Flow

## 代表性 Use Case：LLM 代理請求完整流程

### 場景：個人 AI Agent (OpenClaw) 傳送一個編程問題

**請求進入點**：`POST /v1/chat/completions`  
**認證**：`Authorization: Bearer mnfst_xxxxxxxx`

---

## 完整流程追蹤

```
1. HTTP Request 抵達
   ├── proxy.controller.ts:55 — ProxyController.chatCompletions()
   ├── AgentKeyAuthGuard 驗證 Bearer token
   │   ├── 先查記憶體 cache（TTL 5分鐘）
   │   ├── Cache miss → DB 查詢 agent_api_keys 表（by key_prefix）
   │   ├── scrypt 驗證 key_hash
   │   └── 設定 request.ingestionContext { tenantId, agentId, agentName, userId }
   │
2. 速率限制檢查
   ├── rateLimiter.checkLimit(userId)    — 每用戶 in-memory 限制
   ├── rateLimiter.checkIpLimit(req.ip)  — 每 IP in-memory 限制
   └── rateLimiter.acquireSlot(userId)   — 並發槽位控制
   │
3. 代理請求處理（proxy.service.ts:81）
   ├── sanitizeNullContent(messages)  — 清理 null content
   ├── enforceLimits()                 — 檢查 tenant 的 token/cost 限制規則
   │   └── LimitCheckService.checkLimits() → 若超限回傳 friendly 訊息
   │
4. 評分（resolve.service.ts:24）
   ├── 篩選評分訊息（排除 system/developer role，只保留最後 10 條）
   ├── 偵測 heartbeat（訊息含 HEARTBEAT_OK → 直接路由到 simple tier）
   ├── 取得 session momentum（最近 5 個 tier）
   │
   ├── [specificity 路由 — 優先]
   │   ├── specificityService.getActiveAssignments(agentId)
   │   ├── scanMessages() — 掃描訊息偵測任務類型
   │   └── 若命中 → 直接返回 specificity 結果，跳過 complexity scoring
   │
   └── [complexity 評分]
       ├── scoreRequest(input, config, momentum)  — scoring/index.ts:155
       │   ├── 提取 user 訊息文字
       │   ├── 短訊息（< 50字）快速判斷
       │   ├── checkFormalLogicOverride → 若有 formal logic → reasoning tier
       │   ├── KeywordTrie.scan() — O(n) Aho-Corasick 全文掃描
       │   ├── 評分 13 keyword 維度 + 10 structural 維度
       │   ├── applyMomentum() — session 動量調整
       │   └── scoreToTier() via sigmoid function → tier
       │
       ├── tierService.getTiers(agentId) — 取得該 agent 的 tier 設定
       ├── providerKeyService.getEffectiveModel() — 取得有效模型
       └── resolveProvider() — 多策略 provider 解析
           ├── 1. 從模型名稱前綴推斷（如 anthropic/claude-...）
           ├── 2. 查 discovered models cache
           └── 3. fallback 到 pricing cache
   │
5. API Key 解析
   ├── providerKeyService.getProviderApiKey() — 從 DB 取 encrypted key
   ├── AES-256-GCM 解密 api_key_encrypted
   └── OAuth 特殊處理（OpenAI ChatGPT / MiniMax 訂閱 → OAuth token）
   │
6. Provider 代理轉發（proxy.service.ts:162）
   ├── fallbackService.tryForwardToProvider()
   │   ├── 選擇 provider adapter（Anthropic/Google/Standard）
   │   ├── HTTP 轉發到真實 LLM API
   │   └── 支援 streaming（SSE）
   │
7. Fallback 邏輯（若主要請求失敗）
   ├── shouldTriggerFallback(status) — 特定 HTTP 狀態才觸發
   ├── 從 tier assignment 取得 fallback_models 清單（最多 5 個）
   └── fallbackService.tryFallbacks() — 逐一嘗試
   │
8. 回應處理（proxy-response-handler.ts）
   ├── buildMetaHeaders() — X-Manifest-Tier, X-Manifest-Model 等 header
   ├── Streaming: handleStreamResponse() — 即時轉發 SSE chunks
   │   ├── 解析 provider-specific streaming 格式
   │   ├── 轉換為 OpenAI 標準格式（Anthropic/Google adapter）
   │   └── 收集 usage 資訊（input/output tokens）
   └── Non-streaming: handleNonStreamResponse()
   │
9. 訊息記錄（proxy-message-recorder.ts）
   ├── 計算 cost_usd（基於 OpenRouter 定價快取）
   ├── 去重邏輯（ProxyMessageDedup）— 避免重複寫入
   ├── INSERT 到 agent_messages 表
   │   ├── 所有 routing metadata（tier, reason, model, provider...）
   │   ├── token 使用量
   │   └── caller attribution（OpenClaw/Hermes/SDK 等）
   └── IngestEventBusService.emit(userId) — 觸發 SSE 推送
```

---

## 資料轉換各層

| 層 | 輸入格式 | 輸出格式 |
|---|---------|---------|
| Agent → Manifest | OpenAI-compatible JSON | 同左 |
| Scoring | messages array | tier + confidence + reason |
| Anthropic adapter | OpenAI messages | Anthropic Messages API format |
| Google adapter | OpenAI messages | Gemini generateContent format |
| Response | Provider-specific JSON | OpenAI-compatible JSON |
| Streaming | Provider SSE chunks | OpenAI SSE format |

---

## 資料庫寫入時機

- 每個 HTTP 請求（成功/失敗/fallback）都寫一條 `agent_messages` 記錄
- Fallback 情況：primary 寫 `fallback_error`，fallback 成功寫 `ok`
- Rate limit 429：每個 agent 冷卻 60 秒（不重複寫入）

---

## Dashboard 分析流程

```
前端請求 GET /api/v1/overview
  → OverviewController
  → AggregationService 
  → TypeORM QueryBuilder（addTenantFilter 按 userId 過濾）
  → PostgreSQL 聚合查詢（GROUP BY, date_trunc）
  → 回傳 MetricWithTrend（value + trend_pct）
```

SSE 即時更新：
```
訊息記錄 → IngestEventBusService.emit(userId)
  → SseController 訂閱者收到事件
  → 前端 EventSource 更新 Dashboard
```
