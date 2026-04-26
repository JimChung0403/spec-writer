---
name: Readiness Review
description: 從工程師開工視角審查 spec 找出沒定案會卡開工的項目並分級告知使用者
---

# Readiness Review

提供 spec-writer Phase 7 的指引。輸入：已寫入磁碟的 spec.md（與拆檔產物）；輸出：給使用者的就緒度報告。

不同於 validate-spec 檢查「結構合規」，readiness-review 檢查「工程師拿這份 spec 能不能開工」。

## 執行

### 1. 掃描所有缺漏項目

從 spec.md 章節 10 取得所有 `偵測不到此內容` 與 `待補充` 條目（含彙整自其他章節的）。同時逐章再掃一次，確認沒有漏網之魚（章節 10 與內文應該對齊，但以實際內容為準）。

### 2. 對每一項判定影響等級

依「不解決時工程師會發生什麼事」決定等級：

| 等級 | 判定條件 | 例子 |
|---|---|---|
| **BLOCKER** | 不解就完全無法開工，或開工後產出一定要重做 | 認證方式未定 → 整個 API 無法實作；資料表主鍵 / 外鍵未定 → 無法建表；核心 UI 互動行為未定 → 前端無法做頁面 |
| **HIGH** | 可先做但有重大不確定性，可能要大改 | 鎖定參數（5 次 / 30 分鐘）未業務同意 → 邏輯做了數值要重調；演算法未定 → 可先佔位但要重做 |
| **MEDIUM** | 影響範圍小、可佔位推進、後期修改成本低 | 錯誤訊息文案；email 模板；UI 微調 |
| **LOW** | 不影響開工、屬於補強或加分項 | audit log；圖表美化；附加報表 |

### 3. 判定原則（避免誤判）

- 影響「資料層」未定 → BLOCKER（DB schema 改變成本高）
- 影響「介面合約」未定（API 路徑、請求 / 回應格式、認證）→ BLOCKER
- 影響「核心商業邏輯」分支未定 → BLOCKER（例如「鎖定後是否計入 failed_login_count」這類邏輯分支）
- 影響「業務參數」未定 → HIGH（數值可調整，邏輯不變）
- 影響「文案 / UI 細節」未定 → MEDIUM
- 影響「附加功能 / 改善項」未定 → LOW

不確定時往**較高**等級分類（保守原則：寧可讓使用者多看到一個 BLOCKER 也不要少看一個）。

### 4. 額外風險掃描（不在缺漏清單但可能卡關）

除了 `偵測不到` 與 `待補充`，再額外檢查下列「沉默風險」：

- **跨章節依賴未明**：章節 4 R 引用了章節 5 沒列的欄位
- **狀態機缺端點**：章節 6 狀態圖有狀態但無進入 / 退出條件
- **AC 與 R 不對齊**：某 R 沒有對應的 AC，或 AC 引用的條件 R 沒寫
- **Out of scope 與 Requirements 衝突**：章節 9 排除了某項，但章節 4 又依賴它

每發現一個沉默風險，按「3. 判定原則」分級，加入報告。

### 5. 輸出報告

格式（給使用者看，繁體中文）：

```markdown
# 工程師開工就緒度 Review

spec：specs/{feature-name}/spec.md

## 結論

{以下擇一}
- **READY**：無 BLOCKER、HIGH ≤ 2，可以排工程師開工
- **READY WITH CAVEATS**：無 BLOCKER 但 HIGH ≥ 3，建議先解 HIGH 項目再開工
- **NOT READY**：有 BLOCKER {N} 項，工程師無法開工，請先補資訊

## BLOCKER（{N} 項，不解無法開工）

| # | 項目 | 位置 | 為什麼會卡 |
|---|---|---|---|
| 1 | {項目名稱} | {章節 / R 編號} | {一句說明會卡在哪} |

## HIGH（{N} 項，可先做但有重大不確定性）

| # | 項目 | 位置 | 影響 |
|---|---|---|---|

## MEDIUM（{N} 項，可佔位推進）

| # | 項目 | 位置 |
|---|---|---|

## LOW（{N} 項，補強項）

| # | 項目 | 位置 |
|---|---|---|

## 建議下一步

1. 先解 BLOCKER：{列具體要做什麼，找誰，例：「向 PM Joy 確認鎖定參數最終值」}
2. 解完更新筆記 → 重跑 /boss notes/{feature}/
3. HIGH 項目可在 sprint planning 前釐清
4. MEDIUM / LOW 可在實作中漸進補完
```

### 6. 不要做

- 不要修改 spec.md 內容（review 是 read-only 階段）
- 不要把 BLOCKER 項目「降級」成 HIGH 來讓報告好看（誠實第一）
- 不要對 HIGH / BLOCKER 項目給「你可以這樣做」的暗示性建議（屬於推論）
- 不要省略 LOW 項目（即使無傷大雅也要列，使用者有權看完整缺漏地圖）

## Examples

### Normal：READY（少量 LOW）

**輸入**：spec 章節 10 列 1 項待補充（「audit log 是否記錄」）

**判定**：LOW（屬於改善項）

**報告**：
```
結論：READY
BLOCKER：0、HIGH：0、MEDIUM：0、LOW：1
LOW: audit log 詳細欄位（可在實作後期補）
建議：可直接開工
```

### Edge：READY WITH CAVEATS

**輸入**：spec 章節 10 列 5 項，包括「鎖定參數 5 次/30 分鐘 待補充業務同意」「reset 連結被偷對策 偵測不到此內容」「驗證碼是否加 偵測不到此內容」「密碼強度規則 待補充」「email 模板 待補充」

**判定**：
- 鎖定參數 → HIGH（業務數值可改，邏輯影響小）
- reset 連結被偷 → HIGH（影響整體安全策略，可能要回頭加欄位）
- 驗證碼 → HIGH（可能要新元件、改 UI）
- 密碼強度 → MEDIUM（可佔位）
- email 模板 → MEDIUM（文案）

**報告**：
```
結論：READY WITH CAVEATS
BLOCKER：0、HIGH：3、MEDIUM：2、LOW：0
建議：先解 HIGH 三項再進 sprint planning，否則開工後可能要大改
```

### Rejection：NOT READY

**輸入**：spec 章節 7 全章 `偵測不到此內容` + 章節 5 缺主鍵定義 + 章節 4 R5 「驗證 token」邏輯未明

**判定**：
- API endpoint / method / 認證 全未定 → BLOCKER（後端無法實作）
- 主鍵未定 → BLOCKER（建不了表）
- token 驗證邏輯未明 → BLOCKER（核心商業邏輯）

**報告**：
```
結論：NOT READY
BLOCKER：3、HIGH：0、MEDIUM：0、LOW：0

BLOCKER:
1. API endpoint / method / 認證方式 — 章節 7 — 後端完全無法實作
2. password_reset_tokens 主鍵與外鍵未定 — 章節 5 — DBA 無法建表
3. token 驗證邏輯（過期 + 已用 + hash 比對順序）未明 — 章節 4 R5 — 核心邏輯未定

建議：請先與 PM、DBA、後端 lead 確認上述 3 項，才有辦法排工程師開工。
```
