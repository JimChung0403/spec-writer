---
name: Parse Notes
description: 掃描筆記資料夾或檔案 必要時用 markitdown 轉檔 萃取 spec 需要的七類資訊
---

# Parse Notes

提供 spec-writer 解析筆記的指引。掃描輸入 → 分類檔案 → 偵測 markitdown → 轉檔 → 萃取關鍵資訊 → 偵測衝突 → 輸出結構化中間資料。

## 執行

### 資料夾掃描

輸入若為資料夾路徑（結尾 `/`、或 `test -d` 為真）：

- 用 Glob：`{folder}/**/*` 列出所有檔案（含子目錄）
- 排除：`.spec-writer/**`（自身快取目錄）、其他以 `.` 開頭的隱藏檔、`README.md`、`.gitignore`、`.DS_Store`
- 結果空 → 回 `INSUFFICIENT_DATA：資料夾 {path} 沒有可分析的檔案`

輸入若為檔案清單：直接照清單處理。混合資料夾與檔案：分別處理後合併清單。

### 檔案分類

依副檔名分三類：

| 類別 | 副檔名 | 處理方式 |
|---|---|---|
| native | `.md` `.txt` `.markdown` | 直接 `Read` |
| convertible | `.pdf` `.docx` `.pptx` `.xlsx` `.html` `.htm` `.csv` `.json` `.xml` `.epub` | 需先用 markitdown 轉成 .md |
| unsupported | 其他（圖片 `.png/.jpg`、音訊 `.mp3/.wav`、影片、壓縮檔等）| 列出告知使用者，不分析 |

### file_list 管理（增量分析的關鍵）

每個 feature 資料夾維護一份 `file-list.md`，記錄已掃過的檔案與轉檔結果。位置：

```
notes/{feature}/.spec-writer/
├── file-list.md              ← 檔案清單（manifest）
└── converted/
    ├── mockup.md             ← markitdown 轉檔結果
    └── requirements.md
```

**file-list.md 格式**（固定模板，spec-writer 用 Read / Edit 維護）：

```markdown
# File List — {feature-name}

最後掃描：YYYY-MM-DD HH:MM
最後成功產 spec：YYYY-MM-DD HH:MM   ← 由 spec-writer Phase 6 寫檔成功後更新

## native（直接讀）
- meeting.md (4321 bytes, mtime 1714032000)
- ui-notes.txt (892 bytes, mtime 1714031900)

## converted（markitdown 結果存於 .spec-writer/converted/）
- mockup.pdf (102400 bytes, mtime 1714030000) → converted/mockup.md
- requirements.docx (45200 bytes, mtime 1714029500) → converted/requirements.md

## unsupported（已忽略）
- screenshot.png (12800 bytes, mtime 1714029000) — 圖片，需 OCR
- demo.mp4 (5200000 bytes, mtime 1714028000) — 影片
- old-spec.pdf (88000 bytes, mtime 1714027500) — markitdown 未安裝且使用者選擇忽略
```

`mtime` 為 Unix epoch 秒數，用 `stat -f "%m" file 2>/dev/null || stat -c "%Y" file` 取得（macOS / Linux 雙相容）。

### 增量分析流程

每次分析一個資料夾，照下列順序：

1. **掃描**：Glob 列出當前所有檔案 → 對每個檔案取得 `size + mtime`：
   ```bash
   stat -f "%z %m" file 2>/dev/null || stat -c "%s %Y" file
   ```
2. **載入既有 file-list**：
   - `notes/{feature}/.spec-writer/file-list.md` 不存在 → 視為全新（所有檔案皆為 new）
   - 存在 → Read 並解析，取得 `previous_files`（含 size + mtime + 分類 + cache 路徑）
3. **差異比對**（key = 檔名相對路徑，hash = `size + mtime`）：
   - **new**：在 current 但不在 previous
   - **changed**：在兩者中但 size 或 mtime 任一不同 → 視為 new（要重新分類與轉檔）
   - **removed**：在 previous 但不在 current → 從 file-list 移除，刪除對應 converted/{x}.md
   - **promote-on-markitdown-install**（M2 修補）：若這次 markitdown 偵測通過（步驟 5），把 file-list 中 reason 為「markitdown 未安裝且使用者選擇忽略」的 unsupported 全部 promote 為 changed，重做分類與轉檔
   - **cache-missing**（C2 修補）：unchanged 的 convertible 額外檢查 `test -f notes/{feature}/.spec-writer/converted/{x}.md`，cache 不存在 → promote 為 changed
   - **unchanged**：name + size + mtime 一致，且 cache 存在（若是 convertible）→ 完全沿用
