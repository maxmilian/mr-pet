# Mr.Pet × PetRisk.AI 系統建置提案書

> **提案方**：Max Hsu（技術架構 & 全端開發）  
> **客戶**：Mr.Pet × PetRisk.AI 團隊  
> **日期**：2026-05-09（v1.2 更新於 2026-05-15，補入客戶 BP 投影片資訊）  
> **版本**：MVP 提案（WordPress 混合架構）  
> **會議紀錄**：參見 [2026-05-09 PetAI 會議紀錄](../meeting-notes/2026-05-09-petai.md)  
> **客戶 BP 對位**：參見 [2026-05-15 BP 投影片資訊整理](../meeting-notes/2026-05-15-petrisk-bp-slides-summary.md)

---

## 一、專案概述

Mr.Pet × PetRisk.AI 是一個**人寵健康 AI 生態系**，分為兩大子系統：

| 系統 | 定位 | 目標用戶 | 網域 | 技術選型 |
|------|------|----------|------|---------|
| **Mr.Pet** | 電商平台 + 飼主端 | C 端飼主 | `mrpet.tw` | WordPress + WooCommerce（Linode）|
| **PetRisk.AI** | 獸醫端 SaaS + AI 輔助診斷 | B 端獸醫診所 | `vet.mrpet.tw` | 自建 React + FastAPI（GCP）|

採**混合架構**：電商端用 WordPress + WooCommerce 大幅縮短上線時程；獸醫端因醫療合規需求自建。兩系統透過 **Clerk SSO 中央身份層**與 **PetPoint 中央點數引擎**連動，醫療資料完全隔離於 WordPress 之外。

---

## 一之二、對位客戶 BP 願景

> 客戶 BP 描繪 PetRisk.ai 為「多模態醫療 AI 平台」，本提案處理的是 **MVP 落地階段**。
> 本節說明兩者對位關係，避免投資人簡報 ↔ 工程交付脫節。

### A. PetRisk.ai 核心技術架構五層（BP 揭露）

| 層級 | BP 名稱 | MVP 提案對應 |
|---|---|---|
| L5 | 臨床流程整合 Clinical Workflow Layer（PMS 嵌合、健檢/慢病追蹤） | **Phase 2+**（MVP 不做 PMS 整合，先做獨立 Dashboard） |
| L4 | 預測與建議層 Prediction & Recommendation（風險分數、Next-best-Actions） | §3.2.4 AI Second Opinion（文字診斷建議 + 機率）|
| L3 | 多模態核心引擎 Multimodal Engine（Mixture of Models、Clinical Fusion） | §2.4.3 Vertex AI Gemini Multimodal（MVP 簡化版） |
| L2 | 臨床特徵工程 Clinical Feature Layer（Breed-adjusted Z-Scores、時序、語意嵌入） | **Phase 2+**（需 9 犬 3 貓資料齊備才能跑 Z-Score）|
| L1 | 資料擷取層 Data Acquisition（PetRisk Canonical Schema、First Data Moat） | §3.2.3 影像上傳（MVP 僅 JPEG/PNG，DICOM 列 Phase 2） |

**MVP 取捨**：先把 L4（預測）+ L3（多模態）的最小可用切出來，L1/L2/L5 用簡化版頂著（單一上傳介面 / 不做 Z-Score / 不接 PMS）。

### B. BP 技術棧 vs MVP 提案技術棧

| 角色 | BP 揭露 | MVP 提案 | 差異說明 |
|---|---|---|---|
| Data Ingestion | FastAPI + GCP IAM、DICOM | FastAPI + GCS + IAM | ✅ 對齊（DICOM 延後）|
| Storage | GCS（影像）+ BigQuery（結構化） | GCS（影像）+ Cloud SQL PostgreSQL（結構化） | ⚠️ MVP 用 PostgreSQL，BQ 留給未來分析 |
| Annotation | CVAT | — | MVP 不做大規模標註，沿用台大已標註資料 |
| Pre-process | OpenCV / MONAI / Pydicom / wfdb | — | Phase 2+（DICOM/ECG 進來才需要） |
| Feature Eng. | Python / Pandas / Scikit-learn | — | Phase 2+（Z-Score 階段才導入）|
| Model Training | PyTorch / T-Flow / MONAI / Vertex AI | **直接用 Vertex AI Gemini（不自訓模型）** | ⚠️ MVP **不自訓模型**，用 Gemini Multimodal API + RAG 補臨床知識 |
| Knowledge Engine | LLM + KnowledgeGraph（Neo4j / LangChain） | LlamaIndex + PGVector（向量檢索） | ⚠️ MVP 不導入 Knowledge Graph，僅做 RAG |
| Vet Dashboard | React + Web | React + Refine + Tailwind | ✅ 對齊 |
| Pet Owner App | Flutter for Mobile | **WordPress 站台 + 後續 PWA** | ⚠️ MVP 不做原生 App，Flutter 列 Phase 2+ |

