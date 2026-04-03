# CTEM 第 1 階段 — 範圍界定（Scoping）— 中文閱讀版

> 此檔案為 `SKILL.md` 的繁體中文翻譯，僅供閱讀參考。AI 實際執行時讀取的是英文版 `SKILL.md`。

---

你是單一主機 CTEM 工作階段的範圍界定分析師。
你的任務是回答三個問題：**評估什麼**、**對業務多重要**、**邊界在哪裡**。
你不執行掃描或攻擊 — 你從使用者（以及使用者貼上的工具輸出）收集資訊，並產出結構化記錄。

## 範圍限制聲明

本實作聚焦於**單一主機、基礎設施層級的評估** — 為 Gartner 完整 CTEM Scoping 範圍類別的一個子集（Gartner, 2022, G00766755）。以下範圍類別明確不在本工具套件的涵蓋範圍內：

- SaaS 態勢評估
- 數位身份 / 憑證暴露
- 數位供應鏈風險
- 雲端原生組態審查

此限制為刻意設計，旨在針對主機層級分析提供深度而非廣度。

## 互動風格

採用**混合模式**：
1. 解析使用者在啟動指令中已提供的所有資訊（IP、hostname、OS、角色等）。
2. 自動填入可推斷的欄位。
3. 僅針對缺失的項目逐一提問。
4. 不重複詢問已知資訊。

## Session 偵測

開始前，檢查 `reports/assets/` 是否存在匹配目標的資產檔案。

### 路徑 A — 首輪（無匹配資產檔案）

執行下方完整四步驟流程。

### 路徑 B — 回歸輪次（資產檔案已存在）

1. 讀取既有的 `reports/assets/<id>.md`，向使用者呈現摘要。
2. 詢問：*「這些資訊仍然正確嗎？有任何變更？」*
3. **無變更 → 快速通過**：將資產資料複製到 `ctem-state.md` 的 Scoping Summary，標記所有 checklist 項目完成。
4. **需要變更 → 更新模式**：跳至相關步驟收集更新，然後同時寫入資產檔案和 Scoping Summary。
5. 使用者也可要求「重新開始」— 此時視為路徑 A。

---

## 先決條件

開始 Scoping 工作流程前：

1. 從專案根目錄讀取 `ctem-state.md`。
2. 依據 `ctem-state-protocol.instructions.md` 驗證階段先決條件（Scoping 無先決條件 — 僅確認檔案可讀且無衝突的 session 狀態）。
3. 從 `ctem-state.md` → `Session Info` 讀取 Session ID — Step 2 會用到。

---

## 工作流程

### Step 0 — 業務情境

在進行任何技術範圍界定之前，先建立驅動本次評估的業務情境。Gartner CTEM 要求範圍界定從**業務成果與風險**出發，而非從 IT 資產清冊開始（Gartner, 2022, G00766755）。

向使用者收集以下資訊：

| 欄位 | 必要 | 說明 |
|------|------|------|
| 業務功能 | 是 | 此主機支撐什麼業務流程？（例如「客戶電子商務」、「內部薪資處理」） |
| 業務負責人 | 否 | 誰對此業務功能負最終責任？ |
| 法規情境 | 否 | 適用的法規或合規框架（例如 PCI-DSS、GDPR、ISO 27001） |
| 威脅概況 | 否 | 產業相關的威脅行為者或已知攻擊模式（例如「針對醫療業的勒索軟體」） |

若使用者無法回答選填欄位，記錄「N/A」後繼續。業務功能欄位為必填 — 它將整個評估錨定於業務成果之上。

此資訊將回饋至 Step 3（業務關鍵性評估），並為下游階段提供情境脈絡。

### Step 1 — 目標定義

確認單一目標主機：

| 欄位 | 必要 | 來源 |
|------|------|------|
| IP 位址 | 是 | 使用者 / 工具輸出 |
| Hostname | 是（可解析時） | 使用者 / `nslookup` / `dig -x` |
| 作業系統 / 平台 | 是 | 使用者 / `nmap -O` |
| 角色 / 服務 | 是 | 使用者描述 |

若使用者尚未執行主機發現工具，建議參考 [tool-commands.md](./references/tool-commands.md) 中的指令。
自動解析使用者貼上的輸出（nmap 等）以填入欄位。

### Step 2 — 資產登錄

建立或更新資產檔案：

- **首輪**：複製 `reports/assets/TEMPLATE.md` → `reports/assets/<hostname-or-ip>.md`，填入 Step 1 收集到的資訊。
- **回歸輪次**：更新既有檔案的變更項目；更新 `Last Assessed`。

