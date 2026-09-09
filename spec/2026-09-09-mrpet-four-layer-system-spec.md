# Mr.Pet × PetRisk.AI 四層架構系統規格書

> **提案方**：安璣有限公司（統編 25036259）／技術長 徐銘宏 Max Hsu
> **客戶**：貓做科技股份有限公司
> **依據**：貴司 2026-09-07 定調之「Shared Platform + 三引擎」四層架構
> **日期**：2026-09-09　**版本**：v1.0
> **關聯文件**：[四層架構整案開發報價 v1.0](../proposal/2026-09-09-mrpet-master-quotation-client.md)

---

## 〇、文件定位與使用方式

本文件是《四層架構整案開發報價 v1.0》的**技術附件**，逐項展開報價單 WBS（S1–S14、E1–E8、W1–W14、P1–P11）的實作內容。

**本文件的效力**

| 性質 | 說明 |
|------|------|
| **驗收基準** | 本規格所定義的資料模型、狀態機、流程與介面，即為各 Phase 的驗收依據 |
| **範圍界線** | 未寫入本文件者，不在本次開發範圍。範圍變更走報價單 §七 變更管理 |
| **凍結時點** | 各 Phase 啟動時凍結該 Phase 對應章節；凍結後的變更以書面 Change Request 提出 |

**閱讀順序建議**

- 決策者：§一（架構總覽）→ §六（端到端流程）→ §九（待確認事項）
- 技術審查：§二～§五（各層規格）→ §七（事件契約）→ §八（非功能需求）

**標記說明**

- 🔴 標記處為**尚待貴司決定**的事項，其決定會改變實作方式，彙整於 §九
- ⚙️ 標記處為可透過 Config 中心調整的參數，不需改程式

---

## 一、架構總覽

### 1.1 四層結構

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│ ① Subscription       │ ② VetPoint Medical   │ ③ PetRisk.ai         │
│   Commerce &         │   Clearing Engine    │   Clinical Decision  │
│   Member Health      │                      │   Support Engine     │
│                      │                      │                      │
│ 飼主與寵物           │ 價值轉換工作流       │ 數位知識應用         │
│ 取得與留存會員       │ Wallet→Redemption→   │ Clinical Case +      │
│                      │ Payable→Settlement   │ Evidence→2nd Opinion │
├──────────────────────┴──────────────────────┴──────────────────────┤
│ ⓪ Shared Platform（共用層）                                          │
│                                                                     │
│  核心層  Identity │ Global Pet ID │ Clinic ID │ Consent │ RBAC/Audit │
│  服務層  Notification │ Storage │ Event │ Config │ Adapter │ O11y    │
└─────────────────────────────────────────────────────────────────────┘
```

**設計原則**

1. **共用層是唯一的主檔來源**（single source of truth）。三引擎不得自建 User、Pet、Clinic 主檔，一律透過共用層 API 或其 Client SDK 取用。
2. **引擎之間不直接呼叫彼此**。跨引擎的互動一律經由共用層的 Event Bus（§3.3）或共用層 API，避免三個引擎兩兩耦合成六條介面。
3. **醫療資料與電商資料實體隔離**。醫療資料（病歷、影像、Second Opinion）不進入 WordPress 資料庫；即使 WordPress 遭入侵，醫療資料仍不受影響。
4. **全部介面為 Web（RWD）**，不含原生 App。飼主端以 PWA 形式提供近似 App 的體驗（可加入主畫面、推播）。

### 1.2 技術棧

| 層／模組 | 技術選型 | 說明 |
|---------|---------|------|
| ⓪ 共用層服務 | Python FastAPI on **GCP Cloud Run** | 無狀態、依流量自動擴縮 |
| ⓪ 共用層資料庫 | **Cloud SQL PostgreSQL**（HA + PITR） | 主檔、Consent、Audit |
| ⓪ 事件匯流 | **GCP Pub/Sub** | 含死信佇列與重送 |
| ⓪ 物件儲存 | **Google Cloud Storage**（CMEK 加密 + IAM） | 醫療影像、報告 PDF |
| ⓪ 身份提供者 | **Clerk**（LINE／Google／Apple Login） | 不自建 IdP |
| ① 商城前台 | **WordPress + WooCommerce**（Linode） | 🔴 續用與否見 §9.1 |
| ① 金流 | **綠界 ECPay**（信用卡定期定額、ATM 虛擬帳號） | 定期定額為訂閱制核心 |
| ① 會員中心／預約 | React SPA，掛於 `mrpet.tw` 子路徑 | 資料源為共用層與引擎 ②③ |
| ② 清算服務 | Python FastAPI on Cloud Run + **Cloud SQL PostgreSQL**（獨立 instance） | 金融資料獨立庫 |
| ② 院所端 | React SPA @ `clinic.mrpet.tw` | |
| ③ AI 推論 | **Vertex AI Gemini Multimodal** | 不自訓模型 |
| ③ 知識檢索 | **PGVector**（同醫療 DB）+ LlamaIndex | RAG 檢索與引用回溯 |
| ③ 獸醫端 | React + Refine + Tailwind @ `vet.mrpet.tw` | |
| 監控 | Cloud Logging / Cloud Trace / Cloud Monitoring | |

### 1.3 網域與部署拓撲

| 網域 | 服務 | 使用者 |
|------|------|--------|
| `mrpet.tw` | WordPress 商城 + 會員中心（React） | C 端飼主 |
| `api.mrpet.tw` | 共用層 API（Cloud Run） | 內部三引擎 |
| `clinic.mrpet.tw` | VetPoint 院所端 | B 端院所櫃檯／管理者 |
| `vet.mrpet.tw` | PetRisk.ai 獸醫端 Dashboard | B 端獸醫師 |
| `admin.mrpet.tw` | Mr.Pet Admin（6 模組） | 貴司營運人員 |

```
                    ┌─────────────┐
   飼主瀏覽器 ─────>│ mrpet.tw    │─┐
                    │ (WP + SPA)  │ │
                    └─────────────┘ │
   院所瀏覽器 ─────>┌─────────────┐ │      ┌──────────────────┐
                    │clinic.mrpet │─┼─────>│ api.mrpet.tw     │
                    └─────────────┘ │      │ 共用層 Cloud Run │
   獸醫瀏覽器 ─────>┌─────────────┐ │      └────────┬─────────┘
                    │ vet.mrpet   │─┘               │
                    └─────────────┘                 │
                                                     ├──> Cloud SQL（共用層）
   ┌──────────────┐   ┌──────────────┐              ├──> Cloud SQL（VetPoint）
   │ VetPoint svc │   │ PetRisk svc  │              ├──> Cloud SQL（醫療+PGVector）
   │ (Cloud Run)  │   │ (Cloud Run)  │              ├──> GCS（影像）
   └──────┬───────┘   └──────┬───────┘              └──> Pub/Sub
          └──────────┬───────┘
                     v
              GCP Pub/Sub（事件匯流）
```

### 1.4 環境

| 環境 | 用途 | 資料 | 備註 |
|------|------|------|------|
| `dev` | 開發 | 假資料 | |
| `staging` | UAT、貴司驗收 | 去識別化樣本 | 與 production 同架構 |
| `demo` | Pitch Demo 專用 | 純測試資料 | **與 production 完全隔離**，Demo 結束後可關閉 |
| `production` | 正式 | 真實資料 | 限額試營運見報價單 §八 |

---

## 二、⓪ Shared Platform — 核心層規格

> 對應報價單 S1–S6。此層為 Phase 0 主要交付物，完成後三引擎解除依賴、可並行開發。

### 2.1 S1 — 架構、服務邊界與 API 契約

**交付物**

1. **OpenAPI 3.1 契約檔**（`shared-platform.openapi.yaml`），涵蓋 §2.2–§2.6 與 §三 所有對外端點
2. **Client SDK**（Python 與 TypeScript 各一份，自契約產生），供三引擎引用
3. **資料庫 Schema 與 Migration 腳本**（採版本化 migration，可前進可回滾）
4. **接入指引文件**：三引擎如何取得 token、如何呼叫、錯誤碼對照

**API 通則**

| 項目 | 規範 |
|------|------|
| 協定 | REST over HTTPS，JSON |
| 認證 | Clerk 簽發之 JWT，經共用層驗章；服務間呼叫用 GCP service account token |
| 版本 | 路徑前綴 `/v1/`；破壞性變更升為 `/v2/`，舊版並存至少 3 個月 |
| 冪等 | 所有寫入端點接受 `Idempotency-Key` header，24 小時內重送回傳原結果 |
| 分頁 | cursor-based（`?cursor=&limit=`），不使用 offset |
| 錯誤格式 | `{"error": {"code": "PET_NOT_FOUND", "message": "...", "trace_id": "..."}}` |
| 時間 | 一律 UTC ISO-8601 儲存與傳輸；顯示層轉 `Asia/Taipei` |
| 金額 | 整數（新台幣元），不使用浮點數 |

**API 契約凍結**是 Phase 0 的關鍵里程碑：契約凍結後三引擎即可並行開發，不需等待共用層全部完工。

### 2.2 S2 — Identity（身份與跨系統對映）

**職責**：Clerk 只解決「登入」。本模組解決「同一個人在四個系統裡是誰」。

**資料模型**

```
users
  id                UUID       PK
  clerk_user_id     TEXT       UNIQUE, NOT NULL
  email             TEXT
  phone             TEXT       -- E.164 格式，簡訊發送依據
  display_name      TEXT
  status            ENUM       ACTIVE / SUSPENDED / DELETED
  created_at        TIMESTAMPTZ
  updated_at        TIMESTAMPTZ

