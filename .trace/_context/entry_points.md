# Stage 2.1: Entry Points

## 主要啟動點

### `packages/backend/src/main.ts` — 伺服器啟動

```
bootstrap() 函式啟動流程：
1. NestFactory.create(AppModule, { bodyParser: false })
   - 關閉 NestJS 預設 body parser（Better Auth 需要原始 body）
2. app.enableShutdownHooks()
3. app.useGlobalFilters(SpaFallbackFilter)  — SPA fallback（生產用）
4. app.use(helmet(...))  — 嚴格 CSP（self-only，禁止 CDN）
5. app.use(compression())
6. 開發模式：app.enableCors(...)
7. app.useGlobalPipes(ValidationPipe({ transform, whitelist }))
8. expressApp.use(httpErrorLogger)
9. 生產模式：trust proxy 1
10. 登入限流：express-rate-limit at '/api/auth/sign-in' (20次/15分鐘)
11. Better Auth mount：expressApp.all('/api/auth/*splat', toNodeHandler(auth))
    ⚠️ 在 express.json() 之前 mount，因為 Better Auth 需要控制 body parsing
12. 重新加回 body parser：express.json({ limit: '1mb' })
13. app.listen(port, host)
```

環境變數控制啟動：
- `MANIFEST_EMBEDDED=true` → 跳過 bootstrap（嵌入式模式）
- `PORT` → 監聽 port（預設 3001）
- `BIND_ADDRESS` → 監聽位址（預設 127.0.0.1）

### `packages/backend/src/app.module.ts` — Root Module

全域 Guard 載入順序（order matters！）：
1. **SessionGuard** — Better Auth cookie session 驗證，先確認 `@Public()` 裝飾器
2. **ApiKeyGuard** — X-API-Key header 驗證（已有 session 則跳過）
3. **ThrottlerGuard** — 速率限制（100 req/60s）

模組組成：
- ConfigModule（全域）
- CacheModule（全域，TTL: DASHBOARD_CACHE_TTL_MS）
- ServeStaticModule（如有 frontend dist 目錄）
- ThrottlerModule
- CommonModule、DatabaseModule、AuthModule
- HealthModule、AnalyticsModule、OtlpModule
- ModelPricesModule、NotificationsModule、RoutingModule
- SseModule、GithubModule、PublicStatsModule、SetupModule、FreeModelsModule

### `packages/frontend/src/App.tsx` — 前端入口

SolidJS Router 設定：
- `AuthLayout` 包裹所有路由
- `AuthGuard` / `GuestGuard` 控制存取
- 路由：`/login`, `/register`, `/reset-password`, `/workspace`, `/agents/:name/*`, `/account`, `/model-prices`, `/help`, `/setup`

---

## Module 初始化

### DatabaseModule 初始化

1. TypeORM 連接 PostgreSQL（DATABASE_URL）
2. 開發/測試環境或 `AUTO_MIGRATE=true` → 自動執行 migrations
3. DatabaseSeederService → 若 `SEED_DATA=true` 則植入 demo 資料

### Auth 初始化（在 module 載入之前）

`auth/auth.instance.ts` 在 TypeScript import time 執行（早於 NestJS ConfigModule）：
- 建立 pg.Pool 連線
- 初始化 betterAuth（email/password + Google/GitHub/Discord OAuth）
- OAuth providers 只在設定 CLIENT_ID + CLIENT_SECRET 時啟用
- 電子郵件驗證只在設定 email provider 時啟用

### 快取初始化

PricingSyncService 在 startup 時從 OpenRouter API 拉取定價資料：
- 每日透過 cron job 更新
- 儲存在記憶體 Map 中（不寫資料庫）

---

## 程式入口快速對照

| 請求類型 | Entry Point |
|---------|------------|
| 用戶登入/OAuth | `POST /api/auth/sign-in` → Better Auth handler（在 NestJS 外） |
| Dashboard API | NestJS controller → SessionGuard/ApiKeyGuard |
| LLM Proxy | `POST /v1/chat/completions` → ProxyController → AgentKeyAuthGuard |
| 健康檢查 | `GET /api/v1/health` → HealthController（@Public()） |
| 前端靜態檔案 | ServeStaticModule 服務（生產模式） |
