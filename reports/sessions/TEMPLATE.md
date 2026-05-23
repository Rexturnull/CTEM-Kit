# CTEM Session Report

<!-- 每完成一輪 CTEM 後，由此模板建立報告 -->
<!-- 檔名慣例：reports/sessions/YYYY-MM-DD-<session-id>.md -->

## Session Info

| Field | Value |
|-------|-------|
| Session ID | |
| Target | |
| Started | |
| Completed | |
| Total Backtracks | |
| Previous Session | <!-- 前一輪 Session ID，首輪填 N/A --> |

## Assets In Scope

<!-- 本輪涵蓋的資產清單，每筆對應 reports/assets/ 下的資產檔案 -->

| # | Asset ID | Hostname / IP | Role | Criticality |
|---|----------|---------------|------|-------------|
| | | | | |

## Phase Results Summary

| # | Phase | Status | Key Findings |
|---|-------|--------|--------------|
| 1 | Scoping | | |
| 2 | Discovery | | |
| 3 | Prioritization | | |
| 4 | Validation | | |
| 5 | Mobilization | | |

## Exposure Summary

<!-- 本輪發現的所有暴露，含嚴重性與對應資產 -->
<!-- Severity = Raw Severity，由 Discovery (Phase 2) 填入 -->
<!-- Adjusted Severity = 業務風險調整後的等級，由 Prioritization (Phase 3) 填入 -->
<!-- Validation Status = 由 Validation (Phase 4) 填入 -->
<!-- Status: 由 Discovery (Phase 2) 設定初始值，Validation (Phase 4) 更新 -->

| # | Exposure ID | Asset ID | Title | Severity | Adjusted Severity | Validation Status | Status | Notes |
|---|-------------|----------|-------|----------|-------------------|-------------------|--------|-------|
| | | | | | | | | |

<!-- Status: open / confirmed / false-positive / mitigated / accepted / reopened -->
<!-- Validation Status: confirmed / false-positive / inconclusive / — (未驗證) -->
<!-- escalated = 本輪嚴重性比前輪升高 -->
<!-- reopened = 前輪已 mitigated 但本輪再次偵測到 -->

## Newly Discovered During Validation

<!-- 驗證過程中新發現的暴露（非前序階段掛單的暴露） -->

| # | Exposure ID | Asset ID | Title | Type | Raw Severity | Adjusted Severity | Source | Validation Status |
|---|-------------|----------|-------|------|-------------|-------------------|--------|-------------------|
| | | | | | | | | |

## Attack Paths Identified

<!-- 攻擊路徑：將多個已確認暴露串聯為一条利用鏈 -->

| # | Path ID | Chain (Exposure IDs) | Description | Combined Impact | Status |
|---|---------|---------------------|-------------|-----------------|--------|
| | | | | | |

<!-- Status: confirmed / partial / theoretical -->

## Risk Changes Detected

<!-- 與前輪比較，風險等級有變化的項目（由 Prioritization 階段產出） -->

| # | Exposure ID | Asset ID | Previous Severity | Current Severity | Change Reason |
|---|-------------|----------|-------------------|------------------|---------------|
| | | | | | |

## Backtrack History

| # | From Phase | To Phase | Reason |
|---|------------|----------|--------|
| | | | |

## Remediation Actions

| # | Action | Exposure ID | Priority | Owner | Deadline | Status |
|---|--------|-------------|----------|-------|----------|--------|
| | | | | | | |

## Notes

<!-- 額外觀察、經驗教訓、後續追蹤事項 -->