identity_mappings                    -- 跨系統 ID 對映
  id                UUID       PK
  user_id           UUID       FK users
  system            ENUM       WORDPRESS / VETPOINT / PETRISK
  external_id       TEXT       -- WP user ID、VetPoint member ID…
  linked_at         TIMESTAMPTZ
  UNIQUE (system, external_id)
  UNIQUE (user_id, system)
```

**主要端點**

| 方法 | 路徑 | 說明 |
|------|------|------|
| `GET` | `/v1/users/me` | 由 JWT 解析當前使用者，回傳含各系統 external_id |
| `GET` | `/v1/users/{user_id}` | 需 RBAC 授權 |
| `POST` | `/v1/users/{user_id}/mappings` | 建立跨系統對映（首次登入某引擎時自動觸發） |
| `GET` | `/v1/users/lookup?system=&external_id=` | 反查，供引擎自舊有 ID 找到共用層 user |

**設計要點**

- WordPress 端以 mu-plugin 於使用者首次登入時呼叫 `/mappings` 建立對映，不在 WP 資料庫另存身份主檔
- 使用者刪除採**軟刪除**（status = DELETED）；實體刪除須連動 §2.5 Consent 與 §2.6 Audit 的保存義務，走人工程序

### 2.3 S3 — Global Pet ID（寵物主檔與去識別化）

**職責**：跨系統、跨院所唯一的寵物識別碼。這是三引擎共同的資料樞紐——引擎 ① 的 Pet Profile、引擎 ② 的核銷歸屬、引擎 ③ 的病歷，指的必須是同一隻動物。

**資料模型**

```
pets
  global_pet_id     UUID       PK          -- 對外唯一識別碼
  owner_user_id     UUID       FK users
  name              TEXT
  species           ENUM       DOG / CAT
  breed_code        TEXT                   -- 對應品種代碼表
  sex               ENUM       M / F / M_NEUTERED / F_SPAYED / UNKNOWN
  birth_date        DATE
  microchip_no      TEXT       NULL        -- 🔴 是否介接農業部，見 §9.1
  weight_kg         NUMERIC(5,2) NULL
  status            ENUM       ACTIVE / DECEASED / MERGED / DELETED
  merged_into       UUID       NULL FK pets  -- 重複合併時指向存續紀錄
  created_at        TIMESTAMPTZ

pet_tokens                                 -- 醫療端去識別化
  token_id          UUID       PK          -- 醫療系統只看得到這個
  global_pet_id     UUID       FK pets, UNIQUE
  issued_at         TIMESTAMPTZ
```

**去識別化規則**

- 引擎 ③（PetRisk.ai）**只儲存 `token_id`**，其資料庫內不存飼主姓名、電話、Email，也不存 `global_pet_id`
- 由 `token_id` 反查真實身分的能力僅共用層具備，且每次反查寫入 §2.5 的存取軌跡
- 獸醫端 Dashboard 顯示飼主資訊時，是前端向共用層以使用者自身權限即時取得，非由引擎 ③ 資料庫供給

**重複合併**：同一隻寵物若因飼主重複建檔而有兩筆紀錄，合併時保留較早者為存續紀錄，另一筆 status 改 MERGED 並填 `merged_into`。所有引擎查詢時遇 MERGED 一律自動導向存續紀錄，歷史交易不改寫。

### 2.4 S4 — Clinic ID（院所主檔）

**職責**：院所是引擎 ① 的搜尋標的、引擎 ② 的清算對象、引擎 ③ 的租戶單位，三者必須指向同一份主檔。

**資料模型**

```
clinics
  clinic_id         UUID       PK
  name              TEXT
  tax_id            TEXT                   -- 統一編號，清算開票依據
  contract_status   ENUM       PROSPECT / ACTIVE / SUSPENDED / TERMINATED
  contract_start    DATE
  contract_end      DATE       NULL
  services          TEXT[]                 -- 適用服務代碼，引擎 ① 搜尋條件
  specialties       TEXT[]                 -- 科別
  address           TEXT
  district          TEXT                   -- 行政區，搜尋篩選用
  lat, lng          NUMERIC                -- 地圖與距離排序
  phone             TEXT
  business_hours    JSONB                  -- 引擎 ① 預約時顯示
  settlement_bank   TEXT                   -- 清算匯款帳戶
  settlement_acct   TEXT
  status            ENUM       ACTIVE / INACTIVE

clinic_users                               -- 院所人員綁定
  clinic_id         UUID       FK clinics
  user_id           UUID       FK users
  role              ENUM       CLINIC_ADMIN / CLINIC_OPERATOR / VET
  PRIMARY KEY (clinic_id, user_id, role)
```

**設計要點**

- **只有 `contract_status = ACTIVE` 的院所可執行核銷與出現在引擎 ① 的搜尋結果**，此判斷在共用層統一實施，不由各引擎各自判斷
- 清算帳戶資訊為機敏欄位，API 預設不回傳，需 `FINANCE` 權限方可讀取

### 2.5 S5 — Consent（版本化同意書）

**職責**：這是引擎 ③ 將 AI Second Opinion 提供給飼主查閱的**法遵前提**，不是加分項。同意書會改版，且飼主可撤回，因此必須版本化。

**資料模型**

```
consent_definitions
  consent_type      TEXT       -- TERMS / PRIVACY / MEDICAL_DATA / AI_SECOND_OPINION / MARKETING
  version           INT
  content_uri       TEXT       -- 同意書全文（GCS）
  content_hash      TEXT       -- SHA-256，證明當時簽的是哪一版
  effective_from    TIMESTAMPTZ
  PRIMARY KEY (consent_type, version)

consents
  id                UUID       PK
  user_id           UUID       FK users
  consent_type      TEXT
  version           INT
  scope_pet_id      UUID       NULL   -- 針對特定寵物的授權（如醫療資料）
  granted_at        TIMESTAMPTZ
  revoked_at        TIMESTAMPTZ NULL
  evidence          JSONB      -- IP、User-Agent、勾選時間戳

data_access_logs                     -- 「誰看過我的資料」
  id                UUID       PK
  actor_user_id     UUID       -- 存取者（獸醫、客服）
  actor_clinic_id   UUID NULL
  subject_pet_id    UUID
  resource_type     TEXT       -- MEDICAL_IMAGE / DIAGNOSIS / SECOND_OPINION
  resource_id       TEXT
  action            ENUM       VIEW / DOWNLOAD / EXPORT
  purpose           TEXT       -- 存取目的
  accessed_at       TIMESTAMPTZ
```

**強制規則**

| 情境 | 規則 |
|------|------|
| 存取醫療資料 | 必須存在有效的 `MEDICAL_DATA` 同意（未撤回），否則共用層 Storage 拒發簽章 URL |
| 查閱 AI Second Opinion | 必須額外存在有效的 `AI_SECOND_OPINION` 同意 |
| 同意書改版 | 既有同意不自動遷移；下次登入時提示重新同意，未同意者維持舊版權限範圍 |
| 撤回 | 撤回後即時失效（非次日）；已寫入病歷的歷史紀錄不刪除，但飼主端不再呈現 |
| 存取軌跡 | 每一次醫療資料存取都寫入 `data_access_logs`，飼主可於會員中心自行查閱 |

🔴 同意書的法律文字與責任邊界需由貴司委任律師擬定，安璣負責實作版本化機制與強制檢核（見 §9.2）。

### 2.6 S6 — RBAC 與 Audit

**職責**：跨三引擎的統一權限模型與稽核事件流。各引擎不自建權限系統。

**角色定義**

| 角色 | 範圍 | 典型權限 |
|------|------|---------|
| `PET_OWNER` | 自身 | 讀寫自己的 Pet Profile、查閱自己的錢包與報告 |
| `CLINIC_OPERATOR` | 單一院所 | 查詢會員餘額、執行核銷、查看本院交易 |
| `CLINIC_ADMIN` | 單一院所 | 上列 + 院所資料維護、本院月結查看、人員管理 |
| `VET` | 單一院所 | 上列 + 病歷讀寫、影像上傳、Second Opinion 審閱與 Approve |
| `MRPET_ADMIN` | 全平台 | Admin 6 模組、手動發點、院所管理 |
| `MRPET_FINANCE` | 全平台 | 月結確認、付款登記、清算帳戶讀取 |
| `MRPET_AUDITOR` | 全平台 | 唯讀：稽核日誌、交易查詢，**不可寫入** |

> 引擎 ② V1 的操作介面對應 `MRPET_ADMIN`、`CLINIC_ADMIN`、`CLINIC_OPERATOR` 三角色（見報價單 §2.5）；`MRPET_FINANCE`、`MRPET_AUDITOR`、`VET` 的權限模型於共用層一併建立，介面則分屬引擎 ②③ 與 Phase 2。

**權限模型**：`(role, resource_type, action)` 三元組 + 範圍限定（`scope_clinic_id` / `scope_user_id`）。權限判斷一律由共用層 SDK 執行，引擎不得自行以 if-else 判斷角色。

**Audit 事件**

```
audit_events
  id                UUID       PK
  actor_user_id     UUID
  actor_role        TEXT
  actor_ip          INET
  action            TEXT       -- POINTS_MANUAL_GRANT / REDEMPTION_REVERSE / …
  resource_type     TEXT
  resource_id       TEXT
  before            JSONB      NULL   -- 變更前狀態
  after             JSONB      NULL   -- 變更後狀態
  trace_id          TEXT              -- 串接 §3.6 分散式追蹤
  occurred_at       TIMESTAMPTZ