> **核心訊息給客戶**：MVP 不在「自建多模態模型」這條路上燒時間，而是用 Vertex AI Gemini 把 **L3+L4 的「對外可用體驗」**先做出來。L1（DICOM/CVAT）、L2（Z-Score）、自訓 PyTorch/MONAI 模型，等資料量、標註品質、臨床驗證都到位後再做（建議 Phase 3+，2026 Q4 起）。

### C. MVP 內容範圍（與 BP 口頭確認對齊）

- **5 個專科模組**（具體哪 5 個待客戶確認，從 BP 中 ECG/ASCVD 示意推測心臟科為其一）
- **9 個犬種 + 3 個貓種** = 12 個 breed-adjusted 模型對應
- **5 × 12 = 60 組 sub-model 配對**

> **可行性提醒**：60 組配對在 MVP 8 月底前不可能全部做到「臨床可信」品質。建議第一階段聚焦 **1 專科 × 3 犬種** 做 vertical slice 驗證 RAG + Gemini pipeline，其餘留為 Phase 2 滾動式擴充。此調整需與客戶於 5 月底前確認。

### D. 雙產品定價（與 BP 對齊）

| 產品 | 對象 | BP 揭露定價 | 本提案系統需支援 |
|---|---|---|---|
| **PetRisk.ai** | 合作獸醫端 B2B | 月繳 **39 / 3,000 元**（兩階）| 訂閱方案管理、用量計費（Phase 2） |
| **Mr.Pet** | 飼主端 D2C | 月繳 **598 / 1,598 / 2,989 元**（三階訂閱）| WooCommerce 訂閱 plugin、訂閱盒物流 |

> **MVP 8 月版本**：先支援單一付費模式（PetRisk 用單一價、Mr.Pet 走一次性購物），訂閱方案管理列入 Phase 2。

### E. 命名差異提醒：PetPoint ↔ VetPoints Rewards

- 本提案內部稱「PetPoint 中央點數引擎」
- BP 對外稱「**VetPoints Rewards**」（連接 Mr.Pet ↔ PetRisk.ai 雙邊飛輪的核心機制）
- **建議**：技術內部沿用 PetPoint Engine，對外品牌 UI 一律使用 VetPoints；engine 同一套不需改名

### F. 雙邊商業模式飛輪（BP 揭露，影響系統設計）

BP 明確定義雙邊價值：

- **飼主端**：世界級獸醫建議、更好醫療品質、人寵安全感、老齡醫療險機會、精喜訂閱盒
- **獸醫端**：客源/客單增長、醫療品質提升、溝通效率提升、不損獲利%、資料去識別化

> **系統設計影響**：
> 1. PetPoint/VetPoints 須支援**雙向兌換**（飼主消費累點 / 獸醫端折抵 / 獸醫端推薦回饋）— 本提案 §3.3.3 已涵蓋
> 2. 「老齡醫療險機會」暗示需要 export risk score 給保險合作方 → Phase 2+ API
> 3. 「資料去識別化」承諾 → 本提案 §3.3.2 Token ID 已完整覆蓋
> 4. 「精喜訂閱盒」→ WooCommerce 需加 Subscriptions plugin（Phase 2）

### G. 競爭定位（BP 揭露）

BP 競爭分析象限把 Mr.Pet × PetRisk.AI 放在「**多領域健康管理 + 獸醫療輔助判斷**」獨佔象限，對標：

- 單領域 + 醫療輔助：SignalPET、Zoetis VETSCAN、NxVET
- 單領域 + 消費娛樂：ruff.box、BarkBox、Chewy Goody Box

> **系統設計影響**：「多領域」承諾要求 Dashboard 從 Day 1 就用 **可擴充的專科模組架構**（plugin-style），不要寫死成單一影像分析頁。本提案 §3.2 已預留多模組路徑，但 MVP UI 需明確呈現「專科 selector」入口讓客戶 demo 有故事可說。

### H. Continuous Learning 資料來源（BP 揭露）

BP 列出 5 大期刊資料庫 + 台大獸醫所專家標註：

- PubMed / MEDLINE
- CABI
- Scopus
- IVIS
- Wiley Online Library
- AVMA Journals
- 台大獸醫所專家篩選 / 標註 / 賦能

> **影響本提案 §3.2.4 RAG 設計**：論文 ingestion pipeline 需支援多來源格式（PDF/XML/HTML），不能只認葉老師手上那批。建議 MVP 階段先 ingest 10~20 篇核心論文驗證 pipeline，正式上線後分批導入。

---

## 二、系統架構總覽

參見：[architecture-proposal.html](../architecture-proposal.html)

### 2.1 架構原則

