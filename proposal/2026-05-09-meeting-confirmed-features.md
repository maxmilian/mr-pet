# 2026-05-09 會議確認功能清單

> **會議日期**：2026-05-09 18:10 (GMT+8)
> **會議紀錄**：[2026-05-09-petai.md](../meeting-notes/2026-05-09-petai.md)
> **MVP 上線目標**：2026-08 底
> **業務目標**：2026-01 系統可承載同時 150 人（C 端 Mr.Pet）

---

## 一、Mr.Pet 飼主端（電商）

- [ ] 寵物商品上架（用品、食品、體驗券皆視為商品）
- [ ] 綠界金流串接（信用卡 / 超商 / ATM）
- [ ] 綠界電子發票（B2C 自動上傳財政部）
- [ ] PetPoint 紅利點數累點（消費觸發累點）

---

## 二、PetRisk.AI 獸醫端（SaaS）

### 2.1 帳號與工作平台
- [ ] 獸醫登入專屬 Dashboard
- [ ] 多租戶：每個獸醫獨立 workspace，病患資料相互隔離

### 2.2 病患與病歷管理
- [ ] 病患資料夾管理
- [ ] 醫療影像上傳：X 光、腹部超音波、CT

### 2.3 AI Second Opinion
- [ ] Vertex AI 影像分析，輸出機率建議與進一步檢查建議
- [ ] 可解釋性：引用台大獸醫系期刊論文佐證（葉老師 POC 提供論文資料）
- [ ] 獸醫師 Approve / 修正機制
- [ ] Approve 後資料同步寫入飼主端 PetRisk.AI 病歷 Timeline

### 2.4 結帳與點數折抵
- [ ] 顯示飼主 PetPoint 餘額
- [ ] 輸入診療費用後一鍵折抵
- [ ] 金流由獸醫自行收款，平台不經手現金流（僅記錄消費）

### 2.5 跨診所病歷查看（去識別化）
- [ ] 可看到病患在其他獸醫的診斷內容
- [ ] **看不到**是哪位醫師 / 哪間醫院做的診斷

---

## 三、共享機制

- [ ] SSO 單一登入（LINE / Google 等第三方帳號）跨 Mr.Pet 與 PetRisk.AI
- [ ] Token ID 去識別化：飼主端與獸醫端個資皆指向同一組 Token ID
- [ ] 子網域部署：PetRisk.AI 放 `vet.mrpet.tw`，但為獨立系統
- [ ] 資料隔離：電商個資與醫療個資分離（醫療端對應 ISO 27001 等級）

---

## 四、法律文件

- [ ] 隱私條款（Privacy Policy）
- [ ] 服務條款（Terms of Service / EULA）
- [ ] 免責聲明（Second Opinion 為輔助參考，非正式診斷）

---

## 五、雲端費用估算（MVP 階段）

**MVP 月費目標：約 NT$ 20,000 / 月**

| 項目 | 月費（NT$） | 用途 |
|------|------------|------|
| Linode WordPress 主機 | 1,500 ~ 2,500 | Mr.Pet 電商站（WP + MySQL + Redis） |
| GCP Cloud Run × 2 | 1,500 ~ 2,500 | PetRisk API + PetPoint Engine |
| Cloud SQL PostgreSQL | 2,500 ~ 3,500 | 醫療資料庫（獨立 instance + HA） |
| Cloud Storage | 500 ~ 1,000 | 醫療影像加密儲存（< 100GB） |
| Vertex AI Gemini 推論 | 2,000 ~ 4,000 | AI Second Opinion（依使用量） |
| Cloud Logging + Monitoring | 500 ~ 1,000 | 稽核 Log + 告警 |
| Clerk SSO | 0 ~ 800 | Free tier 10K MAU 內免費 |
| Cloudflare CDN + WAF | 0 | Free plan |
| 網域 / SSL | 100 | mrpet.tw + vet.mrpet.tw |
| miniOrange OAuth plugin | 250 | 攤提 $99/年 |
| Email / SMTP（SendGrid） | 500 ~ 1,000 | 訂單通知、點數提醒 |
| 備份儲存 | 500 ~ 1,000 | 每日備份 GCS Coldline |
| **預估合計** | **約 NT$ 20,000** | MVP 階段（< 1500 MAU） |

### 規模成長預估

| 階段 | MAU | 月費 |
|------|-----|------|
| MVP（8 月上線） | < 100 | NT$ 15,000 ~ 20,000 |
| 9 ~ 12 月 | 300 ~ 600 | NT$ 20,000 ~ 28,000 |
| 1 月（150 人同時在線） | 1,500 | NT$ 28,000 ~ 35,000 |

> **說明**：上述為**雲端基礎設施**月費，不含開發費與年度維護費。