```

- **append-only**：資料庫層以權限限制 UPDATE／DELETE，僅允許 INSERT
- **必須記錄的動作**：所有點數異動、核銷、反沖、月結狀態變更、權限授予、醫療資料存取、Second Opinion 的 Approve／修正
- 保存期限 ⚙️ 預設 7 年（配合商業會計法憑證保存年限）

---
## 三、⓪ Shared Platform — 服務層規格

> 對應報價單 S7–S12。此層隨引擎需求分批交付（S5、S7、S11 於 Phase 1；S6、S8、S9、S12 於 Phase 2）。

### 3.1 S7 — Notification（通知）

**職責**：貴司 9/7 明確點名兩項通知——「收到看診資訊確認的手機簡訊」與「來自 Mr.Pet Health Center 的 Email」。本模組為兩者的共同基礎。

**通道與供應商**

| 通道 | 供應商 | 用途 |
|------|--------|------|
| SMS | 🔴 三竹或 every8d（見 §9.4） | 預約確認、核銷確認碼、重要異動 |
| Email | SendGrid 或 Amazon SES | 報告通知、訂閱扣款、月結對帳表 |
| LINE | LINE Messaging API（Phase 2 預留） | Adapter 已預留介面，本次不實作發送 |

**資料模型**

```
notification_templates
  template_id       TEXT       PK    -- APPOINTMENT_CONFIRMED / REPORT_READY / …
  channel           ENUM       SMS / EMAIL
  subject           TEXT       NULL  -- Email 專用
  body              TEXT             -- 支援 {{變數}} 佔位
  version           INT
  is_active         BOOLEAN

notification_deliveries
  id                UUID       PK
  template_id       TEXT
  channel           ENUM
  recipient         TEXT             -- 手機或 Email（遮罩後入 log）
  user_id           UUID       NULL
  payload           JSONB            -- 變數值
  status            ENUM       QUEUED / SENT / DELIVERED / FAILED / BOUNCED
  provider_msg_id   TEXT       NULL
  attempt_count     INT
  last_error        TEXT       NULL
  created_at, sent_at, delivered_at  TIMESTAMPTZ
```

**行為規範**

- 發送一律**非同步**（經 Pub/Sub），呼叫端不因簡訊供應商延遲而阻塞
- 失敗重送 ⚙️ 預設 3 次，間隔 1／5／30 分鐘；仍失敗則標記 FAILED 並觸發告警
- 簡訊內容不含醫療診斷內容，僅含「報告已就緒，請至會員中心查閱」等指引 + 安全連結
- 送達狀態回寫（供應商 webhook），供貴司查核「簡訊到底有沒有送到」

**V1 通知清單**

| template_id | 通道 | 觸發時機 |
|-------------|------|---------|
| `APPOINTMENT_REQUESTED` | SMS | 飼主送出預約請求 |
| `APPOINTMENT_CONFIRMED` | SMS | 院所受理預約 |
| `APPOINTMENT_DECLINED` | SMS | 院所婉拒或建議改期 |
| `REDEMPTION_CONFIRM` | SMS | 核銷待飼主確認（含確認碼） |
| `REPORT_READY` | Email | 獸醫 Approve 報告後 |
| `SUBSCRIPTION_CHARGED` | Email | 訂閱扣款成功 |
| `SUBSCRIPTION_FAILED` | Email + SMS | 扣款失敗 |
| `MONTHLY_BOX_SHIPPED` | Email | 月光盒出貨 |
| `STATEMENT_READY` | Email | 院所月結對帳表產出 |

### 3.2 S8 — Storage（醫療影像與報告儲存）

**職責**：醫療影像同時被引擎 ③（獸醫端檢視）與引擎 ①（飼主端會員中心）讀取，存取授權必須集中管理。

**設計**

- 檔案存於 GCS，**bucket 不公開**，一律以短效簽章 URL 存取
- 簽章 URL 有效期 ⚙️ 預設 15 分鐘，且與請求者身分綁定
- 發放簽章 URL 前，共用層強制檢查兩件事：**（a）RBAC 是否允許**、**（b）Consent 是否有效**。兩者任一不通過即拒絕，並寫入 `data_access_logs`
- 上傳時計算 SHA-256 存入 metadata，作為 §6.3 可追溯性的輸入指紋

```
stored_objects
  object_id         UUID       PK
  gcs_uri           TEXT
  content_type      TEXT
  size_bytes        BIGINT
  sha256            TEXT
  owner_pet_id      UUID       -- 授權判斷依據
  uploaded_by       UUID
  classification    ENUM       MEDICAL_IMAGE / REPORT_PDF / GENERAL
  created_at        TIMESTAMPTZ
```

- 生命週期 ⚙️：醫療影像保存 7 年後轉 Coldline；一般附件 1 年後轉 Nearline

### 3.3 S9 — Event（事件匯流）

**職責**：取代點對點 webhook。三引擎透過事件解耦，避免兩兩耦合。

**事件契約通則**

```json
{
  "event_id": "01J8...",
  "event_type": "points.redeemed",
  "event_version": 1,
  "occurred_at": "2026-11-03T06:12:44Z",
  "trace_id": "4bf92f...",
  "actor": { "user_id": "...", "role": "CLINIC_OPERATOR" },
  "data": { }
}
```

| 規範 | 內容 |
|------|------|
| 傳遞保證 | at-least-once。**訂閱端必須自行去重**（依 `event_id`） |
| 順序 | 不保證全域順序；需要順序處理者以 `ordering_key`（如 `wallet_id`）指定 |
| 死信 | 連續失敗 ⚙️ 5 次進入死信佇列並告警，可人工重放 |
| 發布方式 | **Transactional Outbox**：業務交易與事件寫入同一 DB 交易，再由背景程序投遞，避免「交易成功但事件遺失」 |
| 版本 | 事件結構破壞性變更時 `event_version` 遞增，新舊並行至少 3 個月 |

**V1 事件清單**

| event_type | 發布者 | 主要訂閱者 |
|-----------|--------|-----------|
| `user.registered` | 共用層 | ①③ |
| `pet.created` / `pet.updated` | 共用層 | ②③ |
| `consent.granted` / `consent.revoked` | 共用層 | ③（撤回時停止呈現） |
| `clinic.activated` / `clinic.suspended` | 共用層 | ①（搜尋結果）②（核銷資格） |
| `subscription.activated` | ① | ②（發點） |
| `subscription.renewed` | ① | ②（發點） |
| `subscription.payment_failed` | ① | 共用層（通知） |
| `points.earned` / `points.redeemed` / `points.reversed` | ② | ①（會員中心顯示） |
| `appointment.requested` / `.accepted` / `.declined` | ① | 共用層（通知） |
| `encounter.completed` | ① 或院所端 | ②（看診回饋發點）③（建立 case） |
| `second_opinion.approved` | ③ | ①（會員中心報告頁）、共用層（通知） |
| `statement.confirmed` | ② | 共用層（通知院所） |

### 3.4 S10 — Config（設定中心）

**職責**：本文件標記 ⚙️ 的所有參數集中於此，調整不需改程式、不需重新部署。

```
config_entries
  scope             TEXT       -- GLOBAL / CLINIC:{id} / ENGINE:VETPOINT
  key               TEXT
  value             JSONB
  value_type        ENUM       INT / STRING / BOOL / JSON
  version           INT
  effective_from    TIMESTAMPTZ
  updated_by        UUID
  PRIMARY KEY (scope, key, version)
