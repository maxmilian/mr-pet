# 客戶新需求：Shared Platform + 三引擎四層架構

> **來源**：客戶（Mt.Luke 路克山 莊彥衫）2026-09-07 17:38–17:39 訊息
> **接收方**：安璣有限公司／技術長 徐銘宏 Max Hsu
> **關聯文件**：
> - [Mr.Pet × PetRisk.AI 系統建置提案書 v1.2](../proposal/2026-05-09-mrpet-petrisk-ai-proposal.md)
> - [VetPoint VMCS V1 開發報價 v1.0](../proposal/2026-09-04-vetpoint-vmcs-quotation.md)

---

## 一、客戶原始訊息（逐字歸檔）

> **17:38** 第一個引擎：**Mr.Pet Subscription Commerce & Member Health Engine**。
> 它負責「飼主與寵物」。用以取得與留存會員。全程於網頁中（不是 APP）進行訂閱方案選擇、進行付款（轉帳或信用卡）、得知月光盒內容、填寫會員資料與 Pet Profile、搜尋合作院所並進行預約與收到看診資訊確認的手機簡訊、獲得來自 Mr.Pet Health Center 的 email 查閱獸醫看診結果（醫療影像與診斷報告、主治獸醫認可的第三方 PetRisk.ai second opinion）。

> **17:39** 第二個引擎：**VetPoint Medical Clearing Engine**。
> 它負責「價值轉換工作流」。用以將 Mr.Pet 給會員的權益，變成真正在獸醫院能用的醫療折抵。並精確完成 Member Wallet → Redemption → Clinic Payable → Settlement 的高安全性消費與醫療行為循環。簡單清楚地解決怎麼讓 B2C 消費真正流入 B2B 醫療場景。全程一樣在網頁中（不是 APP）中進行。

> **17:39** 第三個引擎：**PetRisk.ai Clinical Decision Support Engine**。
> 它負責「數位知識應用」a.k.a GCP 中的 AI 模型 + API。把 Clinical Case 與全球獸醫學資料 Evidence 結合（向量查閱），產出有限、可引用、可審閱、可追溯的 Second Opinion，或不產出。它主要解決的是：如何讓一次日常的醫病看診，變成更好的醫療理解與溝通。如何讓獸醫快速進行與完整的醫療建議掃描。並以 LLM 協助呈現報告，在網頁中進入會員中心查閱。

> **17:39** 在三個引擎進行之前，先建立專案內資訊共享平台 **Shared Platform**。它負責「底層資料的新增、刪減、修改」的共用層。比如：Identity、Global Pet ID、Clinic ID、Consent、RBAC、Audit、Notification、Storage、Event、Adapter、Config、Observability 都屬於這一層。

---

## 二、需求解讀：從「三引擎」變成「一底座 + 三引擎」

VMCS 規格書第 37 節原本定調的是三個引擎的**串接順序**（① → ② → ③）。這次訊息新增了兩件實質變更：

1. **新增第 0 層 Shared Platform**，且明確要求**在三個引擎之前先建**。
2. **三個引擎的載體全部定調為 Web**（訂閱、付款、預約、核銷、報告查閱都在網頁），明確排除原生 App。

```
┌─────────────────────────────────────────────────────────┐
│ ① Subscription    │ ② VetPoint        │ ③ PetRisk.ai    │
│   Commerce &      │   Medical         │   Clinical      │
│   Member Health   │   Clearing        │   Decision      │
├─────────────────────────────────────────────────────────┤
│  ⓪ Shared Platform（共用層，先建）                        │
│  Identity / Global Pet ID / Clinic ID / Consent / RBAC / │
│  Audit / Notification / Storage / Event / Adapter /      │
│  Config / Observability                                  │
└─────────────────────────────────────────────────────────┘
```

---

## 三、Shared Platform 十二元件 vs 現有規劃