1. **混合部署**：電商用成熟 OSS（WP），獸醫端自建以滿足醫療合規
2. **資料邊界**：WordPress 僅管電商會員 + 訂單 + 點數觸發；醫療資料**絕不**進 WP DB
3. **SSO 中央化**：Clerk 為 IdP，WordPress 與 PetRisk.AI 都當 SP
4. **PetPoint 中央化**：點數帳本獨立服務，不綁在 WC plugin 上
5. **去識別化**：獸醫端以 Token ID 處理病患，無法反查飼主真實身分
6. **多租戶隔離**：每間獸醫診所獨立 Workspace（PostgreSQL Row-Level Security）

### 2.2 技術棧

| 層級 | Mr.Pet (C端) | PetRisk.AI (B端) |
|------|-------------|------------------|
| **前端** | WordPress 主題（行動優先）+ WooCommerce | React + Tailwind（Refine 框架，desktop） |
| **後端** | PHP 8.x（WordPress / WooCommerce 核心） | Python FastAPI |
| **資料庫** | MySQL 8（WP 內建）+ Redis Object Cache | Cloud SQL PostgreSQL + PGVector |
| **AI** | — | Vertex AI（Gemini Multimodal）+ LlamaIndex（RAG） |
| **檔案儲存** | WP 媒體庫（商品圖） | Google Cloud Storage（醫療影像，加密） |
| **身份層** | Clerk（中央 IdP，LINE/Google/Apple） | 同左 |
| **點數引擎** | PetPoint Engine（自建，跨系統共用） | 同左 |
| **金流** | 綠界 NewebPay plugin + 綠界發票 plugin | — |
| **CDN / WAF** | Cloudflare（Free plan） | Cloud Load Balancer |
| **部署** | Linode（既有主機） | GCP Cloud Run |

### 2.3 關鍵 Plugin 清單（WordPress 端）

| Plugin | 用途 | 授權 |
|--------|------|------|
| WooCommerce | 電商核心 | GPL（免費） |
| ECPay for WooCommerce | 綠界金流（信用卡 / 超商 / ATM） | 免費 |
| 綠界電子發票 for WC | B2C 發票自動上傳財政部 | 免費 |
| miniOrange OAuth Client | 接 Clerk OIDC（WP 當 SP） | $99/年 |
| Wordfence Security | WAF + 漏洞掃描 + 自動更新 | 免費版即可 |
| LiteSpeed Cache（或 W3 Total Cache） | 頁面 + Redis Object Cache | 免費 |
| Advanced Custom Fields (ACF) | 寵物檔案自訂欄位 | 免費 |
| **自寫 mu-plugin：PetPoint Bridge** | 訂單完成 → POST 到 PetPoint Engine | 自製 |

> 控制 plugin 數量在 10 個內，每次升級先在 staging 站驗證，避免衝突。

### 2.4 技術選型理由與替代方案比較

#### 2.4.1 電商：WordPress + WooCommerce vs. 自建（Next.js + FastAPI）vs. SaaS（Shopify）

| 維度 | WP + WooCommerce ✅ | 自建 Next.js + FastAPI | Shopify |
|------|---------------------|-----------------------|---------|
| 開發工時 | **2 週**（主題客製 + plugin 設定） | 6~8 週 | 1 週但客製受限 |
| 金流（綠界） | 官方 plugin 免費 | 需自寫整合（2~3 週） | 台灣綠界 plugin 不穩定 |
| 電子發票 | 綠界 plugin 一鍵串接 | 需自寫 + 財政部 API 認證 | 須走第三方 app |
| 商品上架 | 後台 GUI，客戶可自助 | 需自製 CMS | 後台 GUI |
| 客製彈性 | 中（plugin + 主題） | 高 | 低 |
| 月費 | NT$ 600 主機 | NT$ 1,200 | USD $39+ + 交易抽成 |
| 資安 | Cloudflare WAF + Wordfence 可控 | 自建較難維持 | 平台託管 |
| **結論** | **MVP 最佳選擇** | 8 月時程做不完 | 不適合台灣金流 / 發票 |

> **為什麼選 WP**：MVP 8 月底上線時程吃緊，電商功能 80% 可用 OSS 滿足，把工時集中在獸醫端 AI 與合規才是價值點。

#### 2.4.2 獸醫端：自建 React + FastAPI vs. 低代碼平台 vs. WP 子站

| 維度 | React + Refine + FastAPI ✅ | Retool / Budibase | WP 子站 |
|------|---------------------------|-------------------|---------|
| 醫療合規（資料隔離） | 完全可控 | 平台限制 | 同一 DB，違反隔離原則 |
| 多租戶 RLS | PostgreSQL Row-Level Security | 平台不支援 | WP 多站需企業版 |
| AI 影像 viewer | Cornerstone.js OSS 可整合 | 不支援 | 不支援 |
| 部署 | GCP Cloud Run 獨立 instance | Vendor lock | 與電商共用主機 |
| 開發工時 | 2 週 Dashboard + 2 週 AI 整合 | 1 週但功能受限 | 1.5 週但合規不過 |
| **結論** | **唯一可行方案** | 無法滿足醫療合規 | 違反資料隔離原則 |

