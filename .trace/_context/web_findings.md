# Stage 1: Web Search 發現

## 搜尋結果摘要

### 1. 官方資源

| 資源 | URL | 關鍵資訊 |
|------|-----|---------|
| 官方文件 | https://manifest.build/docs/introduction | 包含 scoring 演算法說明、tier 定義 |
| GitHub 倉庫 | https://github.com/mnfst/manifest | MIT 授權，Beta 狀態 |
| Docker Hub | https://hub.docker.com/r/manifestdotbuild/manifest | 主要發布管道 |
| Mintlify 文件備份 | https://mnfst-manifest.mintlify.app/ | 文件舊版本 |

### 2. 架構關鍵發現

來源：[GitHub README](https://github.com/mnfst/manifest)

- **23 維度評分演算法**：13 keyword 維度 + 10 structural 維度，< 2ms 完成
- **四 Tier 體系**：Simple / Standard / Complex / Reasoning
- **Scoring 細節**（來源：manifest.build/docs）：
  - 短訊息（< 50 字）且無複雜 keyword → 直接判為 simple，跳過完整評分
  - Session momentum：追蹤最近 5 個 tier（30分鐘窗口），短跟進訊息沿用前一 tier
  - Tier floor 規則：detected tools → standard floor，context > 50K tokens → complex floor，formal logic keywords → reasoning（直接跳過 scoring）

### 3. 部署模式

來源：[manifest.build](https://manifest.build/)

- **Self-hosted**：完全本地，agent → 本地 Manifest 容器 → LLM provider（Manifest server 完全不參與）
- **Cloud**：agent → app.manifest.build → LLM provider（Manifest 只儲存 metadata，不儲存 message 內容）

### 4. 近期架構變動

來源：[GitHub Discussions](https://github.com/mnfst/manifest/discussions)

- **移除獨立 OTLP telemetry**：所有觀測數據現在只透過 routing proxy 收集，強制使用 `manifest/auto` 作為 model name
- **簡化 onboarding**：從多步驟改為 2 步驟（雲端與本地一致）
- **模組重構**：routing module 拆分為更小的 sub-modules（SRP 原則）
- **新增 providers**：Z.ai、MiniMax、OpenRouter free models（Discussion #951）

### 5. 社群頻道

| 管道 | URL |
|------|-----|
| Discord | https://discord.gg/FepAked3W7 |
| GitHub Discussions | https://github.com/mnfst/manifest/discussions |
| GitHub Issues | https://github.com/mnfst/manifest/issues |

### 6. 相關深度文章

- [DEV Community: Why Your OpenClaw Agent Gets Slower and More Expensive](https://dev.to/studio1hq/why-your-openclaw-agent-gets-slower-and-more-expensive-over-time-5c5e)
  - 說明為什麼 conversation depth 增加時成本上升，以及 Manifest 如何透過 session momentum 控制這個問題
  
- [Manifest Review](https://devhub.best/blog/manifest-review)
  - 第三方評測，確認與 OpenRouter 的核心差異：本地路由 vs 雲端代理

### 7. 版本歷史

來源：[GitHub Releases](https://github.com/mnfst/manifest/releases)  
最新已知版本：5.25.3（截至 2026-04）

---

## 未找到的資訊

以下項目未能從 web search 找到，需從程式碼直接 trace：
- 詳細的 migration 歷史說明
- 內部 scoring config 的 calibration 依據
- Provider adapter 實作細節（Anthropic / Google 轉換邏輯）
