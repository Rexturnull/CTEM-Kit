# Reports 目錄說明

## 目錄結構

```
reports/
├── README.md              ← 本檔案
├── sessions/              ← 輪次報告：每輪 CTEM 一份
│   ├── TEMPLATE.md
│   └── 2026-03-29-S001.md （範例）
└── assets/                ← 資產檔案：每台機器一份，長期追蹤
    ├── TEMPLATE.md
    └── 10.0.0.1.md        （範例）
```

## 兩層資料模型

| 維度 | 位置 | 用途 | 生命週期 |
|------|------|------|----------|
| **Session Report** | `sessions/` | 一輪 CTEM 的完整快照（橫切面） | 每輪建立一份，完成後封存 |
| **Asset Profile** | `assets/` | 單台機器的長期風險追蹤（縱切面） | 首次發現時建立，每輪更新 |

## 關聯方式

兩者透過 **Session ID** 和 **Asset ID** 互相引用：

- Session Report 的 `Exposure Summary` 列出每個暴露所屬的 Asset ID
- Asset Profile 的 `Exposure Registry` 記錄每個暴露首次/末次出現的 Session ID
- Asset Profile 的 `Risk Trend Log` 每輪新增一列，呈現風險升降趨勢

```
Session Report (S-002)          Asset Profile (ASSET-001)
┌─────────────────────┐         ┌─────────────────────────┐
│ Exposure Summary     │         │ Exposure Registry        │
│ EXP-005 → ASSET-001 │───引用──│ EXP-005: Low(S-001)→High(S-002) │
│ EXP-006 → ASSET-003 │         │                          │
├─────────────────────┤         ├─────────────────────────┤
│ Risk Changes         │         │ Risk Trend Log           │
│ ASSET-001: Med → High│───引用──│ S-002: High, +1 new     │
└─────────────────────┘         └─────────────────────────┘
```

## 跨輪風險升級偵測

當 agent 執行 Prioritization 階段時：

1. 讀取目標資產的 `reports/assets/<asset>.md`
2. 比對 `Exposure Registry` 中前輪嚴重性 vs 本輪評估結果
3. 若嚴重性升高，在 Session Report 的 `Risk Changes Detected` 表中標記
4. 更新 Asset Profile 的 `Severity History` 欄位（例如 `Low (S-001) → High (S-002)`）
5. ctem-flow 的回溯檢查據此決定是否需要回溯至 Prioritization

## 命名慣例

| 類型 | 檔名格式 | 範例 |
|------|----------|------|
| Session Report | `YYYY-MM-DD-<session-id>.md` | `2026-03-29-S001.md` |
| Asset Profile | `<hostname-or-ip>.md` | `10.0.0.1.md`、`web-prod-01.md` |
| Session ID | `S-XXX` | `S-001`、`S-002` |
| Asset ID | `ASSET-XXX` | `ASSET-001` |
| Exposure ID | `EXP-XXX` | `EXP-001` |
