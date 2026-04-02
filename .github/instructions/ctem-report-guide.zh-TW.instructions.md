---
description: "CTEM 報告產生指南。用途：五階段全部完成、使用者要求輪次報告、歸檔工作階段時讀取。按需載入 — 不要預先載入。"
---

# CTEM 報告產生指南

此文件由 `/ctem-flow` 在需要產生報告時按需讀取。

## 輪次報告

1. 複製 `reports/sessions/TEMPLATE.md` 為 `reports/sessions/YYYY-MM-DD-<session-id>.md`
2. 從 `ctem-state.md` 的工作階段資料填入所有區塊
3. 寫入前先與使用者確認報告內容

## 資產檔案更新

資產檔案在 Scoping（Step 2）時已**建立**基本身份欄位。本節在五階段全部完成後更新 **Exposure Registry、Risk Trend Log 和 Current Risk Summary** 欄位。

報告產生後，對每台範圍內資產：

1. 若 `reports/assets/<asset>.md` 不存在，從 `reports/assets/TEMPLATE.md` 建立
2. 更新以下區塊：
   - **Exposure Registry**：新增或更新暴露項目
   - **Risk Trend Log**：新增本輪的一列記錄
   - **Current Risk Summary**：反映最新評估結果
3. 在 `Severity History` 記錄嚴重性變化（例 `Low (S-001) → High (S-002)`）
4. 每個資產檔案寫入前先與使用者確認

## 報告完成後

報告與資產檔案皆確認完成後：

- 將 `ctem-state.md` 重置為空白範本（所有階段 `not_started`，日誌清空）
- 通知使用者工作階段已歸檔，可啟動新的輪次