4. **分類**：對 new / changed / promote 後的檔案套用「檔案分類」表
5. **markitdown 偵測**（只在出現 new convertible 時執行）：見下節。**偵測通過後立即執行步驟 3 的 promote-on-markitdown-install**
6. **markitdown 轉檔**（只對需要重轉的 convertible：new、changed、promoted）：見下節
7. **準備記憶體中的 next-file-list 草稿**：反映當前掃描結果與分類，**但這時候還不寫到磁碟**（C3 修補：file-list 寫入時機交給 spec-writer Phase 6 在 spec 寫檔成功後執行）
8. **回傳清單**：把 native + converted（可讀的）+ next-file-list 草稿一起交給 spec-writer Phase 2 萃取使用

**重要**：
- unchanged 的 native 檔案下一步仍要 Read（內容萃取），但 unchanged 的 convertible 直接讀 `.spec-writer/converted/{x}.md` 即可，**不重新呼叫 markitdown**
- file-list.md 的寫入由 spec-writer Phase 6 統一處理，parse-notes 只準備草稿，**不要在這裡寫**

### markitdown 偵測

只在「new convertible」清單非空時才偵測：

```bash
command -v markitdown
```

- 退出碼 0 → 已安裝。**立即觸發 promote-on-markitdown-install**：掃 file-list 把所有 reason 為「markitdown 未安裝且使用者選擇忽略」的 unsupported 條目改判為 changed（之後在轉檔步驟一併重轉），讓使用者裝好套件後重跑就能自動補上之前略過的檔案
- 退出碼非 0 → 未安裝。**不要嘗試 pip install**。回報給 spec-writer Phase 1 處理（會問使用者是否忽略這些檔案）

### markitdown 轉檔

對每個 new/changed convertible 執行：

```bash
mkdir -p notes/{feature}/.spec-writer/converted
markitdown {source-path} -o notes/{feature}/.spec-writer/converted/{filename}.md
```

`{filename}.md` 用原檔案名換掉副檔名（例 `mockup.pdf` → `mockup.md`）。

轉檔失敗（非零退出碼）→ 把該檔在 file-list 標為 `unsupported`，附原因（例如「markitdown 轉檔失敗：{stderr}」），繼續處理其他檔案。

### 來源追蹤

每份成功讀入的內容記錄：

- **原始檔案路徑**（轉檔的也記原始 .pdf 路徑，不記轉檔結果）
- **是否經過 markitdown 轉檔**（true / false）
- **轉檔結果路徑**（若有）

這份對照後續寫入 spec 章節 10 的「來源筆記」清單，**只列原始檔案**（使用者看到的是他丟進來的原檔，不是 .spec-writer/converted/ 的快取）。

### 讀檔

對 native 檔案用 `Read` 工具讀取原檔。對 converted 檔案用 `Read` 讀 `.spec-writer/converted/{x}.md`。超過 500 行的筆記分段讀，逐段萃取。

### 萃取七類資訊

從筆記中抓以下七類，每類用 bullet list 記下，**保留原文引用**（用來追溯）：

| 類別 | 抓什麼 | 舉例 |
|---|---|---|
| 1. 功能主題 | 這份筆記在講哪個功能 / 模組 | 「退款流程」「登入」「訂單列表」 |
| 2. 使用者需求 | 使用者想做 / 改什麼 | 「希望可以自助退款」「登入要支援手機號碼」 |
| 3. 現況流程 | 目前怎麼運作 | 「目前退款需聯絡客服」「現在只有 email 登入」 |
| 4. UI / 畫面 | 畫面、按鈕、狀態、互動 | 「訂單頁加一個『申請退款』按鈕」「退款狀態：申請中 / 已退款 / 已拒絕」 |
| 5. 資料表 / 欄位 | 表名、欄位名、型別、約束、對應 | 「orders 表加 refund_status 欄位」「金額欄位對應 order_items.price」 |
| 6. 邊界 / 錯誤 | 例外情況、錯誤訊息 | 「若超過 7 天不可退款」「庫存不足要顯示提示」 |
| 7. 相依 / 外部 | 相依系統、第三方 API、背景任務 | 「要打金流退款 API」「會觸發 email 通知」 |

### 偵測衝突

多份筆記讀完後，對每一類檢查資訊是否一致：

- 衝突 → 記錄：「類別 X：筆記 A 說 {...}，筆記 B 說 {...}」
- 無衝突 → 不記錄
- 資訊互補（不衝突、但不重疊）→ 不記錄為衝突，直接合併

### 推斷 feature name

根據類別 1（功能主題）推斷一個 kebab-case 名稱：

- 單一主題明確 → `{主題}`，例如 `refund-flow`、`login-v2`
- 多主題 → 取最主要的一個，或問使用者
- 無法推斷 → 留空，Phase 3 問使用者

### 輸出中間資料

