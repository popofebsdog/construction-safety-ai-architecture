# 工地現場安全自動判釋系統架構說明

## 1. 系統定位

本系統以工地現場照片與工程元資料為輸入，透過視覺語言模型、營造法規知識圖譜與表單綁定機制，產生安全風險判釋、改善建議與自動填表結果。

系統不直接取代正式職安稽核，而是作為「影像初篩與查核輔助工具」。VLM 產生的照片標註與表單內容應保留人工查核狀態，由職安人員確認是否採納或修正。

## 2. 核心架構概念

整體知識核心稱為 Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜）。Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜）不是單一扁平資料表，而是以法規/內規為核心，串接表單欄位與歷史查驗資料組成，再透過跨圖邊連接成頂層圖譜。

主要節點層如下：

| 層級 | 目的 | 典型節點 |
|---|---|---|
| Regulation Graph | 管理法規條文、層級、適用情境與 Safety Alert（安全警示）閾值 | 法、章、條、項、款、適用條件 |
| Violation Graph | 管理違規類型分類與嚴重等級 | Personal Protective Equipment（PPE，個人防護具）缺失、高處作業缺失、施工架缺失 |
| Improvement Graph | 管理改善措施、期限、驗收方式 | 立即停工、補設護欄、補拍照片複核 |
| Form Field Layer | 管理由法規/內規衍生的查核欄位與填寫規則，是規則落地與歷史資料累積的介面 | 工地安全查核表欄位、鷹架查核表欄位、欄位代碼、必填規則 |
| Historical Inspection Layer | 管理表單欄位被實際填寫後累積出的歷史查驗資料，作為 VLM 注意錨點與機器學習資料來源 | 查驗紀錄、缺失描述、改善前後照片、複核狀態、相似案例 |

其中 Regulation Graph 是核心，Violation 與 Improvement 是法規/內規的判定與處置語意；Form Field Layer 是法規/內規落地後的欄位介面；Historical Inspection Layer 則掛在欄位與規則後面，提供脈絡與學習資料，不直接取代法規判定。

頂層 Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜） 負責維護跨子圖關係：

| 關係 | 說明 |
|---|---|
| triggers | 違規類型觸發哪些法規條文 |
| requires | 法規條文要求哪些現場安全條件 |
| remedied_by | 違規類型對應哪些改善措施 |
| implemented_as | 法規/內規或違規類型落地為哪個表單欄位 |
| has_history | 表單欄位連到哪些歷史查驗紀錄、缺失案例與改善紀錄 |
| overrides | 工地自訂規則、地方規定、國家法規之間的覆蓋關係 |

Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜）可以支援機器學習，但它本身主要是知識結構與推論依據。機器學習通常發生在下列位置：

- 以圖譜節點與邊訓練 graph embedding，提升法規檢索與相似違規推薦。
- 用歷史 Observation（OBS，結構化觀察）、標記框、人工複核結果訓練違規分類模型。
- 用已確認案例訓練嚴重度預測，例如 A/B/C 分級。
- 用 link prediction 協助建議新的 triggers、remedied_by、implemented_as / has_history 關係，再交由人工審核。
- 用 active learning 找出模型最不確定的照片或觀察，優先請稽核人員標註。

因此，Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜）的定位是「可被機器學習利用的知識骨架」。正式法規條文與表單規則仍應保留可追溯、可人工審核的明確規則，不建議完全交給黑箱模型自動生成。

## 3. 上到下處理流程

1. 使用者上傳現場照片與工程元資料。
2. Context Retriever（脈絡檢索器）從法規/內規節點出發，沿著表單欄位連到歷史資料，完成適用規則比對、表單對應與 VLM 判釋錨點建立。
3. Context-Guided Visual Language Model Perception Module（脈絡引導式 VLM 視覺語言模型觀察模組）依注意錨點做全圖判讀，並輸出結構化觀察與疑似區域標記。
4. 人工查核：由職安人員檢視 VLM 標註、照片證據與表單填寫內容，確認是否採納或修正。
5. Output Generator（輸出產生器）產生安全判釋報告、標記後照片、改善建議與表單填寫內容。

## 4. 模組說明

### 4.1 Input Layer

輸入包含照片與工程元資料。照片可為單張或多張；工程元資料包含工地名稱、承包商、拍攝日期、施工階段、工項類型與地點。

此層的重點不是判斷違規，而是建立後續判讀的上下文。例如同一張照片在「模板組立」、「鋼筋綁紮」、「道路施工」下，適用的查核項目會不同。

### 4.2 Context-Guided Visual Language Model Perception Module（脈絡引導式 VLM 視覺語言模型觀察模組）

Visual Language Model（VLM，視覺語言模型）負責把影像內容轉成結構化觀察，而不是直接下法規結論。

若把「危險事件」當成監督式影像辨識類別訓練，會遇到少數事件樣本不足的問題。工地真正危險的事件通常是長尾資料：發生次數少、樣態變化大、標註成本高，因此單獨訓練一個危險事件偵測模型容易成效不穩。

