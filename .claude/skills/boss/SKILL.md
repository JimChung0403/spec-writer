---
name: Boss
description: spec-writer 團隊入口 讀取使用者指定的筆記檔案或資料夾並呼叫 spec-writer 產生 spec
disable-model-invocation: true
allowed-tools: ["Agent"]
argument-hint: "[notes/feature/ 或 notes/a.md notes/b.md ...] 或留空讓 agent 詢問"
---

# Boss

spec-writer 團隊的入口。解析使用者參數，呼叫 spec-writer agent 產生 spec。

## 執行

### 解析輸入

從使用者呼叫取得的參數（slash 指令後空白分隔的字串）：

- **資料夾路徑**（結尾是 `/` 或 `ls -d` 視為資料夾）：原樣傳給 spec-writer，**不在這裡展開**。spec-writer 會用 Glob 掃描整個資料夾（含子目錄）並做檔案分類。
- **檔案路徑**：可多個（空白分隔）。支援絕對路徑和相對路徑（相對於 `teams/spec-writer/`）。
- **混合**：資料夾與檔案可混傳，原樣傳遞。
- **無參數**：不自己掃 `notes/`，直接傳空清單讓 spec-writer 問使用者。

### 呼叫 spec-writer

使用 Agent tool：

- `subagent_type`: `"spec-writer"`
- `description`: `"從筆記產生 spec"`
- `prompt`: 下面這個結構

```
<task>
依照團隊 CLAUDE.md 的工作流程，從指定筆記產生一份 spec 文件。
</task>

<notes>
{使用者提供的筆記檔案清單，一行一個；若為空寫 "(無 — 請主動詢問使用者)"}
</notes>

<output_base>
specs/
</output_base>
```

### 不要做

- 不要自己讀筆記內容
- 不要自己展開資料夾（不要呼叫 Glob、ls、find）
- 不要自己偵測 markitdown 是否安裝
- 不要自己產 spec
- 不要呼叫 spec-writer 以外的 agent
- 如果 spec-writer 回報 `INSUFFICIENT_DATA`，把訊息完整傳給使用者，等回答後重新呼叫 spec-writer

## Examples

### Normal：資料夾呼叫（推薦）

**使用者**：`/boss notes/refund-flow/`

**行為**：原樣傳資料夾路徑給 spec-writer：

```
Agent({
  subagent_type: "spec-writer",
  description: "從筆記產生 spec",
  prompt: "<task>...</task>\n<notes>\nnotes/refund-flow/\n</notes>\n<output_base>specs/</output_base>"
})
```

spec-writer 會掃整個 `notes/refund-flow/`（含子目錄），分類 .md/.txt/.pdf/.docx 等檔案。

### Edge：明列檔案

**使用者**：`/boss notes/refund/meeting.md notes/refund/draft.md notes/refund/spec-draft.pdf`

**行為**：三個路徑原樣傳遞，混合 .md 與 .pdf：

```
<notes>
notes/refund/meeting.md
notes/refund/draft.md
notes/refund/spec-draft.pdf
</notes>
```

spec-writer 收到後會先檢查 markitdown 是否安裝再決定怎麼處理 .pdf。

### Rejection：無參數

**使用者**：`/boss`

**行為**：呼叫 spec-writer，`<notes>` 寫 `(無 — 請主動詢問使用者)`。spec-writer 會主動回問使用者要處理哪些筆記或資料夾，**不要**在這個 skill 裡自己問。