以結構化格式整理（心中草稿，不寫檔）：

```
Feature name 推斷: {名稱 或 "無法推斷"}

七類資訊:
1. 功能主題:
   - {bullet}
2. 使用者需求:
   - {bullet}
...

衝突清單:
- {類別 X}: {筆記 A 說 ... / 筆記 B 說 ...}

原文來源對照:
- 類別 2 > 「希望可以自助退款」 ← notes/refund/meeting.md:12
```

## Examples

### Normal：首次掃描，全為 native

**輸入**：`notes/refund-flow/`，內含 `meeting.md`、`ui-notes.txt`、`data-table.md`

**行為**：file-list 不存在 → 全 3 個視為 new → 分類全為 native → 跳過 markitdown 偵測 → Read 原檔 → 寫入 `.spec-writer/file-list.md` 記錄三個 native 項目。萃取 7 類 → feature name `refund-flow`。

### Edge：含 PDF，markitdown 已裝（首次）

**輸入**：`notes/login/`，內含 `meeting.md` + `mockup.pdf`

**行為**：分類得 1 native + 1 convertible → 偵測 markitdown OK → `markitdown notes/login/mockup.pdf -o notes/login/.spec-writer/converted/mockup.md` → Read 兩個檔案 → file-list 寫入兩項（meeting.md 為 native、mockup.pdf → converted/mockup.md）。

### Edge：增量分析，使用者多丟一個檔

**情境**：上次跑過 `notes/login/`，file-list 已有 meeting.md + mockup.pdf。使用者再丟 `screenshots-text.md`。

**行為**：
1. Glob 掃出 3 個檔案
2. Read 既有 file-list → previous = {meeting.md, mockup.pdf}
3. 比對 → new = [screenshots-text.md]、unchanged = [meeting.md, mockup.pdf]
4. 只對 screenshots-text.md 做分類（native）
5. 不重轉 mockup.pdf（直接讀 `.spec-writer/converted/mockup.md`）
6. file-list 加上 screenshots-text.md，更新時間戳

### Edge：使用者改了 mockup.pdf（size 或 mtime 變動）

**情境 A**：file-list 記 mockup.pdf 是 102400 bytes，重掃發現是 105200 bytes → size 不同 → changed。
**情境 B**：size 同（同尺寸 typo 修改），但 mtime 從 1714030000 變 1714050000 → changed。

**行為**：判定為 changed → 視同 new → 重新轉檔覆蓋 `converted/mockup.md` → Phase 6 寫檔成功後 file-list 才更新。

### Edge：使用者手動刪了 .spec-writer/converted/mockup.md

**情境**：file-list 記 mockup.pdf 是 unchanged 且有 cache，但 `test -f notes/login/.spec-writer/converted/mockup.md` 失敗。

**行為**：cache-missing 觸發 → mockup.pdf promote 為 changed → 重轉。

### Edge：使用者裝了 markitdown 後重跑（M2 場景）

**情境**：上次跑時 markitdown 未裝，使用者選 (b) 忽略繼續，file-list 把 requirements.pdf 標為 unsupported（reason: 「markitdown 未安裝且使用者選擇忽略」）。使用者裝好 markitdown 後重跑，沒對檔案動任何手腳。

**行為**：
1. 步驟 5 markitdown 偵測退出碼 0
2. promote-on-markitdown-install 觸發 → requirements.pdf 從 unsupported 改判 changed
3. 步驟 6 一起轉檔
4. Phase 2 一起萃取，使用者**完全不需手動刪 file-list**

### Edge：含 PDF，markitdown 未裝

**輸入**：`notes/login/` 內含 `meeting.md` + `mockup.pdf`

**行為**：偵測 markitdown 失敗 → **不轉檔、也不寫 file-list**（保留現狀）→ 回 spec-writer Phase 1 的「未安裝決策」流程，由 spec-writer 問使用者是否忽略 PDF。使用者選忽略 → 回頭把 PDF 標為 unsupported 寫入 file-list。

### Rejection：資料夾全是不支援的格式

**輸入**：`notes/screenshots/` 全是 `.png` 截圖

**行為**：分類後 native = 0、convertible = 0、unsupported = N。寫入 file-list 記錄這些檔案為 unsupported → 回 `INSUFFICIENT_DATA：資料夾內無可分析的檔案。截圖請先 OCR 成 .txt 或 .md，或用文字描述補充。`

### Rejection：資訊過少

**輸入**：`notes/short.md` 只有一句「做個登入功能」

**行為**：成功讀檔，只抓到類別 1 `登入`，其他 6 類全空。feature name 可推斷 `login`。但資訊過少，spec-writer 應在 Uncertainty Protocol 回覆 `INSUFFICIENT_DATA`，不要硬上。