```

- **版本化**：改值不覆寫，寫入新版本，保留歷史（稽核需要知道「當時的折抵上限是多少」）
- **範圍繼承**：`CLINIC:{id}` 覆寫 `GLOBAL`，可對個別院所設不同限額
- **Feature Flag**：以同一機制實作，供限額試營運（報價單 §八）逐步開放功能

### 3.5 S11 — Adapter（外部服務統一介面）

**職責**：外部供應商會換。以統一介面隔離，換供應商時只換 Adapter 實作，引擎程式不動。

| Adapter | V1 實作 | 介面涵蓋 |
|---------|--------|---------|
| `PaymentAdapter` | 綠界 ECPay | 定期定額建立／取消、ATM 取號、交易查詢、對帳檔下載、webhook 驗章 |
| `SmsAdapter` | 🔴 三竹或 every8d | 發送、狀態查詢、餘額查詢 |
| `EmailAdapter` | SendGrid／SES | 發送、送達 webhook |
| `LineAdapter` | **僅定義介面，不實作** | Phase 2 |
| `PmsAdapter` | **僅定義介面，不實作** | 院所既有系統整合，Phase 2+ |

### 3.6 S12 — Observability

| 面向 | 實作 |
|------|------|
| 結構化日誌 | JSON 格式，每筆帶 `trace_id`、`user_id`、`clinic_id`；**醫療內容與個資不入日誌**，僅記識別碼 |
| 分散式追蹤 | W3C Trace Context，跨共用層與三引擎傳遞，可還原一次核銷的完整呼叫鏈 |
| 指標 | API 延遲（p50／p95／p99）、錯誤率、Pub/Sub 積壓量、AI 推論耗時與棄權率、簡訊送達率 |
| 告警 | 見 §8.3 |

---

## 四、① Subscription Commerce & Member Health Engine

> 對應報價單 E1–E8。WooCommerce 商城本體已在《系統建置提案書 v1.2》範圍內，本章為 9/7 新定調的增補模組。
>
> 🔴 本章全部以「WordPress + WooCommerce 續用」為前提。若改為自建前台，實作方式與工作量均需重估（見 §9.1）。

### 4.1 E1 — 訂閱制

**資料模型**

```
subscription_plans
  plan_id           TEXT       PK      -- 對應 BP 之 598 / 1598 / 2989 三階
  name              TEXT
  price_monthly     INT                -- 新台幣元
  points_per_cycle  INT                -- 每期發放之 VetPoints
  box_included      BOOLEAN            -- 是否含月光盒
  benefits          JSONB              -- 權益清單（會員中心呈現）
  is_active         BOOLEAN

subscriptions
  id                UUID       PK
  user_id           UUID
  plan_id           TEXT
  status            ENUM       PENDING / ACTIVE / PAST_DUE / CANCELLED / EXPIRED
  payment_method    ENUM       CREDIT_PERIODIC / BANK_TRANSFER
  ecpay_periodic_id TEXT       NULL    -- 綠界定期定額委託編號
  started_at        TIMESTAMPTZ
  current_period_start / _end  TIMESTAMPTZ
  next_billing_at   TIMESTAMPTZ
  cancel_at_period_end BOOLEAN
  cancelled_at      TIMESTAMPTZ NULL

subscription_invoices
  id                UUID       PK
  subscription_id   UUID
  period_start / _end   DATE
  amount            INT
  status            ENUM       PENDING / PAID / FAILED / REFUNDED
  paid_at           TIMESTAMPTZ NULL
  ecpay_trade_no    TEXT       NULL
  retry_count       INT
```

**狀態機**

```
PENDING ──付款成功──> ACTIVE ──扣款失敗──> PAST_DUE ──重試成功──> ACTIVE
                        │                      │
                        │                      └──重試耗盡──> EXPIRED
                        └──使用者取消──> CANCELLED（期末失效）
```

**扣款失敗重試** ⚙️：預設第 1、3、5 日各重試一次；每次失敗寄 Email + 簡訊；三次皆失敗轉 EXPIRED 並停止權益。

**發點串接**：`subscription.activated` 與 `subscription.renewed` 事件由引擎 ② 訂閱，據 `points_per_cycle` 發放 VetPoints。**發點為事件驅動，非同步**；引擎 ② 以 `event_id` 去重，確保同一期不重複發點。

### 4.2 E2 — 轉帳付款與對帳

- ATM 虛擬帳號：每筆訂單向綠界取號，帳號與訂單一對一，避免人工比對錯誤
- 匯款登記：綠界入帳 webhook 自動核銷；若飼主匯入非虛擬帳號（如直接匯公司戶），提供 Admin 人工登記介面
- 對帳：每日下載綠界對帳檔，與系統訂單比對，差異列表供營運人員處理
- 權益開通：確認入帳後才觸發 `subscription.activated`

### 4.3 E3 — 月光盒

```
monthly_boxes
  box_id            UUID       PK
  ship_month        DATE               -- 2026-11-01 表 11 月份盒
  plan_ids          TEXT[]             -- 適用方案
  contents          JSONB              -- 品項、數量、圖片、說明
  reveal_at         TIMESTAMPTZ        -- 內容揭露時間（在此之前不顯示）
  status            ENUM       DRAFT / SCHEDULED / SHIPPED

box_shipments
  id                UUID       PK
  box_id            UUID
  subscription_id   UUID
  status            ENUM       PENDING / SHIPPED / DELIVERED / RETURNED
  tracking_no       TEXT       NULL
  shipped_at        TIMESTAMPTZ NULL
```

- 「得知月光盒內容」＝ 會員中心的當月盒內容頁，於 `reveal_at` 後開放，出貨時另發 Email
- 出貨批次管理提供 CSV 匯出（收件人、地址、品項），供物流商作業；**不含物流商 API 介接**

### 4.4 E4 — 合作院所搜尋

- 資料來源為共用層 `clinics`（S4），引擎 ① 不另存院所資料
- 篩選條件：行政區、科別、適用服務、是否可線上預約
- 排序：預設依距離（瀏覽器定位，使用者可拒絕；拒絕時改依行政區）
- 呈現：清單 + 地圖（Google Maps JavaScript API）
- **僅顯示 `contract_status = ACTIVE` 之院所**

### 4.5 E5 — 線上預約（請求制 V1）

> 🔴 V1 為**請求制**：飼主提出希望時段，院所人工受理。若需真正的時段庫存管理（院所班表、可預約時段、同時段容量、自動確認），見報價單 §2.7 選配（+8 人日）。

```
appointments
  id                UUID       PK
  clinic_id         UUID
  user_id           UUID
  pet_id            UUID
  purpose           TEXT               -- 就診事由
  preferred_slots   JSONB              -- 飼主提供 1–3 個希望時段
  confirmed_at      TIMESTAMPTZ NULL   -- 院所確認之實際時段
  status            ENUM
  clinic_note       TEXT       NULL    -- 院所回覆訊息
  created_at        TIMESTAMPTZ
```

**狀態機**

```
REQUESTED ──院所受理──> ACCEPTED ──看診完成──> COMPLETED
    │                      │
    │                      ├──飼主取消──> CANCELLED
    │                      └──未到──> NO_SHOW
    ├──院所婉拒──> DECLINED
    └──院所建議改期──> RESCHEDULE_PROPOSED ──飼主接受──> ACCEPTED
                                            └──飼主拒絕──> CANCELLED
```

- 每次狀態變更觸發對應簡訊（§3.1）
- 院所端於 `clinic.mrpet.tw` 有預約待辦清單
- ⚙️ 院所未於 N 小時（預設 24）內回覆，系統寄提醒；逾 48 小時自動標記 EXPIRED 並通知飼主

### 4.6 E6 — Health Center 報告查閱

**這是三個引擎交會之處**：影像與診斷來自引擎 ③、飼主身分與 Consent 來自共用層、通知經 S7 發送。

```
health_reports
  id                UUID       PK
  user_id           UUID
  pet_id            UUID
  clinic_id         UUID
  case_id           UUID               -- 引擎 ③ 之 clinical_case
  visit_date        DATE
  vet_diagnosis     TEXT               -- 主治獸醫診斷
  has_second_opinion BOOLEAN
  image_object_ids  UUID[]             -- 指向共用層 stored_objects
  published_at      TIMESTAMPTZ
  status            ENUM       DRAFT / PUBLISHED / WITHDRAWN