| # | 元件 | 現有文件是否涵蓋 | 落差與影響 |
|---|------|-----------------|-----------|
| 1 | **Identity** | ✅ 提案 §3.3.1 Clerk SSO（報價 §7.1 假設不含 IdP 建置） | Clerk 只解決「登入」。跨三引擎的 **User 主檔／跨系統 ID 對映**（WP user ↔ VetPoint member ↔ PetRisk 病患 Token）仍需自建 |
| 2 | **Global Pet ID** | 🟡 僅有 ACF 寵物欄位（§3.1.2）+ Token ID 去識別化（§3.3.2） | **實質新增**。跨系統、跨院所唯一的寵物識別碼；牽涉轉院、多院共病歷、是否對接晶片號 |
| 3 | **Clinic ID** | 🟡 院所資料散落在 VMCS W7（院所端）與 W11（適用院所 Config） | 需抽出**院所主檔**：簽約狀態、適用服務、清算對象、地理位置（引擎 ① 的「搜尋合作院所」直接依賴此表） |
| 4 | **Consent** | 🟡 §3.3.2 僅有「誰看過我的資料」紀錄 | **實質新增**。需版本化同意書引擎（同意版本、撤回、稽核）。這是引擎 ③ second opinion 給飼主查閱的**法遵前提**，不是加分項 |
| 5 | **RBAC** | 🟡 VMCS W10 有五角色 RBAC（5 人日），但僅限 VetPoint 內 | 需上移為跨引擎統一權限模型。**W10 應重寫並上移，避免重複計價** |
| 6 | **Audit** | 🟡 同上，VMCS W10 的 AuditEvent | 同上，上移為共用稽核事件流 |
| 7 | **Notification** | 🔴 提案僅有「點數到期提醒 cron + Email」 | **實質新增且範圍明顯擴大**：看診預約確認**簡訊**、Health Center **報告 email**。需簡訊供應商（三竹／every8d）、樣板管理、送達狀態、費用歸屬 |
| 8 | **Storage** | ✅ §3.2.3 GCS 加密 + IAM | 需抽為共用簽章 URL 服務（醫療影像同時被 ②③ 與會員中心讀取） |
| 9 | **Event** | 🔴 現有設計是點對點 webhook（WP → PetPoint Bridge） | **架構層級變更**：改為共用事件匯流（Pub/Sub）。影響提案 §2.3 mu-plugin 的設計 |
| 10 | **Adapter** | 🟡 綠界金流 plugin 已有，其餘無抽象層 | 金流／簡訊／LINE／院所 PMS 的統一 adapter 介面為新增 |
| 11 | **Config** | 🟡 VMCS W11 商業規則 Config（3 人日），僅限 VetPoint | 上移為跨引擎設定中心，**同樣需避免與 W11 重複計價** |
| 12 | **Observability** | 🔴 提案僅提 Cloud Logging + BigQuery 直查 | Tracing／Metrics／告警為新增；金流與清算系統實務上必要 |

**統計**：12 項中 ✅ 已涵蓋 2 項、🟡 部分涵蓋 7 項、🔴 實質新增 3 項。

---

## 四、對現有報價與時程的影響

### 4.1 VMCS 報價 v1.0（NT$ 833,800／72 人日）需調整

| 影響項 | 說明 |
|--------|------|
| W10（RBAC + Audit，5 人日） | 主體上移 Shared Platform；VMCS 保留 VetPoint 專屬角色對映與防弊規則 |
| W11（Config 化，3 人日） | 主體上移 Shared Platform；VMCS 保留點數業務規則 |
| W1（架構／Schema，4 人日） | 若共用層先定 Identity／Pet ID／Clinic ID，VMCS 的 Schema 需依附共用層而非自訂 |
| §7.1 假設 2、3 | 「不含 IdP 建置」「Subscription／Encounter 由對方系統提供 API」的前提，在共用層由安璣承作時不再成立，須改寫 |
| §7.2 不含項目 | 「原生 App」一項與客戶「全程網頁」一致，可維持 |

