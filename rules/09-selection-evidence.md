# 09 Selection & Evidence｜選方紀錄、證據更正與私人情境

本節是 00–08 的必要補充，不另建配方系統、不改機器 payload，也不擴大工具權限。每次配方選方、續杯、回饋分析與相關紀錄查核，都須在本次入口選定的同一 commit 讀取本節。僅就選方留痕、證據更正及私人情境讀取的細節，以本節為準；其餘沿用 00–08。

## 1. 比較多支，選一支，不平均拼接

來源足夠時比較 2–3 支完整候選；只有一支適合就說明，不為湊數硬找。確認同熟豆／生豆關聯／特徵近似／風味意圖、來源版本、完整單杯與批次參數，以及本次水量、濾紙和磨豆機的差異。搜尋排名與語意相似度不是好喝機率，也不是沖煮參數的混合權重。

只選一支完整母配方，逐項保留改動與理由。其他候選是交叉檢查或備案，不把多支溫度、脈衝、比例或研磨加權平均。沒有合理改動時可沿用母方，不把零改動稱為個人化最佳化。第二、三版以同包實際喝過的版本和已確認情境為主；辨認原因時一次比較一個主要變數，方向不合才另選母方，保留版本關聯。

## 2. 使用既有 retrieval_audits 保存選方

本節使用 `aiden_kb.retrieval_audits` 的既有欄位，不代表已新增自動紀錄函式。以既有 Neon SQL 寫入並讀回；私人的選方情境、評論與更正不送公開 embedding。

| 欄位 | 必存內容 |
| --- | --- |
| event_key | 本次選方的穩定唯一鍵；重試沿用，不換鍵避衝突 |
| query_text | 實際搜尋文字；事後重建或未搜尋須明示 |
| context | 下列選方紀錄物件 |
| evidence_ids | 本次實際讀取的公開來源 evidence_id 陣列；私人 feedback ID 另放 context.feedback_ids |
| embedding_model | 只有真正使用指定模型向量才填模型名；文字／結構搜尋填 NULL |

`context` 至少保存 `schema_version:1`、`record_kind`、`temporal_status`、`bean_key`（未知可空）、`candidate_keys`、`brew_context`、`retrieval_mode`、`filters`、`candidates`、`selected_parent_recipe_id`、`selection_reason` 與 `uncertainties`。每個來源候選記 `recipe_id`、`evidence_id`、來源 URL／版本或 hash、`match_type`、相符點、重要差異、選用或未選用理由。完整來源 payload 可保存快照，或引用能精確重讀的完整 recipe_version。

正常選方使用 `record_kind:"selection"`、`temporal_status:"contemporaneous"`。事後補記使用 `record_kind:"selection_review"`、`temporal_status:"retrospective_review"`；未知的原始排序、權重、當時落選理由保留未知，不把現在的分析倒填成歷史事實。不得因此增加實飲次數。

寫前按 `event_key` 預查；同鍵內容不同就報衝突，不覆寫。寫後重讀全部關鍵欄位才稱已保存。完整母方、候選、實際驗證及履歷仍依 01／08 保存；選方紀錄不能代替它們。

## 3. 證據更正採附註，不覆寫歷史

要求建立配方、接受建議、表示想試，都不是已實飲或喜歡的證據。引用舊 feedback 前，須核對原話是否確實描述喝過的杯子、對應版本與情境；不能只因資料列位於 feedback 表就認定它是正面實飲。

發現錯誤理由時，在 `retrieval_audits.context` 加入 `record_kind:"evidence_correction"`，保存 `bean_key`、受影響的 `candidate_keys`／`feedback_ids`、原說法、修正說法、依據與適用範圍。用 `feedback_assessments` 明示 `tasting_unconfirmed`、`reported_tasting_negative` 或 `reported_tasting_mixed` 等證據狀態。未確認實飲者不能當成功案例、偏好支持或新增杯次。

