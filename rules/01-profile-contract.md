# 01 Profile Contract｜輸入到 Payload（SSOT）

## 輸入門檻（阻斷欄位）
- 必須有焙度（淺／中淺／中／中深／深；中淺中深直接查列，不插值）。
- 必須有處理法；低因／EA／甘蔗與一般水洗分流。
- 單一產地至少國家；混豆可用 `blend` 作身分。
- 烘豆日有就記，缺少不阻斷。

## 原點決策
1. 同包同 key 已收斂紀錄（最高）。
2. Overlay 硬命中（焙度＋處理法同類＋國家／產區＋品種可確認同豆；低因／EA／甘蔗自成一類）。
3. 死表冷啟（底）。
4. Overlay 近命中只列候選；使用者明確選用前不取代死表。

## 參數流水線
`target volume → roast table → altitude/process overlays → single/batch grind routing → grinder conversion → legal snap → payload validation`
每一步保存來源與中間結果；後一步不得重新推翻前一步來源。

## Canonical Profile Payload
必須保存完整 JSON，至少包含：
`schemaVersion, profileTitle, profileType, ratio, bloomEnabled, bloomRatio, bloomDuration, bloomTemperature, ssPulsesEnabled, ssPulsesNumber, ssPulsesInterval, ssPulseTemperatures, batchPulsesEnabled, batchPulsesNumber, batchPulsesInterval, batchPulseTemperatures, singleGrind, batchGrind, originLayer, sourceRevision`

## 最終驗證
- title 符合 Aiden 合法字元（英數＋`!@#$%&*-+?/.,:)(` 空格；`|` 不可用；≤50 字元）。
- ratio／溫度／時間均在合法刻度與範圍。
- pulse 數量等於溫度陣列長度。
- 單杯、批次兩路完整。
- Overview 的原點層級與實際來源一致。
- MCP 建立前必須先通過 validate。

## DB key 與版本
- 主線 key：`bag_id＋目標水量ml＋濾杯＋磨豆機`。
- 主線版本：前一主線最大版本 + 1；實驗列 `is_experiment=true`、記 `parent_version`，不占主版本。
- 同名新包必須新 `bag_id`。
- event_id（deterministic）：主線 `bag_id|water|filter|grinder|main|vN`；實驗 `bag_id|water|filter|grinder|exp|parentV|experimentCode`。同一 event_id 只一列；寫前預查、寫後重讀。
- 每列必填：event_id、bag_id、version_number、parent_version（v1可空，實驗必填）、created_at、咖啡名、水量ml、濾杯、磨豆機、焙度、處理法、產地身分、changed_variable、實泡感想原話、症狀、評分（未明示留空）、profile_title、profile_payload、payload_hash、brewlink（未建可空）、is_converged、is_experiment。

## 不可做
- 不從 archive 取 operational 數字。
- 不因 CIE 不可用而停住。
- 不把近 overlay 自動當原點。
- 已生成且驗證的 payload 在工具交接時不得重新計算。
