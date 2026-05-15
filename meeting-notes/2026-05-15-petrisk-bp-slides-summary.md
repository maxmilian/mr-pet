# PetRisk.ai × Mr.Pet — BP 投影片資訊整理

> 來源：客戶分享的 6 張 BP 投影片（見 `assets/bp-01 ~ bp-06.jpg`）
> 整理日期：2026-05-15
> 用途：彙整目前 BP 對外揭露的產品 / 技術 / 商業模式資訊，供後續 proposal、tech spec、估時依據

---

## 1. PetRisk.ai 核心技術架構（五層堆疊）

| 層級 | 中文名 | 英文名 | 內含 |
|---|---|---|---|
| L5 | 臨床流程整合 | Clinical Workflow Layer | 工作流嵌合 (PMS Integration)、健檢/慢病追蹤 (Health Checks)、提升醫院營收 |
| L4 | 預測與建議層 | Prediction & Recommendation | 疾病風險分數 (Diseases)、風險因子解釋、下一步處置建議 (Next-best-Actions) |
| L3 | 多模態核心引擎 | Multimodal Engine | 結構化/文本/時間序列融合 (Mixture of Models)、Clinical Fusion Engine |
| L2 | 臨床特徵工程 | Clinical Feature Layer | 品種校正 Z-Scores (Breed-adjusted)、時間序列趨勢、症狀語意嵌入 |
| L1 | 資料擷取層 | Data Acquisition | 多源資料、標準化 PetRisk Canonical Schema、**First Data Moat** |

**Data → Model → Outcome → Continuous Improvement** 飛輪閉環。

---

## 2. 技術堆疊細節（採用多模態 AI 架構）

### Pipeline
```
Veterinary Clinics (X-ray / US / CT / ECG, via OCR & SaaS)
   ↓
Data Ingestion Layer (DICOM / Secure Upload, FastAPI + GCP IAM)  ← 醫療級安全管理系統
   ↓
┌─ GCS Storage (Images, ECG files)  ← 影像資料
└─ BigQuery DB (Labs, Metadata)     ← 結構化數據
   ↓
Data Curation / Cleaning / Annotation (CVAT)
   ↓
Data Pre-process (OpenCV / MONAI / Pydicom / wfdb)
   ↓
Feature Engineering (Python / Pandas / Scikit-learn)  ← 萃取特徵 / 生成特徵
   ↓
AI Model Training (PyTorch / T-Flow / MONAI, Vertex AI)
   ↓
┌─ Knowledge Engine (LLM + KnowledgeGraph, Neo4j / LangChain)
└─ Multimodal AI Engine (Image + ECG + Data AI, PyTorch Models)
   ↓
PetRisk.AI Core (Multimodal + Evidence AI)
   ↓
┌─ Vet Dashboard (React + Web)
└─ Pet Owner App (Flutter for Mobile)
```

### Continuous Learning 來源
- **全球獸醫期刊文獻獲取**：PubMed / MEDLINE、CABI、Scopus、IVIS、Wiley Online Library、AVMA Journals（數以萬計期刊論文）
- **台灣大學獸醫所**：專家篩選、賦能、標註
- **AI 工程團隊 × 台大獸醫所** 雙邊協作

### Data Cleaning
針對犬貓 X 光影像做 Denoised 深度降噪 & Edge-Detected 邊緣提取清洗。

---

## 3. 雙產品定位與定價

| 產品 | 對象 | 定位 | 月費 |
|---|---|---|---|
| **PetRisk.ai** | 合作獸醫端（B2B） | 犬貓醫療輔助診斷服務 | NT$ 39 / 3,000 元（兩階定價，推測 39 為入口/試用、3000 為正式版） |
| **Mr.Pet（派特先生）** | 飼主端（D2C） | 訂閱制產品服務（含內容、健康管理、訂閱盒） | NT$ 598 / 1,598 / 2,989 元（三階訂閱） |

### 產品介面對應
- **PetRisk.ai 獸醫端**：Web Dashboard，含病歷檢視、ECG 波形、ASCVD Risk Estimator（10-Year ASCVD Risk 11.4%、Lifetime ASCVD Risk 69%、PAC 78.2% 等示意數據）、AI 輔助判讀區塊
- **Mr.Pet 飼主端**：Mobile App，毛孩主頁、預約看診、健康追蹤、訂閱盒、AI 輔助判讀（飼主可看懂版本）

### Slogan
> 「**聽不太懂的獸醫話 ➤ 看得懂的決策依據**」
> PetRisk.ai（獸醫端 B2B）→ Mr.Pet（飼主端）的資訊轉譯。