#### 2.4.3 AI 引擎：Vertex AI Gemini vs. OpenAI vs. 自架 LLaVA

| 維度 | Vertex AI Gemini ✅ | OpenAI GPT-4o | 自架 LLaVA / MedGemma |
|------|--------------------|--------------- |----------------------|
| 多模態（影像） | 原生 Multimodal | 原生 Multimodal | 需自架 GPU |
| 資料主權 | GCP 台灣 region | 美國，傳輸出境疑慮 | 完全在地 |
| 推論成本（每次） | NT$ 3~6 | NT$ 8~15 | GPU 月費 NT$ 30,000+ |
| RAG 整合 | LlamaIndex + PGVector | LlamaIndex + Pinecone | 需自架向量 DB |
| 醫療合規友善度 | GCP HIPAA 合規 | 醫療資料條款限制多 | 自架最寬鬆但維運重 |
| MVP 適配 | **最佳** | 成本偏高 | 過重 |

> **為什麼選 Vertex AI**：與 GCP 其他服務（Cloud SQL、GCS、IAM）整合最順、成本可控、台灣 region 滿足個資法資料主權要求。MVP 階段使用量 < 200 次/月，月費 NT$ 600~1,500，遠低於自架 GPU。

#### 2.4.4 SSO：Clerk vs. Auth0 vs. Keycloak vs. 自建

| 維度 | Clerk ✅ | Auth0 | Keycloak | 自建 OAuth |
|------|---------|-------|----------|-----------|
| LINE Login 支援 | ✅ 內建 | ✅（需設定） | 需自寫 plugin | 需自寫 |
| 開發工時 | 0.5 週 | 0.5 週 | 1.5 週（自架）| 2 週 |
| MAU 10K 內成本 | **免費** | USD $240/月起 | 自架 NT$ 1,500/月 | 自架 NT$ 1,500/月 |
| 維運負擔 | 零（SaaS） | 零 | 高（自架） | 高 |
| 醫療合規 | SOC 2 Type II | SOC 2 + HIPAA | 自負 | 自負 |
| **結論** | **MVP 最佳** | 規模大才划算 | 過度工程 | MVP 不應自建 |

#### 2.4.5 主機：Linode vs. GCP vs. AWS

| 場景 | 選擇 | 理由 |
|------|------|------|
| WordPress 電商 | **Linode**（既有） | 既有主機可重用、月費低、台灣 latency 可接受、WP 生態成熟 |
| 獸醫端 + AI + 醫療 DB | **GCP** | Vertex AI 原生、Cloud SQL HA、IAM 細緻、台灣 region、HIPAA 合規 |
| 為何不全 GCP？ | 成本考量 | GCE 跑 WP 月費約 NT$ 1,200~2,000，Linode NT$ 600 即可 |
| 為何不全 Linode？ | 缺 AI / 受管 PostgreSQL | Linode 無 Vertex AI 級服務，自架 LLM 與 HA DB 維運重 |

---

### 2.5 技術可行性評估

#### 2.5.1 模組級可行性與成熟度評分

| 模組 | 技術成熟度 | 團隊熟悉度 | 風險等級 | 預期工時 | 工時 buffer |
|------|----------|-----------|---------|---------|-----------|
| WordPress + WooCommerce 商城 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 低 | 2 週 | +0.5 週 |
| 綠界金流 + 發票 plugin | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 低 | 0.5 週 | +0.5 週（測試對帳） |
| Clerk SSO 整合 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 低 | 0.5 週 | 0 |
| PetPoint Engine（自建） | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 中 | 1.5 週 | +0.5 週 |
| WC ↔ PetPoint Bridge | ⭐⭐⭐ | ⭐⭐⭐ | 中 | 0.5 週 | +0.5 週 |
| PetRisk.AI Dashboard | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 低 | 2 週 | +0.5 週 |
| Vertex AI + RAG | ⭐⭐⭐⭐ | ⭐⭐⭐ | **高** | 1.5 週 | **+1.5 週** |
| 多租戶 + Token ID | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 中 | 1.5 週 | +0.5 週 |
| 醫療影像 viewer（Cornerstone.js） | ⭐⭐⭐⭐ | ⭐⭐ | 中 | 含於 Dashboard | +0.5 週 |
| 安全合規 + 滲透測試 | ⭐⭐⭐⭐ | ⭐⭐⭐ | 中 | 1 週 | +0.5 週 |

