---
name: Validate Spec
description: 檢查 spec 草稿的完整度 驗收標準可驗證性 和 Mermaid 語法
---

# Validate Spec

提供 spec-writer 驗證自身產出的指引。輸入：apply-template 產出的 spec 草稿；輸出：通過 / 發現問題清單。

## 執行

逐項檢查下列 6 項，任何一項 FAIL → 回頭修正，不要寫檔。

### 1. 章節完整性

必要章節：1, 2, 3, 4, 5, 8, 9, 10（八項必有）
選填章節：6, 7（可省略整章）

- 必要章節任一缺失 → FAIL
- 必要章節存在但全章只有 `待補充` → 接受，但計入 "空章節數"

### 2. 待補充比例檢查

計算必要章節中「只有 `待補充` 字樣、無實質內容」的章節數：

- ≥ 5 個空章節 → FAIL，建議 spec-writer 回 `INSUFFICIENT_DATA` 給使用者，不寫檔
- 3-4 個空章節 → PASS but WARN，在回報中明顯標示
- ≤ 2 個空章節 → PASS

### 3. 資料對應表格檢查（章節 5）

欄位對應表必須有這 5 欄：`畫面欄位 | DB 表.欄位 | 型別 | 約束 | 說明`

- 缺欄 → FAIL
- 某列的某格為 `待補充` → PASS（可接受）
- 表格完全沒有、只有散文敘述 → FAIL（改成表格）

### 4. Acceptance criteria 可驗證性（章節 8）

每條驗收條件必須可客觀驗證：

**可接受**：
- Given/When/Then 格式
- 條列式可檢查條件（例如「送出後 3 秒內返回結果」）
- 明確的輸入 / 輸出對照

**不可接受（FAIL）**：
- 「系統應正確運作」
- 「使用者體驗應順暢」
- 「效能要好」
- 「符合業務規則」（未指明哪條規則）

發現不可接受條件 → 標記具體哪一條有問題，回頭改寫。若無法改寫（筆記沒提）→ 改成 `待補充：{具體要什麼驗收條件}`。

### 5. Mermaid 語法

每個 ```mermaid ... ``` 區塊檢查：

- 有無合法的 diagram type（`flowchart`、`stateDiagram-v2`、`erDiagram`、`sequenceDiagram`）
- 括號、箭頭、節點 ID 是否成對
- 節點 ID 是否用到特殊字元（中文、空白）—— 中文要包在 `["..."]` 內，不可裸用

錯誤範例：
```
flowchart TD
  開始 --> 結束    ← FAIL：中文節點 ID 要加 [""]
```

正確：
```
flowchart TD
  A["開始"] --> B["結束"]
```

### 6. Open questions 彙總

掃描全 spec 找 `待補充：` 字樣，確認每一項都出現在 Open questions 章節（章節 10）。

- 有 `待補充` 未彙總進章節 10 → FAIL
- 章節 10 應有「來源筆記清單」子章節，列出所有本次處理的**原始** `notes/` 檔案路徑
- 來源筆記清單包含 `.spec-writer/converted/` 路徑 → FAIL（應列原始檔案如 `mockup.pdf`，不是轉檔結果）

### 驗證報告格式

驗證完產出：

```
Validation Result: PASS / PASS_WITH_WARN / FAIL

Checks:
- [✓] 章節完整性: 10/10 存在
- [✓] 待補充比例: 2/8 空章節（PASS）
- [✓] 資料對應表格: 欄位齊全
- [✓] Acceptance criteria: 全部可驗證
- [✓] Mermaid 語法: 2 個區塊正常
- [✓] Open questions 彙總: 5 項全彙總

Warnings:
- (無)

Actions:
- 可寫檔
```

或 FAIL 範例：

```
Validation Result: FAIL

Checks:
- [✓] 章節完整性: 10/10
- [✗] 待補充比例: 6/8 空章節 — 筆記資訊不足
- [✓] 資料對應表格: N/A（整章待補充）
...

Actions:
- 不寫檔
- spec-writer 應回 INSUFFICIENT_DATA 給使用者
```

## Examples

### Normal：PASS

**輸入**：完整 spec 草稿，10 章節滿，資料表格齊全，acceptance 用 Given/When/Then。

**輸出**：PASS，可寫檔。

### Edge：PASS_WITH_WARN

**輸入**：spec 草稿，9/10 章節有內容（選填的 API 章節省略），必要章節中 3 個空（待補充比例偏高但未超標）。

**輸出**：PASS_WITH_WARN，回報時要明顯標示「3 個章節待補充」，建議使用者補完再重跑。

### Rejection：FAIL

**輸入**：spec 草稿，6 個必要章節只有 `待補充` 字樣。

**輸出**：FAIL，指示 spec-writer 不要寫檔，改回 `INSUFFICIENT_DATA` 給使用者，列出缺什麼資訊。
