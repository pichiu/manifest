# Stage 2.3: 核心領域邏輯

## 1. 評分引擎（Scoring Engine）

**位置**: `packages/backend/src/scoring/`  
**核心函式**: `scoreRequest()` in `scoring/index.ts:155`

這是整個系統最核心的業務邏輯。23 維度評分 < 2ms。

### 評分架構

```
scoreRequest(input, config?, momentum?)
├── 快速路徑：空訊息 → standard/ambiguous
├── 快速路徑：短訊息(< 50字) + 無複雜 keyword → simple
├── checkFormalLogicOverride → 若命中 → reasoning (直接跳過評分)
│
├── KeywordTrie.scan(combinedText)  — Aho-Corasick 多模式字串搜尋
│   ├── 13 keyword 維度（見下表）
│   └── 返回 TrieMatch[]（dimension, keyword, position）
│
├── scoreDimensions() — 計算各維度分數
│   ├── keyword 維度：scoreKeywordDimension() — 命中次數正規化
│   └── structural 維度：STRUCTURAL_SCORERS map（Strategy Pattern）
│       ├── tokenCount: 估計 token 數量
│       ├── nestedListDepth: 最大 list 嵌套深度
│       ├── conditionalLogic: if/else/condition 關鍵字數量
│       ├── codeToProse: code block 占比
│       ├── constraintDensity: "must/should/only" 等限制詞密度
│       ├── expectedOutputLength: max_tokens 參數
│       ├── repetitionRequests: "for each/every/all" 等
│       ├── toolCount: tools 數組長度
│       └── conversationDepth: 訊息輪數
│
├── applyMomentum(rawScore, msgLen, momentum)
│   └── 根據最近 5 個 tier 的趨勢調整分數（防止短跟進訊息降級）
│
├── applyTierFloors(effectiveScore, ...)
│   ├── hasTools → max(tier, standard)
│   ├── totalTokens > 50K → max(tier, complex)
│   └── momentumApplied → reason = 'momentum'
│
└── computeConfidence(score, boundaries, confidenceK)
    └── sigmoid 函式：confidence < threshold(0.45) → tier = standard/ambiguous
```

### 23 維度配置（config.ts）

**Keyword 維度（13個）**：
| 維度 | 權重 | 方向 | 說明 |
|------|------|------|------|
| formalLogic | 0.07 | up | 邏輯推論、數學證明 |
| analyticalReasoning | 0.06 | up | 分析推理 |
| codeGeneration | 0.06 | up | 程式碼生成 |
| codeReview | 0.05 | up | 程式碼審查 |
| technicalTerms | 0.07 | up | 技術術語 |
| simpleIndicators | 0.08 | **down** | 問候語、簡單問題（降分） |
| multiStep | 0.07 | up | 多步驟任務 |
| creative | 0.03 | up | 創意寫作 |
| questionComplexity | 0.03 | up | 問題複雜度 |
| imperativeVerbs | 0.02 | up | 命令動詞 |
| outputFormat | 0.02 | up | 格式要求 |
| domainSpecificity | 0.05 | up | 領域專業性 |
| agenticTasks | 0.03 | up | 代理任務 |
| relay | 0.02 | **down** | 轉發/重複訊息（降分） |
| webBrowsing~trading | 0 | up | Specificity 類別（不影響 complexity） |

**Structural 維度（10個）**：
| 維度 | 權重 | 說明 |
|------|------|------|
| tokenCount | 0.05 | 估計 token 數量 |
| nestedListDepth | 0.03 | List 嵌套深度 |
| conditionalLogic | 0.03 | 條件邏輯密度 |
| codeToProse | 0.02 | Code block 占比 |
| constraintDensity | 0.03 | 限制詞密度 |
| expectedOutputLength | 0.04 | max_tokens 參數 |
| repetitionRequests | 0.02 | 重複/批次請求 |
| toolCount | 0.04 | 工具數量 |
| conversationDepth | 0.03 | 對話深度 |

**Tier 邊界**（boundaries）：
```
score ≤ -0.1         → simple
-0.1 < score ≤ 0.08  → standard
0.08 < score ≤ 0.35  → complex
score > 0.35         → reasoning
```

### KeywordTrie（Aho-Corasick 實作）

位置：`scoring/keyword-trie.ts`

- 建構一次（lazy singleton），避免每次請求重建 Trie
- O(n+k) 掃描，n = 文字長度，k = 命中數
- 大小寫不敏感，詞邊界感知

---

## 2. Specificity 路由

**位置**: `packages/backend/src/scoring/specificity-detector.ts`  
**觸發時機**: resolve.service.ts 中，優先於 complexity scoring

9 個任務類別：
`coding` / `web_browsing` / `data_analysis` / `image_generation` /
`video_generation` / `social_media` / `email_management` / `calendar_management` / `trading`

偵測策略：
1. 掃描 messages（最後一條 user 訊息）中的 keyword
2. 掃描 tool names（前綴匹配，如 "browser_" → web_browsing）
3. `x-manifest-specificity` header override（agent 可強制指定）

---

## 3. Fallback 機制

**位置**: `routing/proxy/proxy-fallback.service.ts`

```
tryFallbacks(agentId, userId, fallbackModels, body, stream, ...)
├── for each fallback model in order:
│   ├── 解析 provider（同 resolve 策略）
│   ├── 取得 API key（可能是不同 provider）
│   ├── tryForwardToProvider()
│   └── 成功 → return { success, failures[] }
└── 全部失敗 → return { failures[] }
```

觸發條件（`fallback-status-codes.ts`）：
- 429 Too Many Requests
- 500, 503 Server Error
- 不包含 400 Bad Request（視為 client 錯誤，不 fallback）

---

## 4. Session Momentum

**位置**: `routing/proxy/session-momentum.service.ts`

- 記憶體 Map：`sessionKey → Tier[]`（最近 5 個）
- 30 分鐘 TTL
- 計算邏輯：近期 tier 的加權平均，越新越重要
- 效果：跟進訊息（"yes", "do it"）不會被降至 simple

---

## 5. Provider Key 管理

**位置**: `routing/routing-core/provider-key.service.ts`

- API keys 以 AES-256-GCM 加密儲存於 `user_providers.api_key_encrypted`
- `key_prefix` 欄位用於快速查詢（不需解密全部 key）
- 解密只在實際使用時進行
- Subscription provider（ChatGPT/MiniMax）透過 OAuth 取得 access token

---

## 6. 模型定價快取

**位置**: `model-prices/`

- `PricingSyncService`：啟動時 + 每日從 OpenRouter `/models` 端點同步
- `ModelPricingCacheService`：in-memory Map，key = model name
- 不寫資料庫，純記憶體快取
- 用於：cost 計算、tier auto-assign 的 quality score
- `quality-score.util.ts`：根據 price/quality 比率排序模型，分配到 tier

---

## 7. 多租戶資料隔離

**位置**: `analytics/services/query-helpers.ts:42`

`addTenantFilter(qb, userId, agentName?, tenantId?)`:
- 所有分析查詢都必須透過此函式加 WHERE 條件
- 使用 `user_id` 或 `tenant_id` 過濾（雙重保護）
- TypeORM QueryBuilder API（非 raw SQL）

---

## 8. Agent Message Dedup

**位置**: `routing/proxy/proxy-message-dedup.ts`

問題：Streaming 請求可能收到兩次 usage 資料（OTLP + streaming usage chunk）。
解決：以 (tenantId, agentId, traceId, model, sessionKey) 為 key，使用 write lock 避免重複 INSERT。