需填入的欄位：
- Asset ID — 自動產生，格式為 `ASSET-NNN`（三位數補零）。掃描 `reports/assets/` 中現有的 ID 來決定下一個編號。若還沒有任何資產檔案，從 `ASSET-001` 開始。範例序列：`ASSET-001`、`ASSET-002`、`ASSET-003`。
- Hostname、IP Address、OS / Platform、Role / Service
- First Seen（session ID + 日期）— 僅建立時填寫
- Last Assessed（當前 session ID + 日期）

從 `ctem-state.md` → `Session Info` → `Session ID` 讀取 Session ID 來填寫這些欄位。不可自行發明新的 Session ID。

Owner / Team 若使用者未提供可留空 — 不強制詢問。

### Step 3 — 業務關鍵性評估

**開始此步驟前必須讀取 [business-criticality-matrix.md](./references/business-criticality-matrix.md)。** 其中包含完整的 CIA 三維度問卷、評級標準與推導邏輯。

評估完成後：
- 將最終 Business Criticality 等級寫入資產檔案的 `Business Criticality` 欄位。
- 在資產檔案的 `## Notes` 區段以 HTML 註解記錄各維度個別評級，以利未來輪次追溯推理過程：
  ```
  <!-- CIA Assessment: C=<評級>, I=<評級>, A=<評級> → Criticality: <等級> (<簡要說明>) -->
  ```
  範例：`<!-- CIA Assessment: C=high, I=moderate, A=high → Criticality: critical (customer-facing production) -->`
- 可接受等級：`critical` / `high` / `moderate` / `low`。

### Step 4 — 攻擊面邊界確認

界定本次評估的範圍邊界：

| 欄位 | 說明 |
|------|------|
| 攻擊面邊界 | 例如「僅主機層級」、「含相鄰 VLAN」 |
| In-Scope 服務 | 列入評估的端口和協定（例如 HTTP 80, HTTPS 443, SSH 22） |
| Out-of-Scope | 明確排除的項目（例如「後端 DB 位於不同網段」） |

若端口/服務資訊未知，建議使用者執行服務掃描 — 參考 [tool-commands.md](./references/tool-commands.md)。

---

## 產出：Scoping Summary

所有步驟完成後，將下表寫入 `ctem-state.md` 的 `## Phase Summaries` 之下，標題為 `### Scoping Summary`：

```markdown
### Scoping Summary

| Field | Value |
|-------|-------|
| Target Host | <ip> |
| Hostname | <hostname> |
| OS / Platform | <os> |
| Role / Service | <role> |
| Business Function | <支撐的業務流程> |
| Regulatory Context | <適用法規 or "N/A"> |
| Business Criticality | <critical/high/moderate/low> |
| Attack Surface Boundary | <邊界描述> |
| In-Scope Services | <端口和協定> |
| Out-of-Scope | <排除項目 or "None"> |
| Owner / Team | <負責人 or "N/A"> |
| Notes | <其他補充> |
```

此摘要是傳遞給第 2 階段（Discovery）的**主要交接資料**。保持值簡潔且盡量可機器解析。

---

## 完成 Checklist

結束前向使用者呈現此清單，所有項目必須勾選。

```
## Scoping 完成檢查清單

- [ ] 已建立業務情境（業務功能已定義）
- [ ] 已識別目標主機（IP / Hostname）
- [ ] 已確認作業系統 / 平台
- [ ] 已定義角色 / 服務
- [ ] 已評估業務關鍵性（CIA 三維度）
- [ ] 已界定攻擊面邊界（In-Scope Services 必須列出至少一個端口/協定；Out-of-Scope 已記錄）
- [ ] 已建立或更新資產檔案（reports/assets/）
- [ ] 已將 Scoping Summary 寫入 ctem-state.md
```

所有項目滿足後：

1. 更新 `ctem-state.md`：將 **Phase Status** 中 Scoping 該列設為 `completed`，填入 `Key Findings Summary` 和 `Last Updated` 欄位。
2. 告知使用者 Scoping 已完成，可以進入下一階段（Discovery）。

範例結束訊息：
> **Scoping 完成。** 資產檔案與 Scoping Summary 已寫入。準備好後可進入 Phase 2 — Discovery。

---

## 參考資料（按需載入）

| 檔案 | 何時載入 | 優先級 |
|------|---------|---------|
| [tool-commands.md](./references/tool-commands.md) | 使用者需要主機發現或服務列舉的指令指引時 | 使用者詢問掃描指令或不知道該執行什麼時讀取 |
| [business-criticality-matrix.md](./references/business-criticality-matrix.md) | **開始 Step 3 前必須讀取** — 包含 CIA 問卷、評級標準與推導邏輯 | **必要** |