---

## 4. 雙邊商業模式：價值創造飛輪

中央以 **VetPoints Rewards** 連結 Mr.Pet（飼主端，暖色）與 PetRisk.ai（獸醫端，藍色），形成無限循環 ∞。

### 飼主端價值
- +世界級獸醫建議
- +更好醫療品質
- +人寵安全感
- +獲老齡醫療險機會
- +精喜訂閱盒

### 合作獸醫端價值
- +客源 & 客單增長
- +醫療品質提升
- +溝通效率提升
- +不損獲利 %
- +資料去識別化

---

## 5. 競爭分析（2×2 象限）

| 軸 | Y 軸：獸醫療輔助判斷 ↔ 消費娛樂導向 | X 軸：單一領域方案 ↔ 多領域健康管理 |
|---|---|---|

| 象限 | 玩家 |
|---|---|
| 左上（單領域 + 醫療輔助） | SignalPET、Zoetis VETSCAN、NxVET |
| 左下（單領域 + 消費娛樂） | ruff.box、BarkBox、Chewy Goody Box |
| **右上（多領域 + 醫療輔助＋消費）** | **Mr.Pet × PetRisk.AI**（獨佔象限定位） |

---

## 6. MVP 範圍（口頭確認）

- **5 個專科模組**
- **9 個犬種**
- **3 個貓種**
- 合計：12 個 breed-adjusted × 5 專科 = 60 組 sub-model 對應關係

> 註：BP 投影片附註「*RD 發展細節請見 BP」，本份文件僅整理投影片可見資訊，完整 RD 計畫須索取 BP 全文。

---

## 7. 從投影片可推論的關鍵假設與風險點

### 技術可行性訊號（正面）
- 已選定具體技術棧（PyTorch、MONAI、Vertex AI、Neo4j、LangChain、CVAT、FastAPI、GCP IAM）—非空談
- 有台大獸醫所專家標註管道—解決醫療 AI 最稀缺的「高品質標註資料」
- 採用 DICOM 標準 + Canonical Schema—醫院 PMS 整合有路徑
- Knowledge Engine 走 LLM + Knowledge Graph 雙軌—具備可解釋性

### 待釐清/風險（投影片未明示）
1. **臨床驗證計畫**：AUC / Sensitivity / Specificity 目標值？驗證集規模？
2. **PMS 整合範圍**：要嵌入哪些既有獸醫院系統？客製化深度？
3. **法規路徑**：是否定位為「醫療器材」？台灣 / 海外監管策略？
4. **資料 DPA**：與獸醫院的資料處理協議結構（去識別化保證、所有權）
5. **39 vs 3000 元定價邏輯**：是試用 → 正式？還是基礎 → 進階？綁定服務內容差異？
6. **Mr.Pet 訂閱盒物流**：3 階訂閱（598/1598/2989）內容物與毛利結構未揭露
7. **VetPoints Rewards 經濟模型**：點數發放機制、結算方？是補貼還是淨價值轉移？

---

## 8. 後續文件 / 行動清單

- [ ] 索取完整 BP 全文（RD 發展細節、財務預測、團隊組成）
- [ ] 確認 MVP 5 專科具體為哪 5 個（從 ECG 示意推測心臟科為其一）
- [ ] 確認 9 犬 3 貓品種清單
- [ ] 取得臨床驗證計畫 / 與台大獸醫所 MoU 內容
- [ ] 釐清 PetRisk.ai 39 / 3000 元的服務內容差異
- [ ] 釐清 Mr.Pet 三階訂閱的具體內容物
- [ ] 將以上資訊更新至 `proposal/2026-05-09-mrpet-petrisk-ai-proposal.md`

---

## 附：投影片檔案對應

| 檔名 | 主題 |
|---|---|
| `assets/bp-01-petrisk-tech-architecture-layers.jpg` | Underlying Magic：PetRisk.ai 核心技術架構（五層堆疊） |
| `assets/bp-02-petrisk-tech-pipeline.jpg` | PetRisk.ai 核心技術架構（完整 pipeline 與技術棧） |
| `assets/bp-03-dual-product-pricing.jpg` | 雙產品定價：PetRisk.ai vs Mr.Pet |
| `assets/bp-04-business-model-flywheel.jpg` | 商業模式：雙邊價值創造（VetPoints Rewards 飛輪） |
| `assets/bp-05-product-underlying-magic.jpg` | 產品 & Underlying Magic：聽不懂的獸醫話 → 看得懂的決策依據 |
| `assets/bp-06-competitive-analysis.jpg` | 競爭分析（2×2 象限） |
