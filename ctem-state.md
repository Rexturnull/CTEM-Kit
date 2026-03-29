# CTEM Session State

<!-- 此檔案由 AI 在執行 CTEM 流程時自動維護 -->
<!-- 每個階段結束後更新狀態表和轉換日誌 -->
<!-- ctem-flow 根據此檔案判斷下一步和回溯需求 -->

## Session Info

| Field | Value |
|-------|-------|
| Session ID | *(initialized by coordinator)* |
| Target | *(initialized by coordinator)* |
| Started | *(timestamp)* |
| Current Phase | *(updated after each transition)* |
| Backtrack Count | 0 |
| Max Backtracks | 3 |

## Phase Status

| # | Phase | Status | Key Findings Summary | Last Updated |
|---|-------|--------|----------------------|--------------|
| 1 | Scoping | not_started | — | — |
| 2 | Discovery | not_started | — | — |
| 3 | Prioritization | not_started | — | — |
| 4 | Validation | not_started | — | — |
| 5 | Mobilization | not_started | — | — |

## Transition Log

<!-- 每次階段轉換（包含回溯）記錄在此 -->
<!-- 格式：FROM → TO | REASON | TIMESTAMP -->

| # | From | To | Type | Reason | Timestamp |
|---|------|----|------|--------|-----------|
| *(empty — entries added during session)* | | | | | |

## Backtrack History

<!-- 僅記錄回溯事件，用於追蹤和限制回溯次數 -->

| # | From Phase | To Phase | Trigger | Reasoning |
|---|------------|----------|---------|-----------|
| *(empty — entries added if backtracking occurs)* | | | | |
