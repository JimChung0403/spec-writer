# specs/

產出的 spec 文件會放在這裡，每個功能一個資料夾。

## 典型結構

```
specs/
└── refund-flow/
    └── spec.md         ← 主文件（內容 < 500 行時單檔）
```

內容較多時會自動拆檔：

```
specs/
└── order-system/
    ├── spec.md           ← 主文件（章節 1-4, 8-10）
    ├── data-model.md     ← 章節 5：資料對應
    ├── ui.md             ← 章節 6：畫面行為（選填）
    └── api.md            ← 章節 7：API（選填）
```

## 模板章節

每份 spec 包含（空章節會標 `待補充`）：

1. Context
2. User stories
3. Current state
4. Requirements
5. Data mapping
6. UI / 畫面行為（選填）
7. API / 介面（選填）
8. Acceptance criteria
9. Out of scope
10. Open questions / 待補充