```

**呈現規則**

1. 報告僅在引擎 ③ 的 `second_opinion.approved` 事件（或獸醫直接發布診斷）後才建立並 `PUBLISHED`
2. Second Opinion 區塊**必須明確標示**：由 AI 產生、經主治獸醫 {姓名} 於 {時間} 認可、僅供輔助參考不構成診療行為
3. 若 AI 於該案棄權（§6.4），報告仍發布，Second Opinion 區塊顯示「本案未產出 AI 建議」及原因，**不留白、不隱藏**
4. 飼主未同意 `AI_SECOND_OPINION` 者，不顯示該區塊，其餘照常
5. 影像以簽章 URL 呈現，每次開啟寫入 `data_access_logs`
6. Email 僅含「報告已就緒」與登入連結，**不夾帶醫療內容、不附影像**

---
### 4.7 E7 — 會員資料與 Pet Profile

- 表單流程：註冊 → 同意條款（S5）→ 基本資料 → 新增寵物（S3）→ 完成
- Pet Profile 欄位對應 §2.3 `pets` 表，**資料寫入共用層而非 WordPress**
- 一位飼主可有多隻寵物；多寵物共用單一 VetPoints 錢包（見 §5.2 設計說明）
- 提供資料匯出（個資法查閱權）與帳號刪除申請入口

### 4.8 E8 — 測試與 UAT

- 訂閱下單至權益開通全鏈路 E2E
- 扣款失敗重試情境
- 預約狀態機全路徑
- 報告呈現在四種同意狀態下的行為（全同意／未同意 AI／撤回醫療同意／同意書改版待重簽）

---

## 五、② VetPoint Medical Clearing Engine

> 對應報價單 W1–W14，依貴司提供之《VetPoint Medical Clearing System》規格書全 37 節。
> **V1 範圍界線見報價單 §2.5**（六項明確移出 V1 之功能），本章僅描述 V1 實作範圍。

### 5.1 核心不變量

本引擎處理可清算的金錢等價物，以下四項為系統的不變量，任何實作與變更都不得違反：

| # | 不變量 | 保障機制 |
|---|--------|---------|
| 1 | **Ledger 不可竄改** | append-only，DB 權限禁止 UPDATE／DELETE；更正一律以反向分錄表達 |
| 2 | **餘額必須可由 Ledger 重算** | `wallets.balance` 為快取；每日排程重算比對，差異即告警 |
| 3 | **同一筆核銷不得重複成立** | Idempotency Key + 唯一索引，資料庫層強制 |
| 4 | **點數不得超扣** | 保留機制（Reservation）+ Row Lock，跨院併發亦成立 |

### 5.2 資料模型

```
wallets
  id                UUID       PK
  user_id           UUID       UNIQUE     -- 會員層級單一錢包
  balance           INT                   -- 快取值，可由 ledger 重算
  updated_at        TIMESTAMPTZ

ledger_entries                            -- append-only
  id                UUID       PK
  wallet_id         UUID
  entry_type        ENUM       EARN / REDEEM / EXPIRE /
                               REVERSAL_EARN / REVERSAL_REDEEM /
                               ADJUST_IN / ADJUST_OUT        -- 7 類
  points            INT                   -- 正負號依 entry_type
  balance_after     INT                   -- 當下餘額快照
  expires_at        TIMESTAMPTZ NULL      -- 僅 EARN 有效期
  source_entry_id   UUID       NULL       -- REDEEM 指向被扣的 EARN（FIFO 追蹤）
  ref_type          ENUM       SUBSCRIPTION / ENCOUNTER / REDEMPTION / MANUAL
  ref_id            TEXT
  pet_id            UUID       NULL       -- 標記用，不做額度隔離
  idempotency_key   TEXT       UNIQUE NULL
  created_at        TIMESTAMPTZ
  created_by        UUID

point_reservations
  id                UUID       PK
  wallet_id         UUID
  quote_id          UUID
  points            INT
  status            ENUM       HELD / CONSUMED / RELEASED / EXPIRED
  expires_at        TIMESTAMPTZ           -- TTL
  created_at        TIMESTAMPTZ

quotes
  id                UUID       PK
  clinic_id         UUID
  user_id           UUID
  pet_id            UUID       NULL
  eligible_fee      INT                   -- 可折抵之診療費
  requested_points  INT
  status            ENUM       DRAFT / RESERVED / CONFIRMED / EXPIRED / CANCELLED
  ttl_expires_at    TIMESTAMPTZ
  created_by        UUID                  -- 院所操作員
  created_at        TIMESTAMPTZ

redemptions
  id                UUID       PK
  quote_id          UUID       UNIQUE
  clinic_id         UUID
  user_id           UUID
  encounter_id      UUID       NULL       -- 🔴 可為空，見 §9.3
  points_used       INT
  amount            INT                   -- 折抵金額（點數 × 匯率）
  idempotency_key   TEXT       UNIQUE
  status            ENUM       CONFIRMED / REVERSED
  confirmed_at      TIMESTAMPTZ
  reversed_at       TIMESTAMPTZ NULL
  reversal_reason   TEXT       NULL

clinic_payables                           -- 院所應收
  id                UUID       PK
  clinic_id         UUID
  redemption_id     UUID       UNIQUE
  amount            INT
  statement_id      UUID       NULL       -- 歸屬之月結單
  status            ENUM       OPEN / STATEMENTED / REVERSED

statements                                -- V1 三態
  id                UUID       PK
  clinic_id         UUID
  period            DATE                  -- 2026-11-01 表 11 月
  total_amount      INT
  line_count        INT
  status            ENUM       DRAFT / CONFIRMED / PAID
  confirmed_at      TIMESTAMPTZ NULL
  confirmed_by      UUID       NULL
  paid_at           TIMESTAMPTZ NULL
  payment_ref       TEXT       NULL       -- 匯款單號（人工登記）
  UNIQUE (clinic_id, period)

fraud_alerts
  id                UUID       PK
  rule_code         TEXT
  clinic_id         UUID
  user_id           UUID       NULL
  severity          ENUM       LOW / MEDIUM / HIGH
  detail            JSONB
  status            ENUM       OPEN / REVIEWING / DISMISSED / CONFIRMED
  created_at        TIMESTAMPTZ
```

**多寵物設計說明**：錢包在**會員層級**，`pet_id` 僅為交易標記，不做額度隔離。此設計須於院所端與飼主端明確告知，避免「這隻的點數怎麼被那隻用掉」的客訴。

### 5.3 核銷主流程（W3／W4）

```
① 院所查詢會員      POST /v1/clinic/members/lookup
   （雙認證：手機末四碼 + 會員條碼/QR）
        │
        v
② 產生 Quote 並保留點數   POST /v1/quotes
   ├─ 開啟交易，SELECT ... FOR UPDATE 鎖定 wallet
   ├─ 檢查：可用餘額 = balance − 未過期的 HELD 保留
   ├─ 檢查：requested_points ≤ min(可用餘額, eligible_fee ÷ 匯率)
   ├─ 檢查：⚙️ 單筆上限、單日上限、100 點級距
   ├─ 寫入 point_reservations (status=HELD, TTL ⚙️ 預設 15 分鐘)
   └─ 提交交易 → 回傳 quote_id 與折抵金額
        │
        v
③ 飼主確認         簡訊確認碼 或 飼主端 PWA 按鈕
   （備援：QR / PIN，🔴 代操作規則見 §9.3）
        │
        v
④ Confirm（原子交易） POST /v1/redemptions  [Idempotency-Key 必填]
   ├─ 開啟交易
   ├─ ledger_entries  INSERT (REDEEM, FIFO 依 expires_at 由近至遠扣)
   ├─ redemptions     INSERT (status=CONFIRMED)
   ├─ clinic_payables INSERT (status=OPEN)
   ├─ point_reservations UPDATE → CONSUMED
   ├─ wallets.balance UPDATE
   └─ 三表同交易，任一失敗全域 rollback
        │
        v
⑤ 院所端即時顯示入帳與應收；飼主端錢包同步更新
   發布事件 points.redeemed
```

**TTL 到期**：背景排程每分鐘掃描逾期 HELD 保留，改為 EXPIRED 並釋放額度，對應 Quote 轉 EXPIRED。⚙️ 院所端提供「延長一次」按鈕（預設可延長 15 分鐘，僅一次）。

**冪等保證**：相同 `Idempotency-Key` 於 24 小時內重送，回傳首次結果且不重複建立交易。此為資料庫唯一索引強制，非應用層判斷。

### 5.4 完整反沖（W5）

> V1 **僅支援完整反沖**（整筆作廢後重開），不支援部分反沖與跨月調整（見報價單 §2.5）。

```
POST /v1/redemptions/{id}/reverse   { reason }

交易內：
  ├─ ledger_entries  INSERT (REVERSAL_REDEEM, 正值回補, 沿用原 expires_at)
  ├─ redemptions     UPDATE → REVERSED
  ├─ clinic_payables UPDATE → REVERSED（若已歸屬 statement 且該 statement
  │                   狀態為 DRAFT，同步自 statement 移除並重算總額）
  └─ wallets.balance UPDATE
```

**回補的到期日沿用原 EARN 的 `expires_at`**，不重新計算——否則反覆核銷／反沖可無限展期點數。

**限制**：若所屬 statement 已為 CONFIRMED 或 PAID，系統拒絕反沖，須走人工爭議流程（🔴 SLA 見 §9.3）。

### 5.5 月結與清算（W6）

**狀態機（V1 三態）**

```
DRAFT ──財務確認──> CONFIRMED ──登記匯款──> PAID
  ^                     │
  └──退回（僅 DRAFT→CONFIRMED 之前可改）
