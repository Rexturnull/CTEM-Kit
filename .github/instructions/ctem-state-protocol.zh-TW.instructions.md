---
description: "CTEM 狀態管理協定。用途：讀取或寫入 ctem-state.md、追蹤階段進度、執行回溯檢查、管理階段轉換。"
---

# CTEM 狀態協定

此協定規範所有 CTEM 技能和提示如何與 `ctem-state.md` 互動。

## 合法的階段狀態值

| 狀態 | 意義 |
|------|------|
| `not_started` | 階段尚未開始 |
| `in_progress` | 階段正在執行中 |
| `completed` | 階段已完成，所有產出已記錄 |

不允許使用其他狀態值。

## 先決條件

階段的先決條件為：所有前序階段的狀態皆為 `completed`。Scoping（範圍界定）無先決條件。

## 開始任何階段之前

1. 從專案根目錄讀取 `ctem-state.md`
2. 確認目前階段並驗證先決條件是否已滿足（見上方）
3. 若先決條件未滿足，**停止**並回報缺少的項目
4. 將該階段在 Phase Status 表中的狀態設為 `in_progress`

## 完成任何階段之後

1. 更新 `ctem-state.md` 中的階段狀態表：
   - 將已完成的階段狀態設為 `completed`
   - 記錄時間戳記及關鍵發現摘要
2. 在 `## Phase Summaries` 區塊寫入（或更新）該階段的摘要（見下方）
3. 在轉換日誌中新增一筆記錄
4. 回溯檢查與報告生成由 `/ctem-flow` 管理 — 本協定不執行這些動作

## 階段摘要區塊

每個階段完成時，在 `ctem-state.md` 的 `## Phase Summaries` 區域寫入結構化摘要。這是階段間資料傳遞的主要機制 — 下游階段讀取這些摘要，而非存取多個檔案。

摘要區塊必須作為 `## Phase Summaries` 的子層級，命名為 `### Scoping Summary`、`### Discovery Summary`、`### Prioritization Summary`、`### Validation Summary`、`### Mobilization Summary`。

每個階段技能定義自己的摘要格式。摘要必須簡潔、結構化（適當使用表格），且僅包含下游階段所需的可操作資料。

當因回溯重新執行某階段時，其摘要區塊必須被**替換**（而非附加）為更新後的結果。

## 狀態檔案格式

`ctem-state.md` 遵循固定結構（參見 `ctem-state.md` 中的範本）。