因此建議將原第 2 步與第 3 步整合為「脈絡引導式 VLM 判讀」。此模組不先要求模型學會所有危險事件，而是先用既有資料建立注意錨點，再讓 VLM 依錨點看圖。

注意錨點的來源包含：

- Regulation Graph（法規圖）：提示該工項適用哪些法規條件。
- Form Field Layer（表單欄位層）：由法規/內規節點連到必看欄位，例如安全帽、護欄、施工架踏板。
- Historical Inspection Layer（歷史查驗資料層）：從欄位過去累積的查驗表、缺失紀錄、改善照片與複核結果中找出相似情境常見風險。
- 工程元資料：工項、施工階段、地點、承包商、日期與拍攝任務。

此設計的分工如下：

1. Context Retriever（脈絡檢索器）先產生「這張照片應注意什麼」。
2. VLM 同時做全圖掃描與錨點導向判讀。
3. VLM 必要時輸出 bounding box（bbox，矩形框）或 polygon（多邊形區域）作為證據標記。
4. VLM 產生的標註與表單內容進入人工查核，由職安人員確認是否採納或修正。

這樣的好處是：系統用歷史查驗表與法規圖譜提供脈絡，而不是要求模型從少量危險事件照片中硬學出所有風險型態。少數案例仍可透過 active learning（主動學習）累積，逐步補強模型。

照片標記輸出由 Visual Language Model grounding（視覺語言模型定位）產生，包含疑似違規範圍、違規類型提示與人工查核狀態。標記不是最終法規結論，而是後續 Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜）判定的影像證據。

結構化觀察可包含：

| 欄位 | 說明 |
|---|---|
| observation_id | OBS-001 這類 Observation（OBS，結構化觀察）唯一編號 |
| category_hint | 模型初步判斷的類別 |
| description | 可見事實描述 |
| location | 影像中的位置 |
| linked_annotations | 對應的 BOX/POLY 標記 |
| evidence | 支撐觀察的可見線索 |
| review_status | 待人工查核 / 已查核 / 已修正 |
| review_note | 需人工確認或修正的原因 |

重要原則：Visual Language Model（VLM，視覺語言模型）只負責「看到了什麼」，Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜）與 Threshold Evaluator（閾值判定器） 才負責「這是否違規」。

### 4.3 Category Router（類別路由器）

Category Router（類別路由器）將 Observation（OBS，結構化觀察）分類到對應法規域。分類可採混合策略：

- 規則關鍵詞：安全帽、反光背心、護欄、鷹架、電纜、吊掛。
- 語意嵌入：使用 BGE-M3 multilingual embedding model（多語語意嵌入模型）或其他多語嵌入模型。
- 多標籤分類：同一觀察可同時屬於 Personal Protective Equipment（PPE，個人防護具）與高處作業。

輸出是分組後的觀察集合，例如：

- Personal Protective Equipment（PPE，個人防護具）: OBS-001, OBS-004
- 高處作業: OBS-001, OBS-002
- 施工架: OBS-002, OBS-005

### 4.4 CRKG Knowledge Core（營造法規知識圖譜核心）

Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜）是系統可維護性的核心。它以法規/內規為核心，連接違規類型、改善措施、表單欄位與歷史查驗紀錄，讓系統同時具備規則推論與案例脈絡檢索能力。

這樣設計的好處是：

- 法規更新時，主要更新 Regulation Graph。
- 表單改版時，更新 Form Field Layer，並保留欄位版本與對應法規/內規節點。
- 新增工地內規時，可建立 Site Rule 並用 overrides 邊覆蓋既有規則。
- 同一違規可同時觸發多條法規、改善措施與多份表單。

### 4.5 Multi-Category Retriever（多類別圖譜檢索器）

Retriever（檢索器）的任務是從觀察找到最相關的法規子圖。它不只回傳單一條文，而是回傳一個包含違規類型、條文、改善措施、表單欄位與歷史案例的小子圖。

建議流程：

1. 以類別過濾候選法規域。
2. 以語意嵌入搜尋相關 ViolationType / RegulationArticle。
3. 從命中節點做 BFS 路徑展開。
4. 帶回 triggers、requires、remedied_by、implemented_as、has_history 相關節點。
5. 依 overrides 關係套用更嚴格或更高優先權規則。

### 4.6 Threshold Evaluator（閾值判定器）

Threshold Evaluator（閾值判定器）判定 Observation（OBS，結構化觀察）是否達到 Safety Alert（安全警示）條件。

判定時需同時考量：

- 觀察內容是否明確命中違規條件。
- 工程元資料是否符合該法規適用情境。
- 是否存在更嚴格的地方規定或工地自訂規則。
- 是否需要多張照片交叉驗證。

建議處理規則：

- VLM 標註作為影像證據，不直接作為最終稽核結論。
- 達到規則條件者可先產生疑似違規與建議表單內容。
- 職安人員完成人工查核後，系統才將結果標記為已採納或已修正。

