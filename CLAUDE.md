# Spec Writer Team

從筆記產生工程師可直接開工的 spec 文件。

## 團隊目標

讀取使用者提供的筆記（notes/ 資料夾的特定檔案），套用擴展版 Atlassian PRD 模板，產出一份完整的 spec 文件到 `specs/{feature-name}/`。

## 溝通語言

**一律使用繁體中文**。技術名詞（API、endpoint、schema、等）可保留英文。

## 單 Agent 架構

本團隊採用單 agent 架構：

- **spec-writer**（agents/ 根目錄）：唯一執行 agent，讀筆記、套模板、驗證、寫檔
- **skills**（skills/）：spec-writer 工作時的參考指引，不是獨立 agent

因工作流為 one-shot（讀筆記 → 產 spec），規模小，不套用多 agent 的 coordinator / worker 拆分。spec-writer 兼任自我驗證。

## 工作流程

1. 使用者呼叫 `/boss notes/{feature}/`（推薦資料夾）或 `/boss notes/a.md notes/b.pdf`（明列檔案）
2. `skills/boss/SKILL.md` 解析參數，原樣傳給 spec-writer（不展開資料夾）
3. spec-writer 依序執行：
   - **Phase 1**：盤點檔案（資料夾掃描 → 分類 native/convertible/unsupported → 增量比對 file-list → markitdown 偵測與轉檔 → 不支援檔案徵詢）
   - **Phase 2**：解析筆記（`skills/parse-notes/SKILL.md`）
   - **Phase 3**：確認 feature 名稱（向使用者確認資料夾名）
   - **Phase 4**：套用模板（`skills/apply-template/SKILL.md`）
   - **Phase 5**：結構驗證（`skills/validate-spec/SKILL.md`）
   - **Phase 6**：寫入 `specs/{feature-name}/` 與 file-list（不回報）
   - **Phase 7**：工程師開工就緒度 review（`skills/readiness-review/SKILL.md`）+ 合併回報給使用者

## 外部依賴

| 工具 | 何時需要 | 安裝 |
|---|---|---|
| markitdown | 筆記資料夾含 `.pdf` `.docx` `.pptx` `.xlsx` `.html` `.csv` `.json` `.xml` `.epub` | `pip install 'markitdown[all]'`（Python 3.10+） |

**spec-writer 不會自動 pip install**。偵測到缺套件 → 列受影響檔案 + 安裝指令，讓使用者選 (a) 安裝重跑、(b) 忽略繼續。

## 增量分析（file_list 機制）

每個 feature 資料夾首次處理後會建立 `notes/{feature}/.spec-writer/`：

- `file-list.md`：已掃過的檔案清單與分類狀態
- `converted/{filename}.md`：markitdown 轉檔結果

之後再對同一資料夾呼叫 `/boss`，agent 只會：
- 處理 file-list 中沒有的「新檔」
- 對 size 變動的檔案重新分類與轉檔
- 略過 unchanged 的轉檔（直接讀 cache）

要強制重做全部 → 手動刪除 `.spec-writer/` 整個資料夾。

## 輸出拆檔規則

- 內容 < 500 行 → 單一 `spec.md`
- 內容 ≥ 500 行 → 拆成：
  - `spec.md`（主文 + 章節 1-4, 8-10）
  - `data-model.md`（章節 5，總是拆）
  - `ui.md`（章節 6，**僅當該章內容 ≥ 100 行才拆**，否則留在主文）
  - `api.md`（章節 7，**僅當該章內容 ≥ 100 行才拆**，否則留在主文）
- 拆檔時 `spec.md` 要有連結指向子檔

## 技術約束

**SQL 方言固定為 Oracle SQL**。所有 spec 章節 5（Data mapping）、章節 7（API）內的 DDL / DML 範例一律採 Oracle 語法（`VARCHAR2`、`NUMBER`、`CHECK` 約束、`SEQUENCE` + `TRIGGER` 替代 auto-increment）。禁止 MySQL `ENUM` / `AUTO_INCREMENT`、PostgreSQL `SERIAL` / `CREATE TYPE`。詳見 `.claude/rules/spec-template.md`。