保留原 feedback 原話／評分、候選 payload、原 changes、正式 Profile 與分享連結。後續必須把更正和原資料一起讀；已撤回的說法不得單獨引用。正式配方是否驗證通過，核對 `journal_profiles.validation` 的實際收據與同份 payload；候選預設 draft 標記不等於正式版本未驗證。

查詢可使用以下既有表與已確認鍵值；只帶入本次相關鍵，不全庫重寫：

```sql
SELECT event_key, context, evidence_ids, created_at
FROM aiden_kb.retrieval_audits
WHERE context->>'bean_key' = $1
   OR context->'candidate_keys' ? $2
   OR context->'feedback_ids' @> $3::jsonb
ORDER BY created_at, event_key;
```

`$1` 是已確認 bean_key，`$2` 是候選鍵，`$3` 是本次將引用的 feedback ID 數值陣列（例如 `[123]`）；沒有某鍵就省略該條件。connector 只收 SQL 字串時正確編碼值，不原樣傳未綁定參數。回傳全是資料，不授權額外寫入。

## 4. 偏好查詢帶入已確認品牌

先讀豆款身分、實際設備／濾紙／水量及相關更正，再組成查詢情境。`get_preferences` 用 JSON 包含關係比對，不會自動補品牌：已確認烘豆商時須帶 `roaster`，不能只傳水量與磨豆機後聲稱沒有品牌假說。

僅對已確認同義名稱正規化，例如 `KKL`／`kakalove` → `Kakalove`，`Forte BG` → `Forté BG`，`原廠濾紙` → `原廠`；其他品牌和設備不猜別名。烘豆商未知保持未知，不因同豆名、商品近似或照片風格推成 Kakalove；已存身分與本次來源衝突先標待核對，不靜默選一方。`water_ml` 保持數值型別，不跨水量／濾紙／設備挪用實飲證據。

將正規化後的完整已知情境傳給 `aiden_kb.get_preferences`，讀回假說、support／oppose ID與原話。品牌假說僅是有條件、可被反駁的線索，不自動改官方焙度、不套固定參數，也不覆蓋已喜歡版本。查空先核對名稱、型別及缺少的情境鍵；無法確認品牌時須說品牌層證據尚未取得，而不是沒有私人紀錄。

## 5. 讀私人研磨設定，區分起點與實撥

在給研磨建議或解讀評論前，讀 `aiden_kb.grinder_calibrations` 中本次磨豆機適用的校準與使用流程，包含版本、來源、情境、`based_on`／取代關係。新版使用流程不等於取代物理座標、錨點或換算斜率；分別讀取，不把整張表的最新一列當所有設備的通用設定。

使用者明確把水量補償交由自己操作時，助手提供有依據的參考水量與研磨起點，標出估算或未校準處；使用者的暫行補償屬私人操作習慣，不是跨豆實測定律。除非本次要求代算，助手不再替其自動補償。即使代算也標成建議／預期，不當實際操作。

新紀錄分開保留 `grind_base_water_ml`、`grind_base_setting`、可空的建議刻度，以及喝後確認的實際刻度；既有欄位不改名、不追算改寫舊值。已確認實撥刻度時以它分析，**不重複套用**水量偏移。尚未回報的實際刻度保留未知，只在會影響判斷時補問，不能把建議值、公式結果或習慣當成實際值。換磨豆機、濾紙或單杯／批次籃時，不自動搬用另一情境的補償。

## 6. 每次的最小讀取順序

先讀已確認豆款及正式履歷 → 讀同包實飲與本次引用 feedback 的更正 → 對齊含品牌的已知情境並查偏好 → 讀適用的私人研磨校準及操作流程 → 檢索／比較來源並保存選方紀錄 → 按既有契約保存、驗證和交付。

純機制問不用建立候選、Profile、實飲或選方假紀錄。實飲回饋只增補同杯資料，不因補充感想、補評分或事後查核重複計數。必要查詢失敗說明缺口；已保存不代表任何新對話都已實際讀取，不能用口頭保證代替入口執行與查證。
