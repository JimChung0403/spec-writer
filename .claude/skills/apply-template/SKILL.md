---
name: Apply Template
description: 把解析完的筆記資訊依照模板套出完整 spec 決定是否拆檔
---

# Apply Template

提供 spec-writer 套用模板的指引。輸入：parse-notes 的七類中間資料；輸出：完整 spec 草稿（已結構化，尚未寫檔）。

模板定義見 `.claude/rules/spec-template.md`。

## 執行

### 章節對應

把七類資訊填進 10 章節：

| 模板章節 | 從哪幾類資訊來 |
|---|---|
| 1. Context | 類別 2（需求）+ 類別 3（現況）的綜合敘述 |
| 2. User stories | 類別 2（需求）轉寫成 "As a ... I want ... so that ..." |
| 3. Current state | 類別 3（現況）+ 類別 5 的現有欄位 |
| 4. Requirements | 類別 2（需求）+ 類別 6（邊界/錯誤）逐條 |
| 5. Data mapping | 類別 5（資料表 / 欄位）整理成表格 |
| 6. UI / 畫面行為（選填） | 類別 4（UI）— 筆記無內容就整章省略 |
| 7. API / 介面（選填） | 類別 7（相依）中涉及 API 的部分 — 無就省略 |
| 8. Acceptance criteria | 類別 2 + 類別 6 推導驗收條件 |
| 9. Out of scope | 筆記中明講「不做」的，或推斷的合理範圍界線 |
| 10. Open questions | 所有 `待補充` 項目 + 筆記原文衝突提示 |

### 轉寫原則

- **會議口語 → 正式**：「客戶說希望可以自己退款」→「使用者應能在訂單詳情頁發起退款申請」
- **不加需求**：筆記沒提到的功能不要補
- **邊界化明**：每條 Requirement 寫成 `觸發條件 → 預期行為 → 錯誤 / 邊界處理` 三段

### Requirements 章節格式

每個 requirement 照這個結構：

```markdown
### R{n}. {需求標題}

**觸發**：{什麼情況下發生}
**預期行為**：{系統應該怎麼反應}
**邊界 / 錯誤**：
- {邊界 1}
- {錯誤 1 + 處理方式}
```

### Data mapping 表格格式

```markdown
### 涉及資料表

| 表名 | 用途 | 變更 |
|---|---|---|
| orders | 訂單主表 | 新增欄位 |
| refunds | 退款記錄 | 新表 |

### 欄位對應表

| 畫面欄位 | DB 表.欄位 | 型別 | 約束 | 說明 |
|---|---|---|---|---|
| 退款原因 | refunds.reason | VARCHAR2(500) | NOT NULL | 使用者輸入 |
| 退款金額 | refunds.amount | NUMBER(10,2) | NOT NULL, CHECK (amount > 0) | 不可超過原訂單金額 |

### 新增 / 修改欄位

**新增**（Oracle SQL）：
- `orders.refund_status VARCHAR2(20) DEFAULT 'none' NOT NULL` + `CHECK (refund_status IN ('none','pending','refunded','rejected'))`

**修改**：無
```

### Mermaid 使用時機

- 流程非線性（有分支 / 迴圈）→ `flowchart`
- 狀態變化明顯 → `stateDiagram-v2`
- 資料表多且有關聯 → `erDiagram`
- 線性流程（A→B→C）→ **不用** Mermaid，條列就夠

### 待補充標記

筆記沒提到的內容寫：

```markdown
待補充：{具體缺什麼}
```

**不要寫** `TBD`、`待確認`、`TODO`。一律用 `待補充：{原因}`。

### 拆檔判斷

全部章節填好後估算總行數：

- < 500 行 → 單檔 `spec.md`
- ≥ 500 行 → 拆成：
  - `spec.md`（章節 1-4, 8-10）+ 指向子檔的連結
  - `data-model.md`（章節 5）
  - `ui.md`（章節 6，若內容 ≥ 100 行）
  - `api.md`（章節 7，若內容 ≥ 100 行）

主檔 `spec.md` 用 Markdown 連結指向子檔：

```markdown
## 5. Data mapping

詳見 [data-model.md](./data-model.md)
```

## Examples

### Normal：單檔

**輸入**：退款功能筆記，七類都有內容，估算 300 行。

**輸出**：單一 `spec.md`，10 章節，每章滿，其中 3 章用了表格、1 章用了 Mermaid flowchart。

### Edge：拆檔

**輸入**：訂單系統筆記，涵蓋列表 / 詳情 / 新增 / 編輯 / 刪除 5 個子功能，資料表 8 張，估算 1200 行。

**輸出**：
- `spec.md`（主文 350 行）
- `data-model.md`（資料 400 行，8 張表 + 欄位對應）
- `ui.md`（畫面 300 行，5 個子功能畫面）
- `api.md`（150 行，12 個 endpoint）

主檔各章對應位置放連結。

### Rejection：章節大量 `待補充`

**輸入**：筆記只涵蓋類別 1、2、3，類別 4-7 全無。

**輸出**：章節 1-3 有內容，章節 4-10 全寫 `待補充：{原因}`。寫完交給 validate-spec 判斷是否比例過高（> 50% 空白 → 建議 spec-writer 回 `INSUFFICIENT_DATA` 而不是寫半成品 spec）。