> **結論：VMCS 報價 v1.0 的金額與 WBS 在共用層範圍確定前，不宜視為定案。** 建議出 v1.1 標註共用層依賴。

### 4.2 時程風險：嚴格序列會延後第一個可驗證產出

現行 VMCS 方案 B 規劃 **11/28 Stage 1 上線**（院所可實際核銷）。若共用層必須完整先行，這個里程碑會直接順延共用層的開發期。

**建議做法：Thin Platform 先行，隨引擎長出**

- 共用層第一版只做「三引擎當下真的會用到的」：Identity 對映、Global Pet ID、Clinic ID、Config、Audit 骨架
- Consent、Event Bus、Observability、Notification 樣板隨引擎 ①③ 需求逐步補完
- 好處：避免先建一套沒有使用者的抽象層（這是共用層專案最常見的失敗模式），也能保住 Stage 1 的上線節奏

### 4.3 引擎 ① 的範圍比提案 v1.2 大

客戶這次點名的引擎 ① 功能，有數項不在原提案 §3.1（WordPress + WooCommerce 電商）範圍內：

| 新點名功能 | 提案 v1.2 現況 |
|-----------|---------------|
| 訂閱方案選擇與**訂閱制**扣款 | ❌ 原為一次性商品購買（WooCommerce 標準結帳），訂閱需 WC Subscriptions 或自建 |
| 轉帳付款 | 🟡 ECPay ATM 已涵蓋，但**人工轉帳對帳**未涵蓋 |
| 月光盒內容揭露 | ❌ 未涵蓋（訂閱盒每月內容排程與通知） |
| **搜尋合作院所並線上預約** | ❌ 完全未涵蓋。這是一個獨立的預約模組（院所行事曆／時段／取消改期） |
| 看診確認**簡訊** | ❌ 未涵蓋（見 Notification） |
| Health Center **email 報告查閱**（影像＋診斷＋second opinion） | 🟡 §3.2.4 有「Approve 後寫入飼主端 Timeline」，但無 email 發送與會員中心報告頁 |

---

## 五、需向客戶確認的事項

1. 🔴 **Shared Platform 是否由安璣承作？** 若是，需要獨立 WBS 與報價；若否，需要對方交付介面契約與時程，否則 VMCS 無法動工。
2. 🔴 **三引擎的承作分工與順序**：VMCS（②）已有報價，①③ 是否也委由安璣？三者是否同期並行？
3. 🔴 **共用層先行是否可接受 Thin Platform 做法**（見 4.2），或客戶要求完整先建（則 VMCS 11/28 上線日必須放棄）。
4. **WordPress + WooCommerce 混合架構是否仍成立**：Identity／Consent／Pet Profile 若上移共用層，WP 只剩商城角色，需重新確認技術選型。
5. **線上預約模組**的規格深度：只做「送出預約需求 + 院所回覆」，還是要真的做院所行事曆時段管理？兩者工作量差距很大。
6. **簡訊供應商與費用歸屬**：供應商選擇（三竹／every8d）、每則成本由誰負擔、月量估計。
7. **Global Pet ID 是否需對接農業部寵物晶片號**，或僅平台內部識別。
8. **Consent 的法遵範圍**：第三方 AI second opinion 提供給飼主查閱，同意書內容與醫療責任歸屬需法律意見（與 VMCS 報價 §6 第 1、2 項法遵問題同批處理）。

---

## 六、建議下一步

1. 先與客戶確認第五節第 1–3 項（承作分工、順序、Thin Platform），這三項決定後續所有估算的前提。
2. 確認後產出 **Shared Platform WBS 與報價**（粗估 20–30 人日區間，正式數字待範圍定案）。
3. 同步發 **VMCS 報價 v1.1**：標註共用層依賴、把 W10／W11 標為「與共用層取捨」、更新 §7.1 假設。
4. 引擎 ① 依第 4.3 節落差另出增補報價（訂閱制、月光盒、院所搜尋預約為三大新增模組）。
