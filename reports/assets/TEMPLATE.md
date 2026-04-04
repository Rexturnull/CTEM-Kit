# Asset Profile

<!-- 每台受檢資產一份，長期維護，跨輪次追蹤 -->
<!-- 檔名慣例：reports/assets/<hostname-or-ip>.md -->
<!-- 例如：reports/assets/10.0.0.1.md、reports/assets/web-prod-01.md -->

## Asset Identity

| Field | Value |
|-------|-------|
| Asset ID | <!-- 唯一識別碼，例如 ASSET-001 --> |
| Hostname | |
| IP Address | |
| OS / Platform | |
| Role / Service | <!-- 例如：Web Server、DB Server、Domain Controller --> |
| Business Criticality | <!-- critical / high / moderate / low --> |
| Owner / Team | |
| First Seen | <!-- 首次納入 CTEM 的 Session ID 與日期 --> |
| Last Assessed | <!-- 最近一次評估的 Session ID 與日期 --> |

## Current Risk Summary

<!-- 反映最新一輪評估後的狀態，每輪結束後更新 -->

| Metric | Value |
|--------|-------|
| Overall Risk Level | <!-- critical / high / medium / low / info --> |
| Open Exposures | <!-- 數量 --> |
| Highest Severity | <!-- 目前未修復的最高嚴重性 --> |
| Trend | <!-- ↑ escalating / → stable / ↓ improving --> |

## Exposure Registry

<!-- 此資產歷來所有被發現的暴露，持續累積 -->
<!-- Severity History = Raw Severity（工具原始評級），由 Discovery (Phase 2) 寫入 -->
<!-- Adjusted Severity = 業務風險調整後的等級，由 Prioritization (Phase 3) 寫入 -->

| # | Exposure ID | Title | First Seen (Session) | Last Seen (Session) | Severity History | Adjusted Severity | Current Status |
|---|-------------|-------|----------------------|---------------------|------------------|-------------------|----------------|
| | | | | | | | |

<!-- Severity History 範例：low (S-001) → medium (S-002) → high (S-003) — 此為 Raw Severity 記錄 -->
<!-- Adjusted Severity 範例：high (S-002) — 由 Prioritization 根據業務影響評估後設定 -->
<!-- Current Status: open / mitigated / accepted / reopened / false-positive -->

## Risk Trend Log

<!-- 每輪評估後新增一列，記錄該輪的風險快照 -->

| # | Session ID | Date | Risk Level | Open Count | Delta | Notes |
|---|------------|------|------------|------------|-------|-------|
| | | | | | | |

<!-- Delta: +2 new, -1 mitigated 等簡要說明 -->

## Remediation History

<!-- 針對此資產的修復行動紀錄 -->

| # | Exposure ID | Action Taken | Date | Verified In (Session) | Result |
|---|-------------|--------------|------|-----------------------|--------|
| | | | | | |

<!-- Result: resolved / partial / failed / pending-verification -->

## Notes

<!-- 資產特殊狀況、維護窗口、已知限制等 -->