```

| 階段 | 動作 | 執行者 |
|------|------|--------|
| 月初排程 | 彙總上月所有 `clinic_payables`（status=OPEN）產出 statement（DRAFT） | 系統 |
| 對帳 | 院所於院所端查看明細，有異議於 ⚙️ N 日內提出（🔴 SLA 見 §9.3） | 院所 |
| 確認 | 財務確認金額無誤，statement 轉 CONFIRMED，明細鎖定 | `MRPET_FINANCE` |
| 付款 | 人工匯款後於系統登記匯款單號，轉 PAID | `MRPET_FINANCE` |

- 對帳表匯出：CSV／Excel，含每筆核銷之日期、會員代號、寵物、點數、金額、Encounter
- **V1 不含銀行 API 自動匯款**，付款為人工執行 + 系統登記

### 5.6 防弊規則（W10，V1 三條）

| 代碼 | 規則 | ⚙️ 預設閾值 | 嚴重性 |
|------|------|-----------|--------|
| `FRAUD_RAPID_REPEAT` | 同一會員於同一院所短時間內重複核銷 | 10 分鐘內 ≥ 3 筆 | HIGH |
| `FRAUD_HIGH_AMOUNT` | 單筆折抵金額異常偏高 | 單筆 > NT$ 5,000 | MEDIUM |
| `FRAUD_DAILY_LIMIT` | 單一院所單日折抵總額超限 | 單日 > NT$ 50,000 | HIGH |

- 告警**不阻擋交易**（避免誤擋正常就診），寫入 `fraud_alerts` 並通知營運人員
- 規則為 Rule-based，不含 AI Fraud Detection（規格第 26 節允許 V1 採此做法）

### 5.7 Admin 6 模組（W9）

錢包查詢／手動發點（須填事由，寫入 Audit）／交易查詢／院所管理／月結彙總／稽核日誌。其餘 5 模組屬 Phase 2（報價單 §2.7）。

### 5.8 驗收基準（W13）

1. 同一飼主同時在兩家院所發起 Quote → 點數不得超扣
2. Confirm 重送相同 Idempotency Key → 只成立一筆
3. Ledger 重算餘額 = Wallet balance（連續 10 日零差異）
4. 完整反沖後，Wallet、Ledger、SettlementLine 三方一致
5. 月結彙總金額 = 當月 SettlementLine 加總

---
## 六、③ PetRisk.ai Clinical Decision Support Engine

> 對應報價單 P1–P11。範圍為貴司 BP 五層架構之 **L3（多模態核心）＋ L4（預測與建議）** 的 MVP 切面，並納入 9/7 定調之三項要求：**可引用、可審閱、可追溯**，以及**「或不產出」**。
>
> **不含**：自訓模型（PyTorch／MONAI）、DICOM 與 CT／超音波、CVAT 標註工程、Breed-adjusted Z-Score（L2）、Knowledge Graph、院所 PMS 嵌合（L5）。

### 6.1 設計原則

| # | 原則 | 具體要求 |
|---|------|---------|
| 1 | **有限** | 只在證據充分時產出建議；不足即棄權（§6.4），不勉強生成 |
| 2 | **可引用** | 每一條建議都必須附上檢索命中的文獻段落，無引用者不得輸出 |
| 3 | **可審閱** | 一律先進入獸醫審閱佇列；**未經 Approve 不得呈現給飼主** |
| 4 | **可追溯** | 每次推論留存模型版本、Prompt 版本、語料快照、輸入 hash，可於日後完整重現 |
| 5 | **不構成診療** | 全部輸出標示為輔助參考；主治獸醫為唯一診斷責任人 |

### 6.2 資料模型

> 本引擎資料庫**只存 `pet_token_id`**，不存 `global_pet_id`、飼主姓名、電話或 Email（§2.3 去識別化規則）。

```
clinical_cases
  id                UUID       PK
  clinic_id         UUID
  pet_token_id      UUID                  -- 去識別化識別碼
  vet_user_id       UUID                  -- 主治獸醫
  specialty         TEXT                  -- 專科代碼，🔴 V1 實作 1 個，見 §9.1
  species           ENUM       DOG / CAT
  breed_code        TEXT
  age_months        INT
  sex               TEXT
  chief_complaint   TEXT                  -- 主訴
  clinical_findings TEXT       NULL       -- 理學檢查所見
  status            ENUM       DRAFT / SUBMITTED / IN_REVIEW / CLOSED
  created_at        TIMESTAMPTZ

case_images
  id                UUID       PK
  case_id           UUID
  object_id         UUID                  -- 指向共用層 stored_objects
  modality          ENUM       XRAY                 -- V1 僅 X 光
  body_part         TEXT
  sha256            TEXT                  -- 可追溯性輸入指紋
  uploaded_at       TIMESTAMPTZ

inference_runs                            -- 每次推論一筆，永不刪除
  id                UUID       PK
  case_id           UUID
  model_id          TEXT                  -- 如 gemini-multimodal
  model_version     TEXT                  -- 精確版本字串
  prompt_version    TEXT                  -- Prompt 模板版本
  retrieval_config  JSONB                 -- top_k、相似度門檻等
  input_hash        TEXT                  -- 影像 sha256 + 結構化輸入的合併雜湊
  outcome           ENUM       PRODUCED / ABSTAINED / ERROR
  abstain_reason    ENUM NULL  INSUFFICIENT_EVIDENCE / LOW_CONFIDENCE /
                               POOR_INPUT_QUALITY / OUT_OF_SCOPE
  confidence        NUMERIC    NULL
  latency_ms        INT
  created_at        TIMESTAMPTZ

inference_citations                       -- 引用與語料快照
  id                UUID       PK
  inference_run_id  UUID
  doc_id            UUID
  chunk_id          UUID
  snapshot_text     TEXT                  -- 命中當下的原文，語料日後更新亦不影響追溯
  similarity        NUMERIC
  cited_in_output   BOOLEAN               -- 是否真的被引用於輸出

second_opinions
  id                UUID       PK
  inference_run_id  UUID
  case_id           UUID
  content           TEXT                  -- 結構化建議（見 §6.3 輸出格式）
  differentials     JSONB                 -- 鑑別診斷清單與機率
  status            ENUM       PENDING_REVIEW / APPROVED / REVISED / REJECTED
  reviewed_by       UUID       NULL
  reviewed_at       TIMESTAMPTZ NULL
  revision_note     TEXT       NULL       -- 獸醫修正說明
  revision_of       UUID       NULL       -- 指向前一版
  published_at      TIMESTAMPTZ NULL

knowledge_docs
  doc_id            UUID       PK
  title, authors, journal, year, doi      TEXT
  source_uri        TEXT
  license_note      TEXT                  -- 🔴 授權範圍，見 §9.2
  ingested_at       TIMESTAMPTZ

knowledge_chunks
  chunk_id          UUID       PK
  doc_id            UUID
  section           TEXT
  content           TEXT
  embedding         VECTOR(768)           -- PGVector
  token_count       INT
```

### 6.3 推論流程（P2／P3／P5）

```
① 獸醫建立 Case，上傳影像、填主訴與理學所見
        │
        v
② 輸入品質檢核
   ├─ 影像可讀性、解析度下限、是否為預期部位
   ├─ 必填臨床欄位是否齊備
   └─ 不通過 → ABSTAINED (POOR_INPUT_QUALITY)，流程結束
        │
        v
③ 向量檢索（RAG）
   ├─ 以主訴 + 品種 + 年齡 + 影像所見組成查詢
   ├─ PGVector 相似度檢索，⚙️ top_k 預設 8、相似度門檻預設 0.72
   ├─ 命中不足 ⚙️ 最少 3 段 → ABSTAINED (INSUFFICIENT_EVIDENCE)
   └─ 命中段落全文寫入 inference_citations.snapshot_text
        │
        v
④ Vertex AI Gemini Multimodal 推論
   ├─ 輸入：影像 + 結構化臨床資料 + 檢索到的文獻段落
   ├─ 輸出：結構化 JSON（見下）
   └─ 要求每條建議標註其依據的引用編號
        │
        v
⑤ 輸出後檢核
   ├─ 有無引用？無引用之建議整條丟棄
   ├─ 信心值 < ⚙️ 門檻（預設 0.6）→ ABSTAINED (LOW_CONFIDENCE)
   └─ 全數丟棄後無內容 → ABSTAINED (INSUFFICIENT_EVIDENCE)
        │
        v
⑥ 寫入 second_opinions (status = PENDING_REVIEW)
   進入獸醫審閱佇列。**此時飼主端看不到任何內容。**
