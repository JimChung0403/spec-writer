# notes/

把筆記丟在這裡。**推薦每個功能一個資料夾**，agent 會掃整個資料夾自動分析。

## 推薦結構（每個功能一個資料夾）

```
notes/
├── refund-flow/                  ← 一個 feature 一個資料夾
│   ├── meeting-2026-04-15.md
│   ├── ui-mockup.pdf             ← 非文字檔需 markitdown
│   ├── data-table-draft.md
│   └── .spec-writer/             ← agent 自動產生的快取（勿手動編輯）
│       ├── file-list.md          ← 檔案清單與轉檔狀態
│       └── converted/            ← markitdown 轉檔結果
│           └── ui-mockup.md
└── login/
    └── spec-draft.md
```

平鋪也可以，但用 `/boss` 時要明列檔案路徑。

## 支援的檔案類型

| 類別 | 副檔名 | 處理方式 |
|---|---|---|
| 原生（直接讀） | `.md` `.txt` `.markdown` | 直接讀，最快 |
| 需轉檔（透過 markitdown） | `.pdf` `.docx` `.pptx` `.xlsx` `.html` `.csv` `.json` `.xml` `.epub` | 需先安裝 markitdown |
| 不支援 | 圖片 `.png/.jpg`、音訊 `.mp3/.wav`、影片 `.mp4`、壓縮檔 `.zip` | agent 會列出並徵求是否忽略繼續 |

## 安裝 markitdown（只在資料夾含 .pdf/.docx/.xlsx 等時才需要）

```bash
pip install 'markitdown[all]'
```

或精選依賴（較輕量）：

```bash
pip install 'markitdown[pdf,docx,pptx,xlsx]'
```

需 Python 3.10+。安裝後重跑 `/boss` 即可。

未安裝時 agent 會偵測到並讓你選擇：(a) 安裝後重跑、(b) 忽略這些檔案只處理 .md/.txt。

## 呼叫方式

**資料夾（推薦）**：

```
/boss notes/refund-flow/
```

agent 會掃整個資料夾、自動分類、必要時轉檔、產生 spec 到 `specs/refund-flow/`。

**明列檔案**：

```
/boss notes/refund-flow/meeting-2026-04-15.md notes/refund-flow/ui-mockup.pdf
```

**留空（會詢問）**：

```
/boss
```

## 增量分析

每個資料夾跑過後會留下 `.spec-writer/file-list.md` 記錄已分析的檔案。下次再丟新檔進去重跑 `/boss notes/refund-flow/`，agent 只會處理新檔，已轉檔的不會重轉。

要強制重新分析全部 → 刪掉 `.spec-writer/` 整個資料夾再跑。

## 不要做

- 不要動 `.spec-writer/` 內的檔案（會讓增量分析失準）
- 不要把 `.spec-writer/` 提交到 git（建議加進 `.gitignore`）
- 不要把 spec 放到 `notes/`（spec 屬於 `specs/`）