## 關鍵原則

### CRITICAL：spec 內容必須是筆記寫的事實，禁止任何推論

這是團隊最高紅線。所有 spec 內容**只能**來自筆記字面上寫的內容，不可以加：

- 「合理猜測」（例：筆記沒提時間限制 → 不可寫「3 秒內回應」）
- 「業界標準」（例：筆記沒提演算法 → 不可寫「用 BCrypt」「用 SHA-256」）
- 「工程師會這樣做的」邏輯延伸（例：筆記說「鎖定 30 分鐘」但沒提鎖定期間如何處理新嘗試 → 不可推論「直接拒絕」）
- 「明顯應該有的」細節（例：筆記說「跳轉 dashboard」 → 不可寫成 `/dashboard` URL）
- 索引、約束、欄位型別細節若筆記沒給 → 不可自填

筆記沒寫的內容，依下列規則標記，**禁止填空白或合理推論**：

| 情境 | 標記方式 | 範例 |
|---|---|---|
| 筆記提到但細節缺 | `待補充：{具體缺什麼}` | 「鎖定機制 待補充：解鎖方式（自動 vs 手動）」 |
| 筆記完全沒提，但章節必要 | `偵測不到此內容（筆記未涵蓋）` | 「鎖定期間如何處理新嘗試 → 偵測不到此內容」 |
| 選填章節（6/7）筆記完全沒提 | 整章省略（不寫章節標題） | 章節 7 API 整章不出現 |

### 不硬編

筆記沒寫的內容，按上面紅線規則標記。Open questions 章節必須彙整所有「待補充」與「偵測不到此內容」項目。

### 先問後做

以下情況 **立刻停下來問使用者**：

- 呼叫時未指定筆記檔案 / 資料夾
- 筆記間資訊衝突（A 說 X，B 說 Y）
- 筆記提到「畫面 / 資料表」但無細節
- feature 名稱無法從筆記明確推斷
- `specs/{feature-name}/` 已存在（問覆蓋或改名）
- 資料夾含需轉檔檔案但 markitdown 未安裝（問安裝重跑或忽略繼續）
- 資料夾含 unsupported 檔案（圖片、音訊、影片）（問是否忽略繼續）

### 忠實轉寫

- 筆記是口語、會議紀錄、截圖 OCR → spec 要改寫成正式技術語氣
- 但不加筆記沒有的需求、欄位、流程

## 目錄結構

```
teams/spec-writer/
├── CLAUDE.md                      ← 本檔
├── notes/                         ← 使用者丟筆記的地方
│   ├── README.md
│   └── {feature}/                 ← 推薦：一個功能一個資料夾
│       ├── meeting.md
│       ├── mockup.pdf             ← 非文字檔由 markitdown 轉
│       └── .spec-writer/          ← agent 自動產生（增量分析快取）
│           ├── file-list.md
│           └── converted/
│               └── mockup.md
├── specs/                         ← 產出 spec 的地方
│   └── README.md
└── .claude/
    ├── agents/
    │   └── spec-writer.md         ← 唯一 agent
    ├── skills/
    │   ├── boss/SKILL.md             ← /boss 入口
    │   ├── parse-notes/SKILL.md      ← 含資料夾掃描 + markitdown + file-list 管理
    │   ├── apply-template/SKILL.md
    │   ├── validate-spec/SKILL.md    ← 結構合規檢查
    │   └── readiness-review/SKILL.md ← 工程師開工就緒度 review (Phase 7)
    ├── rules/
    │   └── spec-template.md       ← 模板定義（canonical）
    └── settings.json
```

## 偏離 A-Team 通則的說明

- **不使用 `.worklog/` 多階段結構**：工作流為單次 one-shot，沒有多階段交接。產出的 spec 本身即為最終輸出；`Open questions` 章節即是「待補充清單」。
- **process reviewer 由 spec-writer 兼任**：依 `rules/reviewer-mandate.md` 的例外條款（3 agent 以下），coordinator 可兼任 process review。
- **無 code reviewer**：本團隊產出 Markdown spec 不是程式碼，由 `skills/validate-spec/SKILL.md` 檢查 spec 品質。