```

**結構化輸出格式**

```json
{
  "summary": "影像顯示右前肢…",
  "differentials": [
    { "condition": "…", "probability": 0.62, "citations": [1, 3] },
    { "condition": "…", "probability": 0.21, "citations": [2] }
  ],
  "recommended_next_steps": [ { "action": "…", "citations": [1] } ],
  "limitations": "本建議基於單張 X 光與有限臨床資訊…",
  "citations": [
    { "n": 1, "title": "…", "journal": "…", "year": 2024, "section": "Discussion" }
  ]
}
```

**可追溯性（P5）**：`inference_runs` + `inference_citations` 兩表使任一份 Second Opinion 都能回答「當時用了哪個模型版本、哪個 Prompt、檢索到哪些原文段落、輸入影像是哪一張」。語料日後更新或下架，不影響既有紀錄的可還原性。

### 6.4 Abstention 機制 —「或不產出」（P4）

貴司明示 Second Opinion 應「產出有限、可引用、可審閱、可追溯的建議，**或不產出**」。棄權在本系統是**正式的第一級輸出**，不是錯誤。

| 棄權原因 | 判定條件 | 對獸醫顯示 | 對飼主顯示 |
|---------|---------|-----------|-----------|
| `POOR_INPUT_QUALITY` | 影像不可讀／必填臨床欄位缺漏 | 「輸入資料不足以分析」+ 具體缺什麼 | 不呈現 AI 區塊，標示未產出 |
| `INSUFFICIENT_EVIDENCE` | 檢索命中段落 < 門檻，或全數建議無引用 | 「知識庫中無足夠文獻支持」 | 同上 |
| `LOW_CONFIDENCE` | 模型信心值低於門檻 | 「信心不足，不提供建議」 | 同上 |
| `OUT_OF_SCOPE` | 非 V1 支援之專科或品種 | 「本案超出目前支援範圍」 | 同上 |

**設計要點**

- 棄權同樣寫入 `inference_runs`，計入棄權率指標（§8.2）。棄權率是本引擎的品質指標之一：過低代表門檻太鬆、過高代表知識庫覆蓋不足
- 棄權時**明確告知原因**，不以空白或轉圈掩飾
- 獸醫可於補充資料後重新送出，產生新的 `inference_run`（不覆寫舊紀錄）
- ⚙️ 三項門檻（檢索段落數、相似度、信心值）皆為 Config，可依實測調整

### 6.5 獸醫審閱工作流（P8）

```
PENDING_REVIEW ──獸醫 Approve──────> APPROVED ──> 發布事件 second_opinion.approved
      │                                              └─> 引擎 ① 建立 health_report
      ├──獸醫修正內容後 Approve──> REVISED（保留原版於 revision_of）
      └──獸醫判定不適用────────> REJECTED（不呈現給飼主，紀錄保留）
```

| 規則 | 說明 |
|------|------|
| **未 Approve 不外流** | `PENDING_REVIEW` 與 `REJECTED` 之內容，飼主端一律不可見 |
| **修正留痕** | 獸醫修正時保留 AI 原始輸出與修正後版本，並記錄修正說明 |
| **署名** | 飼主端呈現時標示「經主治獸醫 {姓名} 於 {時間} 認可」 |
| **審閱亦入 Audit** | Approve／修正／退回皆寫入共用層 `audit_events` |
| **上線初期** | 建議首月 100% 人工審閱，不開放任何自動發布（報價單 §八） |

### 6.6 專科與品種範圍（P9）

| 項目 | V1 實作 | 說明 |
|------|--------|------|
| 專科 | **1 個完整 vertical slice**，UI 保留其餘 4 個 placeholder | 🔴 哪一個專科待貴司指定（§9.1） |
| 品種 | **3 個犬種** | BP 規劃之 9 犬 3 貓，其餘待語料與標註到位後擴充 |
| 影像 | JPEG／PNG 之 X 光 | DICOM、CT、超音波屬 Phase 3 |

placeholder 專科於 UI 可見但不可選，點擊顯示「即將推出」，使 Demo 與投資人簡報得以呈現完整產品輪廓，同時不誤導使用者以為已可使用。

### 6.7 獸醫端 Dashboard（P10）

- 案件工作佇列：待送出／分析中／待審閱／已完成，預設顯示本院所本人之案件
- 案件詳情：影像檢視（Cornerstone.js，支援縮放、視窗調整、測距）、臨床資料、AI 建議與引用清單（可展開原文段落）
- 審閱介面：Approve／修正／退回三按鈕，修正為就地編輯
- 多租戶隔離以 PostgreSQL **Row-Level Security** 實施，於資料庫層保證院所間不可互見，非僅靠應用層過濾

### 6.8 臨床評測（P11）

- 建立 ⚙️ 至少 50 例之評測集，涵蓋正常、典型異常、邊界與應棄權案例
- 由貴司指定之獸醫進行人工複核，記錄：建議是否合理、引用是否切題、棄權是否恰當
- 評測結果作為門檻參數（§6.4）調校依據
- **評測不等同臨床驗證**；正式臨床效度驗證屬研究範疇，不在本次開發範圍

---

## 七、端到端流程（跨引擎）

以下五條流程串起四層，是整案的驗收主軸。

### 7.1 入會與訂閱 → 發點

```
飼主 註冊(Clerk) ─> 同意條款(S5) ─> 建立 Pet Profile(S3) ─> 選訂閱方案(E1)
   ─> 綠界定期定額扣款(S11) ─> subscription.activated 事件(S9)
   ─> 引擎② 依方案發放 VetPoints(EARN, 設 expires_at)
   ─> 會員中心顯示餘額與權益
```

### 7.2 搜尋院所 → 預約 → 到院

```
飼主 搜尋院所(E4, 資料源 S4) ─> 送出預約請求(E5)
   ─> appointment.requested ─> 簡訊通知院所(S7)
   ─> 院所受理 ─> appointment.accepted ─> 確認簡訊給飼主(S7)
   ─> 到院看診
```

### 7.3 看診 → 核銷折抵

```
櫃檯 查詢會員(雙認證) ─> 輸入診療費與折抵點數 ─> Quote + 保留(W3)
   ─> 飼主收到確認簡訊/PWA 確認 ─> Confirm 原子交易(W4)
   ─> Ledger + Redemption + Payable 三表同時成立
   ─> points.redeemed 事件 ─> 飼主端錢包即時更新
```

### 7.4 影像 → Second Opinion → 飼主查閱

```
獸醫 建立 Case、上傳影像(P7 → S8) ─> 推論(P3) 或 棄權(P4)
   ─> second_opinions PENDING_REVIEW ─> 獸醫審閱(P8)
   ─> Approve ─> second_opinion.approved 事件
   ─> 引擎① 建立 health_report 並 PUBLISHED(E6)
   ─> Email 通知飼主(S7) ─> 飼主登入會員中心查閱
        （檢查 Consent + RBAC → 發簽章 URL → 寫入 data_access_logs）
```

**此流程橫跨三個引擎與五個共用層元件，是整案整合風險最高之處**，於 P3 整合驗收階段列為第一優先測試項目。

### 7.5 月結清算

```
月初排程 彙總 clinic_payables ─> statements(DRAFT)
   ─> statement.confirmed 前，院所對帳（有異議走爭議流程）
   ─> MRPET_FINANCE 確認 ─> CONFIRMED（明細鎖定）
   ─> 人工匯款 ─> 登記匯款單號 ─> PAID
   ─> 對帳表 Email 給院所(S7)
