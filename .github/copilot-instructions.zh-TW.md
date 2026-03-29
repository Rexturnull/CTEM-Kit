# CTEM Kit — 全域專案指引

## 專案定位

此工作區是一個 **CTEM（持續威脅暴露管理）Kit** — 基於提示驅動的自動化工具組，對應 Gartner CTEM 五階段框架。不包含程式碼，所有邏輯透過 Markdown 指引、技能和提示定義表達。

## 啟用規則

以下 CTEM 規則**僅在使用者明確進入 CTEM 情境時生效**：

- 呼叫 `/ctem-flow` 或任何 `/ctem-*` 技能
- 明確提及 CTEM session（如「開始新輪次」、「繼續評估」、「phase complete」）

**非 CTEM 請求不受下列規則約束**，以一般 Copilot 行為回應即可。

## CTEM 模式下的全域規則

1. **流程控制權歸屬**：階段排序、回溯、轉換決策由 `/ctem-flow` 管理。五階段定義、回溯邏輯、狀態更新的詳細規則見 `ctem-flow.prompt.md`，此處不重複。
2. **狀態檔案治理**：`ctem-state.md` 的讀寫規則見 `ctem-state-protocol.instructions.md`，此處不重複。
3. **工具整合**：技能指示使用者執行外部工具（nmap、nuclei、nessus 等）並解析貼上的輸出，不直接執行工具。
4. **語言**：所有提示內容以繁體中文呈現。