> 成熟度：⭐⭐⭐⭐⭐ 業界標準 / ⭐⭐⭐⭐ 廣泛使用 / ⭐⭐⭐ 可行但需驗證 / ⭐⭐ 新興技術

#### 2.5.2 高風險項目深度評估

**[A] Vertex AI + 期刊 RAG（最高風險）**

- **風險來源**：
  1. 葉老師期刊論文標註下週才啟動，AI 訓練資料時程外部相依
  2. Gemini Multimodal 對 X 光影像的醫療領域準確度未驗證
  3. RAG 引用準確度需獸醫師人工標註校驗
- **可行性結論**：**技術可行、時程有風險**
- **緩解策略**：
  - Phase 3 拆兩階段：先建 demo 模式（無 RAG，只回 Gemini 原始輸出）→ 期刊資料齊備後切換正式 RAG
  - 與葉老師約定每週同步論文進度，若 7 月底資料未齊，8 月先以 demo 上線、9 月正式啟用
  - PoC 階段（5/底前）先用 10 篇論文做 vertical slice 驗證 RAG pipeline

**[B] PetPoint 跨系統一致性（中高風險）**

- **風險來源**：WP webhook 失敗 / 重送導致重複累點或扣點
- **可行性結論**：**技術可行**
- **緩解策略**：
  - PetPoint API 強制 idempotency key（WC order_id 或 vet transaction_id 為 key）
  - 失敗補償：mu-plugin 實作 retry queue + 本地 audit log
  - 對帳機制：每日 cron 比對 WC 訂單與 PetPoint 帳本，差異告警

**[C] 醫療資料合規（中風險，責任高）**

- **風險來源**：個資法、特種個資（醫療）保護要求高，違規責任重
- **可行性結論**：**技術可行**
- **緩解策略**：
  - Token ID 一律單向 hash + Mapping Service 隔離部署
  - WP DB 與醫療 DB 完全分離（不同 GCP project，不同 IAM）
  - 法律條款由律師審閱（不在本提案範圍）
  - 季度滲透測試 + 每月 plugin CVE 掃描

**[D] WordPress 安全性（中風險）**

- **風險來源**：WP plugin 漏洞、暴力破解、SQL injection
- **可行性結論**：**可控**
- **緩解策略**：
  - Cloudflare WAF + Wordfence + plugin 白名單 ≤ 10 個
  - `/wp-admin` IP 白名單 + 強制 2FA
  - 自動更新 + 每月人工 CVE 檢視
  - WP DB 帳號最小權限（無法跨庫查詢）

**[E] 獸醫公會 / 法規接受度（業務風險，非技術）**

- **風險來源**：獸醫師接受 AI 輔助診斷意願、公會規範
- **緩解策略**：林泰佑教授 + 雙北市獸醫公會 Demo（已 promised）

#### 2.5.3 效能與規模可行性

**MVP 階段（< 1500 MAU，目標同時 150 人）**

| 指標 | 目標 | 設計上限 | 證據 |
|------|------|---------|------|
| 同時在線（C 端） | 150 | 500+ | WP + Cloudflare CDN，靜態頁面 cache 命中 95% |
| AI 分析延遲 | < 30 秒 | 60 秒 | Vertex AI Gemini 平均 5~15 秒 + RAG 檢索 < 3 秒 |
| 影像上傳 | < 100MB | 500MB | GCS resumable upload |
| 並發 AI 請求 | 5/s | 20/s | Cloud Run autoscaling（min=0, max=10） |
| DB QPS | 100 | 1000 | Cloud SQL db-f1-micro 起步，可 scale up |

**1500 MAU 後的擴展路徑**
- WP：升等 Linode plan（NT$ 600 → 1,500），加開 Redis Object Cache
- PetRisk.AI：Cloud Run min instance 拉到 1，避免冷啟動
- Cloud SQL：升 db-g1-small，加 read replica
- Vertex AI：用量 1000+ 次/月時談 committed use 折扣

#### 2.5.4 整體可行性結論

| 維度 | 評估 |
|------|------|
| **技術可行性** | ✅ 全部模組技術成熟，無需重大研發突破 |
| **時程可行性** | ⚠️ 11 週緊湊，最大瓶頸為 Vertex AI + RAG（依賴外部資料） |
| **成本可行性** | ✅ MVP 月費 NT$ 3,150~5,550，遠低於自建全套方案 |
| **合規可行性** | ✅ 去識別化 + 資料隔離 + Cloud HIPAA 合規區可滿足 |
| **團隊能力** | ✅ 全端 + AI 整合在熟悉領域，WP 主題客製需 PHP 兼職補位 |
| **建議** | **可執行**，但 AI 模組必須拆 demo / 正式兩階段，降低 8 月上線風險 |

---

## 三、功能規格

### 3.1 Mr.Pet 電商平台（C端，WordPress + WooCommerce）