```

---

## 八、非功能需求

### 8.1 效能

| 項目 | 目標 |
|------|------|
| 共用層 API 回應（p95） | < 300 ms |
| 核銷 Confirm 交易（p95） | < 800 ms |
| 院所端會員查詢（p95） | < 500 ms |
| 院所搜尋（含地圖）首屏 | < 2 s |
| AI 推論端到端（p95） | < 30 s（含檢索與生成；超時顯示進行中，非阻塞畫面） |
| 月結彙總（500 院所 × 1 萬筆） | < 10 分鐘 |

### 8.2 監控指標

| 類別 | 指標 |
|------|------|
| 可用性 | 各服務 uptime、健康檢查 |
| 金融正確性 | **每日 Ledger 重算 vs. wallet.balance 差異數（目標恆為 0）**、保留逾期未釋放數、冪等衝突次數 |
| 事件 | Pub/Sub 積壓量、死信數、重放次數 |
| 通知 | 簡訊送達率、Email bounce 率、重送次數 |
| AI | 推論成功／棄權／錯誤比率、各棄權原因分布、平均引用數、平均延遲、獸醫修正率 |
| 業務 | 日核銷筆數與金額、訂閱轉換與流失、預約受理率 |

> **獸醫修正率**（Approve 前被修改的比例）是 AI 品質最直接的指標，建議貴司列為長期追蹤項目。

### 8.3 告警

| 等級 | 條件 | 通報 |
|------|------|------|
| P1 立即 | Ledger 與 balance 出現差異；Confirm 交易失敗率 > 1%；服務不可用 | 電話 + 訊息 |
| P2 一小時內 | Pub/Sub 積壓 > ⚙️1000；簡訊送達率 < 90%；AI 錯誤率 > 10% | 訊息 |
| P3 次日 | 防弊告警產生；保留逾期異常增加；棄權率單日變動 > 20% | Email |

### 8.4 可用性與備援

| 項目 | 規格 |
|------|------|
| 資料庫 | Cloud SQL **高可用（HA）**，跨區備援 |
| 備份 | 自動每日備份 + **PITR 時間點還原**，保留 ⚙️ 30 天 |
| RPO | ≤ 5 分鐘（金融資料） |
| RTO | ≤ 4 小時 |
| 服務 | Cloud Run 多實例，滾動更新不中斷 |

### 8.5 安全

| 面向 | 措施 |
|------|------|
| 傳輸 | 全站 HTTPS（TLS 1.2+），HSTS |
| 靜態加密 | Cloud SQL 與 GCS 預設加密；醫療影像 bucket 採 CMEK |
| 存取控制 | 最小權限 IAM；醫療 bucket 僅引擎 ③ 與共用層 service account 可讀 |
| 機敏欄位 | 清算帳戶、手機號碼於 API 回應預設遮罩，需特定權限方可完整讀取 |
| 日誌 | **醫療內容與個資不入日誌**，僅記識別碼與 trace_id |
| WordPress 加固 | WAF、外掛白名單、自動更新、每月安全檢查（醫療資料本就不在 WP） |
| 速率限制 | 認證端點與查詢端點分別限流，防列舉攻擊 |
| 滲透測試 | 不在本次範圍，可另行安排第三方（報價單 §6.2） |

### 8.6 相容性

- 瀏覽器：Chrome／Safari／Edge 最新兩個版本；iOS Safari 15+、Android Chrome 最新兩版
- 飼主端與院所端為 RWD，於手機、平板、桌機均可操作
- 院所端考量櫃檯實務，主要流程支援鍵盤操作，不強制滑鼠

---
## 九、待貴司確認事項（🔴 彙整）

本文件內所有 🔴 標記彙整於此。與報價單 §五 為同一份清單，此處補充其技術影響。

### 9.1 架構與範圍

| # | 事項 | 出現於 | 技術影響 |
|---|------|--------|---------|
| 1 | **是否採 Thin Platform 策略**（共用層隨引擎長出，非完整先建） | §一 | 決定 Phase 0 交付範圍。若要求完整先建，S5–S12 全數前移，Phase 0 由 4 週延長為 8–10 週 |
| 2 | **WordPress + WooCommerce 是否續用** | §四 | Identity／Consent／Pet Profile 上移共用層後 WP 僅剩商城角色。若改自建前台，E1–E7 的實作方式全面改寫 |
| 3 | **Pitch Demo 時程選項 A／B** | 報價單 §3.4 | 選項 A 需先凍結 S1 契約再讓引擎 ② 搶跑，有整合返工風險 |
| 4 | **線上預約深度**：請求制或時段庫存制 | §4.5 | 時段庫存需新增院所班表、時段容量、衝突檢查等模型（選配 +8 人日） |
| 5 | **引擎 ③ 的 5 個專科為哪 5 個、V1 先做哪 1 個** | §6.2、§6.6 | 決定 Prompt 設計、評測集組成、語料 ingestion 的優先順序 |
| 6 | **Global Pet ID 是否對接農業部寵物晶片號** | §2.3 | 若要對接，需確認外部介接可行性與資料正確性責任歸屬 |

### 9.2 法遵（影響系統定位）

| # | 事項 | 出現於 | 技術影響 |
|---|------|--------|---------|
| 7 | **VetPoints 法律定位**：1 點 = NT$1、可 100% 折抵、現金清算給院所，性質接近儲值。**需法律意見確認是否落入電子支付管理條例** | §五 | 若認定為電子支付，系統定位、資金保管與申報機制均需重新設計 |
| 8 | **折抵的發票與稅務處理**：折抵金額在院所端是否列為銷售額？院所是否須開發票給 Mr.Pet？ | §5.2、§5.5 | 直接決定 `clinic_payables` 與 `statements` 是否需帶稅務欄位與稅額計算 |
| 9 | **AI Second Opinion 的醫療責任歸屬與同意書文字** | §2.5、§6.5 | 決定 `AI_SECOND_OPINION` 同意書內容、免責聲明呈現方式與位置 |
| 10 | **期刊語料授權範圍**：可否用於商業 RAG 檢索？引用呈現的形式限制？可否儲存原文快照？ | §6.2 | 直接決定 `inference_citations.snapshot_text` 能否保留原文——這是可追溯性的基礎 |

> 第 7、9、10 項建議同批委任律師出具意見。第 10 項若不允許儲存原文快照，可追溯性須改以「引用指標 + 版本號」實作，追溯完整度會下降。

### 9.3 引擎 ② 規格待定項

| # | 事項 | 出現於 | 建議做法 |
|---|------|--------|---------|
| 11 | **折抵單位互斥**：規格第八節「100 點起、100 為單位、無上限」與第十節「輸入 800」「允許 100% 折抵」不一致 | §5.3 | 建議：100 點級距、上限 = min(可用點數, Eligible Fee) |
| 12 | **退費回補**：醫療服務退費時是否回補點數？ | §5.4 | 建議回補並沿用原到期日，否則可無限展期 |
| 13 | **Encounter 歸屬**：`encounter_id` 由引擎 ② 或 ③ 產生？ | §5.2 | 引擎 ③ 未上線前需允許 nullable |
| 14 | **飼主無法確認時的備援**：院所可否代操作？ | §5.3 | 此為最易生弊端之環節，需明定風控與稽核規則 |
| 15 | **點數到期的會計處理**：已發出點數是否認列負債？到期是否沖銷？ | §5.2 | 影響財報與 Statement 呈現 |
| 16 | **爭議 SLA**：院所異議須於多久內提出？月結關帳後如何處理？ | §5.4、§5.5 | V1 無 Adjustment 功能，須定人工流程 |
| 17 | **Reservation TTL 值** | §5.3 | 規格建議 5 分鐘，實務可能不足；建議 15 分鐘 + 延長一次 |
| 18 | **多寵物 Wallet 權責告知方式** | §5.2 | 需於院所端與飼主端明確揭露不做額度隔離 |

### 9.4 營運

| # | 事項 | 出現於 | 技術影響 |
|---|------|--------|---------|
| 19 | **簡訊供應商**（三竹／every8d）與費用歸屬、月量估計 | §3.1、§3.5 | 決定 `SmsAdapter` 的實作對象與送達 webhook 格式 |

---

## 十、交付對照

本規格各章節與報價單 WBS、交付階段之對照：

| 規格章節 | WBS | Phase | 里程碑 |
|---------|-----|-------|--------|
| §2.1 架構與 API 契約 | S1 | P0 | 🚩 **API 契約凍結**——三引擎解除依賴 |
| §2.2–§2.4 Identity／Pet ID／Clinic ID | S2–S4 | P0 | 🚩 平台地基交付 |
| §3.4 Config | S10 | P0 | |
| §五 全章（引擎 ②） | W1–W14 | P1 | 🚩 W12 Pitch Demo／W16 VetPoint 上線 |
| §2.5 Consent、§3.1 Notification、§3.5 Adapter | S5、S7、S11 | P1 | |
| §4.1–§4.3、§4.7 訂閱／轉帳／月光盒／Profile | E1–E3、E7 | P1 | 🚩 訂閱商業閉環上線 |
| §六 全章（引擎 ③） | P1–P11 | P2 | 🚩 Second Opinion 上線 |
| §2.6 RBAC/Audit、§3.2 Storage、§3.3 Event、§3.6 O11y | S6、S8、S9、S12 | P2 | |
| §4.4–§4.6 搜尋／預約／報告查閱 | E4–E6 | P2 | 🚩 醫療動線上線 |
| §七 端到端流程、§八 非功能需求 | S14、E8 | P3 | 🚩 **整案驗收上線** |

---

## 十一、名詞對照

| 名詞 | 說明 |
|------|------|
| **VetPoints** | 對外品牌名稱（貴司 BP 用語）。本文件之 Wallet／Ledger 即其帳本實作 |
| **PetPoint Engine** | 早期提案之內部代號，即現在的引擎 ② VetPoint Medical Clearing Engine |
| **Global Pet ID** | 跨系統、跨院所唯一的寵物識別碼（共用層核發） |
| **Token ID** | 醫療端使用的去識別化寵物識別碼，與 Global Pet ID 一對一但不可由醫療端反查 |
| **Encounter** | 一次看診事件，為核銷與病例的共同歸屬單位 |
| **Quote** | 核銷前的報價與點數保留單，尚未實際扣點 |
| **Redemption** | 已確認成立的核銷交易，扣點已寫入 Ledger |
| **Clinic Payable** | 因核銷而產生的院所應收款 |
| **Statement** | 月結單，彙總單一院所單月之應收 |
| **Abstention** | AI 主動棄權，即貴司所指「或不產出」 |
| **Thin Platform** | 共用層只做引擎當下真正需要的能力，隨引擎需求逐步補完的策略 |
| **RAG** | Retrieval-Augmented Generation，先檢索文獻再生成建議，是「可引用」的技術基礎 |
| **PITR** | Point-In-Time Recovery，資料庫時間點還原 |
| **RLS** | Row-Level Security，PostgreSQL 資料列層級權限，用於多租戶隔離 |

---

## 十二、聯絡資訊

| 項目 | 內容 |
|------|------|
| 公司 | 安璣有限公司（統編 25036259） |
| 聯絡人 | 技術長 徐銘宏 Max Hsu |
| 電話 | 0975-387-805 |
| Email | max.hsu@ankey.tech |
| LINE | max.hsu |

---

*本文件為機密文件，僅供貓做科技股份有限公司內部評估使用。*