### 4.7 Output Generation Layer（輸出產生層）

輸出包含兩部分：

1. Safety Alert Report（安全警示報告）：安全判釋報告。
2. Form Auto-Filler：表單自動填寫。

報告內容應包含：

- 違規或疑似違規清單。
- 對應照片與 Observation（OBS，結構化觀察）編號。
- 對應照片上的標記範圍、疑似違規類型與人工查核狀態。
- 觸發法規或內規。
- 嚴重等級。
- 改善建議。
- 限期改善日期。
- 人工查核狀態與查核備註。

表單自動填寫依據 Construction Regulation Knowledge Graph（CRKG，營造法規知識圖譜）的 implemented_as 邊運作。Output Generator（輸出產生器）不需要知道所有表單邏輯，只需從法規/內規節點讀取對應 Form Field Layer（表單欄位層）的欄位規則並填入內容；歷史查驗紀錄則透過 has_history 邊提供相似案例與注意錨點。

## 5. 建議資料格式

### Attention Anchor

```json
{
  "id": "ANCHOR-001",
  "photo_id": "IMG-20260528-001",
  "graph_path": ["法規/內規", "表單", "歷史資料"],
  "source_nodes": ["REG-WORK-HEIGHT-001", "FORM-PPE-F03", "HIS-2025-PPE"],
  "work_item": "施工架作業",
  "focus": "高處作業人員是否具備個人防護具與防墜措施",
  "checklist_fields": ["安全帽", "安全帶", "護欄", "踏板完整性"],
  "historical_record_refs": ["HIS-2025-0412", "HIS-2025-1020"],
  "prompt_hint": "請優先檢查施工架平台、臨邊與作業人員頭部/腰部防護。"
}
```

### Photo Annotation

```json
{
  "id": "BOX-001",
  "photo_id": "IMG-20260528-001",
  "anchor_id": "ANCHOR-001",
  "geometry_type": "bbox",
  "coordinates": {
    "x": 0.62,
    "y": 0.18,
    "width": 0.16,
    "height": 0.28
  },
  "object_type": "person",
  "violation_type_hint": "疑似未配戴安全帽",
  "review_status": "待人工查核",
  "visual_style": {
    "label": "疑似未戴安全帽",
    "color": "red"
  }
}
```

### Observation

```json
{
  "id": "OBS-001",
  "photo_id": "IMG-20260528-001",
  "anchor_id": "ANCHOR-001",
  "observation_type": "visual_observation",
  "description": "施工架上有作業人員疑似未配戴安全帽",
  "location": "影像上方與下方施工架作業區",
  "linked_annotations": ["BOX-001", "BOX-003"],
  "category_hint": ["PPE", "高處作業", "施工架"],
  "visual_evidence": ["人員頭部未見安全帽", "人員位於施工架作業區"],
  "decision_scope": "僅為視覺觀察，非最終違規結論",
  "review_status": "待人工查核",
  "review_note": "需由職安人員確認是否確實未配戴安全帽"
}
```

### Violation Decision（違規判定資料）

```json
{
  "id": "VIOLATION-001",
  "source_observations": ["OBS-001"],
  "status": "疑似不符合",
  "severity": "B",
  "matched_rules": ["REG-HEIGHT-PPE-001"],
  "evidence_annotations": ["BOX-001", "BOX-003"],
  "rule_basis": ["法規/內規", "表單: 個人防護具.安全帽.F03", "歷史資料: HIS-2025-PPE"],
  "improvements": ["補拍近景確認安全帽", "確認高處作業 Personal Protective Equipment（PPE，個人防護具）配戴狀態"],
  "form_bindings": [
    {
      "form": "工地安全查核表",
      "field": "個人防護具.安全帽.F03",
      "value": "疑似未配戴安全帽，需人工複核"
    }
  ],
  "requires_human_review": true
}
```

## 6. 風險與控制

| 風險 | 控制方式 |
|---|---|
| Visual Language Model（VLM，視覺語言模型）誤判傳播 | VLM 標註預設進入人工查核，多角度照片交叉驗證 |
| 法規版本錯誤 | Regulation Graph 加入生效日、版本、來源與審核狀態 |
| 表單誤填 | Form Field Layer 欄位加上型別、必填規則、版本與人工確認狀態 |
| 場景不足 | 輸入層要求工程元資料，必要時要求補拍特定角度 |
| 法規責任風險 | 報告標示為初步影像判讀，不作為最終稽核結論 |

## 7. 最適合的系統呈現方式

對外展示建議用網頁版，因為它能同時放架構圖、模組說明與影像標記範例。對正式提交可再轉成報告書或 PPT。

本次產出建議使用：

- `index.html`：展示用網頁。
- `architecture_report.md`：報告書文字稿。
- 後續若要投影片，可將網頁中的每個 section 拆成 1 張 slide。