#### 3.1.1 商城功能（WC 內建）
- [x] 商品上架：寵物用品、食品、體驗券（後台 GUI 上架，客戶可自助）
- [x] 商品分類與搜尋
- [x] 購物車與結帳流程
- [x] 訂單管理與物流追蹤
- [x] 庫存管理（後台）

> 以上由 WooCommerce 直接提供，僅需主題客製化與基礎設定。

#### 3.1.2 會員系統（Clerk + WP）
- [ ] Clerk 提供 LINE / Google / Apple 登入
- [ ] WP 透過 OIDC plugin 接 Clerk（飼主一個 Clerk 帳號跨站）
- [ ] 寵物檔案（ACF 自訂欄位，可同步至 PetRisk.AI）

#### 3.1.3 PetPoint 紅利積分（自建中央服務）
- [ ] 訂單完成 webhook → PetPoint Engine 累點
- [ ] 點數查詢 API（飼主端 + 獸醫端共用）
- [ ] 獸醫端折抵 API（idempotency key 防重複扣帳）
- [ ] 點數到期提醒（cron + Email）

#### 3.1.4 金流整合（綠界 plugin）
- [ ] ECPay for WC：信用卡 / 超商 / ATM
- [ ] 綠界發票 plugin：B2C 發票自動上傳財政部
- [ ] 退款 / 作廢流程（WC 後台直接操作）

---

### 3.2 PetRisk.AI 獸醫端 SaaS（B端，自建）

#### 3.2.1 多租戶管理
- [ ] 診所註冊與帳號開通
- [ ] 獸醫師帳號管理（所內權限）
- [ ] 獨立 Workspace（PostgreSQL Row-Level Security）

#### 3.2.2 病患管理
- [ ] 病患登錄（透過 Token ID 去識別化）
- [ ] 飼主資料綁定（Clerk SSO）
- [ ] 就診歷程 Timeline
- [ ] 病歷資料夾管理

#### 3.2.3 醫療影像上傳
- [ ] MVP 支援格式：JPEG / PNG（DICOM 列入 Phase 2）
- [ ] MVP 影像類型：X 光（CT / 超音波列入 Phase 2）
- [ ] 上傳至 GCS 加密儲存（IAM 控管，僅 PetRisk 後端可讀）
- [ ] 影像預覽（Cornerstone.js 開源 viewer）

#### 3.2.4 AI Second Opinion
- [ ] 影像分析（Vertex AI Gemini Multimodal）
- [ ] 文字診斷建議與機率預測
- [ ] **可解釋性**：引用台大獸醫系期刊論文（PGVector RAG）
- [ ] 獸醫師 Approve / 修正機制
- [ ] Approve 後寫入病歷與飼主端 Timeline

#### 3.2.5 結帳與 PetPoint 折抵
- [ ] 診療費用輸入
- [ ] 即時查詢飼主 PetPoint 餘額（呼叫 PetPoint Engine）
- [ ] 點數折抵計算與扣帳（一次性 deduction）
- [ ] 現金 / 刷卡收款記錄（診所自行收款，平台不經手金流）
- [ ] 發票開立整合（Phase 2）

#### 3.2.6 報表與統計（Phase 2）
- 8 月 MVP 不做；以 Cloud Logging + BigQuery 直查代替

---

### 3.3 共享基礎設施

#### 3.3.1 Clerk SSO（中央 IdP）
- 支援 LINE Login、Google、Apple Sign-In
- JWT Token 跨系統傳遞（WP / PetRisk 都認）
- 單點登入 + 單點登出
- Free tier 10K MAU 內免費，超過後 $25/月起

#### 3.3.2 Token ID 去識別化服務
- 飼主真實身分 ↔ Token ID 一對一映射
- 醫療端僅能看到 Token ID，無法反查真實身分
- 獸醫間病歷共享時隱藏診斷醫師身分
- 飼主授權紀錄（誰看過自己的資料）

#### 3.3.3 PetPoint 引擎（中央點數帳本）
- 獨立 Cloud Run 服務 + PostgreSQL 帳本
- WP 訂單觸發累點（webhook in，自寫 mu-plugin）
- PetRisk 結帳折抵（REST API + idempotency key）
- 對外只暴露 REST API + JWT 驗證

#### 3.3.4 資料儲存架構

| 資料類型 | 儲存位置 | 隔離等級 |
|---------|---------|---------|
| 電商訂單 / 商品 / WP 會員 | Linode MySQL（WP 內建） | 標準商業資料 |
| PetPoint 帳本 | Cloud SQL PostgreSQL | 個資保護 |
| 醫療病歷（去識別化） | Cloud SQL PostgreSQL（獨立 instance） | 機敏醫療資料 |
| 醫療影像 | GCS（加密 + IAM） | 機敏醫療資料 |
| 期刊論文向量 | PGVector（同醫療 DB） | 知識庫 |
| Clerk 身份資料 | Clerk 託管 | 個資保護 |

