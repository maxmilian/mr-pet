# Mr.Pet × PetRisk.AI 系統建置提案書

> **提案方**：Max Hsu（技術架構 & 全端開發）  
> **客戶**：Mr.Pet × PetRisk.AI 團隊  
> **日期**：2026-05-09  
> **版本**：v1.1 MVP 提案（WordPress 混合架構）  
> **會議紀錄**：參見 [2026-05-09 PetAI 會議紀錄](../meeting-notes/2026-05-09-petai.md)

---

## 一、專案概述

Mr.Pet × PetRisk.AI 是一個**人寵健康 AI 生態系**，分為兩大子系統：

| 系統 | 定位 | 目標用戶 | 網域 | 技術選型 |
|------|------|----------|------|---------|
| **Mr.Pet** | 電商平台 + 飼主端 | C 端飼主 | `mrpet.tw` | WordPress + WooCommerce（Linode）|
| **PetRisk.AI** | 獸醫端 SaaS + AI 輔助診斷 | B 端獸醫診所 | `vet.mrpet.tw` | 自建 React + FastAPI（GCP）|

採**混合架構**：電商端用 WordPress + WooCommerce 大幅縮短上線時程；獸醫端因醫療合規需求自建。兩系統透過 **Clerk SSO 中央身份層**與 **PetPoint 中央點數引擎**連動，醫療資料完全隔離於 WordPress 之外。

### v1.1 與 v1.0 主要差異

| 項目 | v1.0（全自建） | v1.1（WordPress 混合）|
|------|--------------|---------------------|
| 電商平台 | Next.js + FastAPI 自建 | WordPress + WooCommerce |
| SSO | 自建 OAuth | Clerk 託管 IdP |
| 主機 | 全 GCP | Linode（電商）+ GCP（獸醫端） |
| 開發工時 | 16.5 週 | **~11 週** |
| 月維運 | NT$ 9,000~14,500 | **~NT$ 3,150** |

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
- PetPoint 折抵 API 串接

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
| **合計** | **NT$ 3,150~5,550** | 比 v1.0 省 60~70% |

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

1. **立即行動**：確認 v1.1 提案與預算上限，啟動 Linode WP 環境與 Clerk 帳號
2. **本週**：完成 PetRisk.AI 操作流程 Wireframe；採購 miniOrange OAuth plugin
3. **下週**：與葉老師會面，確認期刊論文格式與 RAG ingestion 流程
4. **持續**：雙北市獸醫公會 Demo 安排與時程確認

---

## 九、聯絡資訊

| 項目 | 內容 |
|------|------|
| **提案人** | Max Hsu |
| **職稱** | 技術架構師 / 全端工程師 |
| **GitHub** | https://github.com/maxmilian/mr-pet |
| **備註** | 本提案 v1.1 採 WordPress 混合架構，於 v1.0 全自建版本基礎上根據可行性評估調整 |

---

*本文件為機密提案，僅供 Mr.Pet × PetRisk.AI 內部參考使用。*
