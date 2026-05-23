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

## 欄位寫入權責表（Field Ownership）

以下表格明確定義 Asset Profile (`reports/assets/`) 中每個區塊由哪個階段負責寫入，避免重複或衝突。

| Asset Profile 區塊 | 欄位 | Primary Writer | Secondary Writer | 說明 |
|---------------------|------|----------------|-----------------|------|
| **Asset Identity** | Asset ID, Hostname, IP, OS, Role | Phase 1 (Scoping) | — | 首次建立或更新 |
| **Asset Identity** | Business Criticality | Phase 1 (Scoping) | — | CIA 三維評估結果 |
| **Asset Identity** | First Seen | Phase 1 (Scoping) | — | 僅首次建立時寫入 |
| **Asset Identity** | Last Assessed | Phase 1 (Scoping) | Phase 2 (Discovery) | Phase 1 首次建立時寫入；Phase 2 每輪更新 |
| **Exposure Registry** | Exposure ID, Title, First/Last Seen, Severity History | Phase 2 (Discovery) | Phase 4 (Validation) | Phase 2 新增/更新；Phase 4 新增驗證中發現的暴露 |
| **Exposure Registry** | Current Status | Phase 2 (Discovery) | Phase 4 (Validation) | Phase 2 設初始值；Phase 4 更新為 confirmed/false-positive |
| **Exposure Registry** | Adjusted Severity | Phase 3 (Prioritization) | Phase 4 (Validation) | Phase 3 評估；Phase 4 僅對新發現暴露執行 in-place 評分 |
| **Current Risk Summary** | Overall Risk Level, Open Exposures, Highest Severity, Trend | Phase 3 (Prioritization) | Phase 4 (Validation), Phase 5 (Mobilization) | Phase 3 初始評估；Phase 4 排除 false-positive 後重算；Phase 5 修復/接受後重算 |
| **Risk Trend Log** | 每輪一列 | Phase 5 (Mobilization) / Report Generation | — | Phase 5 完成後由 ctem-flow 觸發 Report Generation 寫入 |
| **Remediation History** | Action ID, Action Taken, Verified, Session ID | Phase 5 (Mobilization) | — | 修復行動紀錄，含 Action ID 與 Session ID 以支援跨 Session 追蹤 |
| **Notes** | CIA Assessment comment | Phase 1 (Scoping) | — | HTML 註解記錄評估詳情 |
| **Notes** | Validation notes | Phase 4 (Validation) | — | inconclusive 結果標記等 |

> **規則**：Primary Writer 為欄位的主要寫入者。Secondary Writer 僅在其職責範圍內更新（如 Phase 4 更新驗證狀態）。未列於表中的階段可**讀取**但不可修改非其職責的欄位。

## 暴露狀態生命週期（Exposure Status Lifecycle）

以下定義 Exposure Registry 中 `Current Status` 的所有合法值與轉換規則。

### 合法狀態值

| 狀態 | 設定者 | 定義 |
|------|--------|------|
| `open` | Phase 2 (Discovery) | 暴露已發現，尚未驗證或處理 |
| `confirmed` | Phase 4 (Validation) | 經驗證可被利用 |
| `false-positive` | Phase 4 (Validation) | 經驗證不可被利用或不存在 |
| `mitigated` | Phase 2 (Discovery) / Phase 5 (Mobilization) | 已修復（由 Discovery 跨 session 比對判定，或 Mobilization 修復後標記） |
| `accepted` | Phase 5 (Mobilization) | 經業務決策接受風險，不修復 |
| `reopened` | Phase 2 (Discovery) | 前輪已 mitigated 但本輪再次偵測到 |

### 狀態轉換矩陣

下表定義所有合法的狀態轉換。未列出的轉換皆為不合法。

| 來源狀態 | 目標狀態 | 觸發者 | 條件 |
|----------|----------|--------|------|
| *(initial)* | `open` | Phase 2 | 新發現暴露首次寫入 |
| `open` | `confirmed` | Phase 4 | 驗證成功（PoC 已執行，可被利用） |
| `open` | `false-positive` | Phase 4 | 驗證失敗（明確證據證明不可利用） |
| `open` | `mitigated` | Phase 2 | 跨 session 比對：本輪未偵測到且使用者確認已修復 |
| `confirmed` | `mitigated` | Phase 5 | 修復完成並經驗證 |
| `confirmed` | `accepted` | Phase 5 | 業務決策接受風險 |
| `mitigated` | `reopened` | Phase 2 | 跨 session 比對：前輪 mitigated 但本輪再次偵測到 |
| `reopened` | `confirmed` | Phase 4 | 重新開啟的暴露經驗證確認可利用 |
| `reopened` | `false-positive` | Phase 4 | 重新開啟的暴露經驗證為誤報 |
| `reopened` | `mitigated` | Phase 2 | 跨 session 比對：本輪未偵測到且確認已修復 |
| `accepted` | `reopened` | Phase 2 | 已接受風險但暴露惡化，重新評估 |

### 狀態轉換圖

```
                          Phase 2          Phase 4            Phase 5
*(initial)* ───→ [open] ──┬─────────→ [confirmed] ──┬──→ [mitigated]
                        │                              │
                        ├─────────→ [false-positive]   └──→ [accepted]
                        │                                      │
                        └──→ [mitigated] ───→ [reopened] ────┘
                                             │
                                  Phase 4    ├──→ [confirmed]
                                             └──→ [false-positive]
```

> **説明**：Discovery (Phase 2) 運用的 cross-session 狀態（`new`、`known`、`escalated`）為暫時性比對標記，最終寫入 Exposure Registry 時一律轉化為 `open`。其在 Discovery Summary 的 `Status` 欄位中保留，用於記錄縱向變化但不作為持久狀態。