> **關鍵**：WordPress 的 MySQL **絕不**儲存醫療資料，僅電商相關。即使 WP 被攻陷，醫療資料仍安全。

---

## 四、安全與合規設計

### 4.1 個資保護
- **去識別化**：所有醫療紀錄以 Token ID 儲存，真實身分獨立加密存放於 Clerk + Mapping DB
- **最小權限**：獸醫僅能看到自己診所的病患資料（PG RLS）
- **存取紀錄**：所有資料查詢保留 Audit Log

### 4.2 WordPress 端資安強化
- **Cloudflare WAF + Wordfence**：阻擋已知漏洞攻擊
- **plugin 白名單**：僅安裝 §2.3 列表中的 plugin
- **自動更新**：WP core / plugin 自動安裝安全更新
- **每月人工安檢**：檢查 plugin 漏洞公告（CVE）
- **資料庫權限分離**：WP DB 帳號僅能存取 wp_* 表，無法跨庫查詢
- **管理後台 IP 白名單**：`/wp-admin` 限制 IP 來源

### 4.3 法律文件
- [ ] 隱私條款（Privacy Policy）
- [ ] 使用者服務條款（Terms of Service）
- [ ] 免責聲明（Second Opinion 非正式診斷）
- [ ] 個資同意書（寵物醫療資料使用授權）

### 4.4 資安標準
- SSL/TLS 全站加密（Cloudflare 自動憑證）
- 資料庫加密靜態儲存
- API 速率限制 + Cloudflare WAF
- 季度漏洞掃描

---

## 五、MVP 範圍與里程碑

### 5.1 MVP 功能範圍（8 月底上線）

**Mr.Pet 電商 MVP**
- WordPress 商城上線（10 個 SKU 內）
- Clerk SSO（LINE / Google）
- 綠界金流 + 電子發票
- PetPoint 累點（消費 1 元 = 1 點）

**PetRisk.AI MVP**
- 單一診所帳號（暫不開放自行註冊）
- 病患上傳 X 光片（JPEG / PNG）
- Vertex AI Gemini 影像分析 + 期刊 RAG
- Second Opinion 文字報告（含論文引用）
- 獸醫師 Approve 機制
- PetPoint / VetPoints 折抵 API 串接
- **專科模組 selector UI**（對齊 BP 5 專科架構，MVP 先實作 1 專科 vertical slice，UI 預留其餘 4 個 placeholder）
- **品種選擇**：BP 規劃 9 犬 3 貓，MVP 先支援 3 犬種驗證 pipeline，其餘 9 種列 Phase 2

**Phase 2（9 月後）**
- DICOM 支援、CT / 超音波分析
- 多診所自助註冊
- 報表與營運統計
- iOS / Android App（PWA → 原生）

### 5.2 開發時程

| 階段 | 時間 | 交付項目 |
|------|------|---------|
| **Phase 1** | 5/中 ~ 6/初 | Linode WP + WC 基礎建置、Clerk 設定、綠界 plugin 串接 |
| **Phase 2** | 6/初 ~ 7/初 | PetPoint Engine、WP↔PetPoint Bridge、商品上架 SOP、SSO 跨站測試 |
| **Phase 3** | 7/初 ~ 8/初 | PetRisk.AI Dashboard、影像上傳、Vertex AI + RAG、Approve 流程 |
| **Phase 4** | 8/初 ~ 8/中 | 雙系統串接測試、安全強化、UAT、滲透測試 |
| **Go-Live** | 8 月底 | 正式上線、種子診所導入（目標 3~5 間） |

### 5.3 上線後目標

| 指標 | 8 月 | 9 月 | 10 月 | 12 月 |
|------|------|------|-------|-------|
| 合作診所數 | 5 | 15 | 30 | 50+ |
| 月活躍飼主 | 100 | 300 | 600 | 1500 |
| AI 分析次數 | 50 | 200 | 500 | 1000+ |

---

## 六、預算規劃

### 6.1 一次性開發成本

| 項目 | 說明 | 預估工時 |
|------|------|---------|
| 系統架構與環境建置 | Linode WP + GCP 環境 + CI/CD | 0.5 週 |
| WordPress + WC 商城建置 | 主題客製、商品上架 SOP、plugin 設定 | 2 週 |
| 綠界金流 + 發票 plugin 串接 | 串接、測試、對帳流程 | 0.5 週 |
| Clerk SSO 整合 | WP OIDC + PetRisk JWT 驗證 | 0.5 週 |
| PetPoint Engine | 累點、折抵、查詢 API、idempotency | 1.5 週 |
| WC ↔ PetPoint Bridge | mu-plugin + webhook | 0.5 週 |
| PetRisk.AI Dashboard | Refine 框架 + 病歷 + 影像上傳 | 2 週 |
| Vertex AI + RAG | LlamaIndex + 期刊 ingestion + Prompt 調校 | 1.5 週 |
| 多租戶 + 去識別化 + Token ID | PG RLS + Mapping Service | 1.5 週 |
| 安全 + 合規 + 文件 + 滲透測試 | WP 加固、隱私條款、UAT | 1 週 |

> **總開發工時**：約 **11 週**（5/中 ~ 8/初），保留 3-4 週 buffer 給 UAT 與改 bug  
> **建議配置**：1 位全端（Max）+ 1 位 PHP/WP 兼職（主題客製階段 2 週）

### 6.2 每月維運成本

| 項目 | 預估月費 | 說明 |
|------|---------|------|
| Linode（既有，可能升等） | NT$ 600~1,200 | WP + MySQL + Redis |
| GCP Cloud Run × 2 | NT$ 600~1,000 | PetRisk API + PetPoint Engine（min=0） |
| Cloud SQL PostgreSQL | NT$ 1,200~1,800 | 醫療 DB（獨立 instance）|
| Vertex AI 推論 | NT$ 300~800 | 50~200 次/月 |
| Cloud Storage | NT$ 200~500 | 醫療影像（< 100GB） |
| Clerk SSO | NT$ 0 | Free tier 10K MAU |
| Cloudflare CDN + WAF | NT$ 0 | Free plan |
| miniOrange OAuth plugin | NT$ 250 | 攤提 $99/年 |
| **合計** | **NT$ 3,150~5,550** | MVP 階段 |

> 1500 MAU 後 Clerk 升級 + Cloud Run 流量上升，月費約 NT$ 7,000~10,000

---

## 七、風險與因應

| 風險 | 影響 | 因應措施 |
|------|------|---------|
| 獸醫公會接受度 | 高 | 與台大動物醫院 POC，取得公會理事長認可 |
| AI 診斷準確度 | 中 | 期刊 RAG 支撐可解釋性，明確標示「輔助參考」 |
| **AI 模組時程**（葉老師論文標註下週才啟動） | 高 | 8 月先以 demo 模式上線，9 月正式啟用 |
| 8 月時程壓力 | 中 | WP 縮短電商工時，集中火力於 AI 與獸醫端 |
| 醫療個資法規 | 高 | 去識別化、隱私條款、免責聲明完備；醫療資料不入 WP DB |
| 點數法規（電子票證） | 中 | PetPoint 設計為「折扣點數」非儲值，規避電子票證規範 |
| **WordPress 安全性** | 中 | Cloudflare WAF + Wordfence + plugin 白名單 + 自動更新 + 每月安檢 |
| **WP plugin 衝突** | 中 | plugin 數量上限 10 個內，升級先在 staging 驗 |
| **PHP 不在原 stack** | 低 | 找 1 位 PHP 兼職（主題客製 2 週）；WP plugin 大多 GUI 操作不需深 PHP |

---

## 八、後續建議

1. **立即行動**：確認 v1.2 提案與預算上限，啟動 Linode WP 環境與 Clerk 帳號
2. **本週（與客戶釐清的 BP 對位事項）**：
   - 確認 MVP 5 專科具體為哪 5 個（推測心臟科為其一）
   - 確認 9 犬種 + 3 貓種具體品種清單
   - 確認 PetRisk.ai 39 元 vs 3000 元方案的服務內容差異
   - 確認 Mr.Pet 598 / 1598 / 2989 三階訂閱具體內容物（含訂閱盒）
   - 確認對外品牌沿用 BP 的「VetPoints Rewards」，內部仍用 PetPoint Engine
3. **本週**：完成 PetRisk.AI 操作流程 Wireframe（含專科 selector + 品種 selector）；採購 miniOrange OAuth plugin
4. **下週**：與葉老師會面，確認期刊論文格式與 RAG ingestion 流程；同步確認台大獸醫所標註資料規模與授權條件
5. **持續**：雙北市獸醫公會 Demo 安排與時程確認
6. **下階段（Phase 2+ 規劃）**：BP 五層架構中尚未涵蓋的 L1（DICOM/CVAT）、L2（Z-Score）、自訓模型（PyTorch/MONAI）需獨立提案估時，建議 2026 Q4 後啟動

---

## 九、聯絡資訊

| 項目 | 內容 |
|------|------|
| **提案人** | Max Hsu |
| **職稱** | 技術架構師 / 全端工程師 |
| **GitHub** | https://github.com/maxmilian/mr-pet |
| **備註** | 本提案採 WordPress 混合架構，根據可行性評估調整 |

---

*本文件為機密提案，僅供 Mr.Pet × PetRisk.AI 內部參考使用。*
