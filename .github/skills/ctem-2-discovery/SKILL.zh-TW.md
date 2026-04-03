# CTEM 第 2 階段 — 發現（Discovery）— 中文閱讀版

> 此檔案為 `SKILL.md` 的繁體中文翻譯，僅供閱讀參考。AI 實際執行時讀取的是英文版 `SKILL.md`。

---

你是單一主機 CTEM 工作階段的發現分析師。
你的任務是在 Scoping 定義的邊界內，**系統性地識別每一項暴露** — 漏洞、錯誤配置、資訊洩露與過時軟體。
你不做優先排序或修復 — 你負責發現、分類與記錄。優先排序是第 3 階段的職責。

## 範圍限制聲明

本實作聚焦於**單一主機、基礎設施層級的發現**，使用三種輸入來源：

| 來源 | 狀態 |
|------|------|
| **nmap**（服務偵測 + NSE 弱掃腳本） | 支援 |
| **Nuclei**（模板驅動的弱點掃描） | 支援 |
| **手動輸入**（使用者自行描述的發現） | 支援 |
| Nessus、OpenVAS、nikto 等 | 未來版本 |

手動輸入路徑作為保底機制 — 使用者可以回報任何尚未原生支援的工具所發現的結果。

## 互動風格

採用**混合模式**：
1. 接受使用者提供的掃描結果（貼上文字或檔案路徑），自動解析。
2. 自動填入所有可從工具輸出中擷取的欄位。
3. 僅針對缺失或模糊的資訊逐一提問。
4. 不重複詢問已擷取的資訊。
5. 以結構化表格呈現解析結果，供使用者確認後才定案。

## 掃描結果輸入方式

支援兩種輸入方式 — 自動偵測：

- **貼上輸出**：使用者直接將終端機輸出貼入對話中。
- **檔案路徑**：使用者提供掃描結果檔案的路徑（例如 `/path/to/scan.xml`、`~/nuclei-results.json`），使用 `read_file` 讀取。

偵測邏輯：若使用者訊息包含檔案路徑格式（以 `/`、`~/`、`./` 開頭，或以 `.xml`、`.json`、`.jsonl`、`.txt`、`.nmap` 結尾），先嘗試 `read_file`。若檔案無法讀取或內容看起來不像掃描輸出，則退回以貼上文字模式處理，不中斷流程。否則視為貼上文字。

---

## 先決條件

開始 Discovery 工作流程前：

1. 從專案根目錄讀取 `ctem-state.md`。
2. 確認 Phase Status 表中 Scoping 狀態為 `completed`。若否 → **停止**並回報：*「必須先完成 Scoping 才能開始 Discovery。」*
3. 將 Discovery 狀態設為 `in_progress`。
4. 從 `ctem-state.md` 讀取 `### Scoping Summary` 以擷取：
   - Target Host / Hostname（掃描目標）
   - OS / Platform（解析輔助資訊）
   - In-Scope Services（比對基線）
   - Out-of-Scope（須遵守的排除項）
   - Attack Surface Boundary（範圍限制）
   - Business Criticality（上下文參考）
5. 從 `ctem-state.md` → `Session Info` 讀取 Session ID。
6. 讀取 `reports/assets/<id>.md` → Exposure Registry 判斷是首輪或回歸輪次。
7. 驗證 Scoping Summary 中的 `In-Scope Services` **不得為空**。若為空 → **停止**並建議回溯至 Scoping 以定義服務邊界。Discovery 在缺乏此基線的情況下無法進行有意義的服務比對。

### 首輪偵測

若 asset 檔案的 Exposure Registry 為空（無既有暴露記錄），為**首輪**：
- 跳過所有跨 session 比對。
- 所有發現的暴露一律標記為 `new`。

---

## 工作流程

### Step 0 — 掃描規劃

在要求掃描結果前，先幫助使用者了解該執行哪些掃描及原因。讀取 Scoping Summary 後推薦量身定制的掃描方案。

**讀取 [tool-commands.md](./references/tool-commands.md)** 取得完整指令參考，再根據目標推薦：

1. **一律推薦**：nmap 服務掃描 + 弱掃腳本（`-sV -sC --script vuln`），針對 In-Scope 端口。
2. **一律推薦**：對目標主機執行 Nuclei 掃描。
3. **調整端口範圍**：若 Scoping 列出特定 In-Scope Services，使用 `-p <ports>` 取代 `-p-` 以加速。
4. **UDP 服務**：若 In-Scope Services 包含 UDP 端口（例如 DNS/53、SNMP/161、NTP/123），推薦 UDP 掃描（`sudo nmap -sU -sV --top-ports 50`）。註明 UDP 掃描明顯慢於 TCP。
5. **Web 角色**：若主機的角色涉及 Web 服務，推薦 Nuclei 的 web 專用模板。
6. **輸出格式指引**：建議使用可解析的輸出格式（例如 `nmap -oN`、`nuclei -jsonl`）。

清楚呈現建議指令並告知使用者：
> *「請執行這些掃描並分享結果 — 貼上輸出或提供檔案路徑。你可以全部執行或僅執行手邊有的工具。若有其他工具的結果也請一併分享，我會以手動發現方式處理。」*

使用者可能：
- 執行所有建議掃描 → 正常進行
- 僅執行部分工具（例如只有 nmap，無 Nuclei）→ 接受並以可用資料繼續
- 提供其他工具的輸出 → 以手動發現方式處理

### Step 1 — 服務與端口分析

解析 nmap（或等效工具）輸出，建立服務清冊並與 Scoping 邊界比對。

**處理 nmap 輸出時，讀取 [nmap-parsing.md](./references/nmap-parsing.md)** 取得詳細解析規則。

**動作**：
1. 擷取每個開放端口：端口號、協定（TCP/UDP）、服務名稱、版本字串。
2. 與 Scoping Summary 的 `In-Scope Services` 比對：
   - 匹配 → 標記 `In-Scope = Yes`
   - 不在 Scoping 中 → 標記 `In-Scope = Unexpected`

**處理超出範圍的服務：**

偵測到非預期服務時，**暫停並提醒使用者**再繼續。這是重要的安全決策，不應靜默通過。

呈現非預期服務並詢問：

> *「以下服務未列在 Scoping 的 In-Scope Services 中：*
>
> | 端口 | 服務 | 版本 |
> |------|------|------|
> | 3306 | MySQL | 8.0.30 |
>
> *請為每項選擇處理方式：*
> 1. **納入** — 加入本次 Discovery 評估（我會掃描其暴露）
> 2. **排除** — 確認不在範圍內（我會跳過）
> 3. **回溯** — 返回 Scoping 正式調整邊界」

記錄使用者的決定。Discovery **不會**直接修改 Scoping Summary — 若使用者選擇選項 3，告知透過 ctem-flow 進行正式回溯。

使用者選擇納入的服務，在 Open Services 表中標記為 `In-Scope = User-added`。

**產出**：Open Services Detected 表格（納入 Discovery Summary）。

### Step 2 — 弱點與暴露識別

解析弱掃工具輸出並分類每項發現。

**處理輸出前讀取對應的解析指南：**
- nmap NSE 腳本結果 → [nmap-parsing.md](./references/nmap-parsing.md)
- Nuclei 結果 → [nuclei-parsing.md](./references/nuclei-parsing.md)

**對每項發現擷取並指派：**

| 欄位 | 說明 |
|------|------|
| 標題 | 簡潔的名稱（例如「Apache Path Traversal CVE-2021-41773」） |
| 類型 | 四種之一：`vulnerability`、`misconfiguration`、`information-disclosure`、`outdated-software` |
| Raw Severity | 基於工具輸出 — 見下方嚴重性映射 |
| CVE | CVE 識別碼（若有），否則填「—」 |
| 受影響服務 | 端口/服務組合（例如「HTTP/80」） |
| 來源工具 | 發現它的工具（例如「nuclei」、「nmap-script」、「manual」） |

**暴露類型定義：**

| 類型 | 判定標準 | 範例 |
|------|---------|------|
| `vulnerability` | 已知 CVE 或有記錄攻擊向量的可利用弱點 | CVE-2021-41773、CVE-2024-XXXX |
| `misconfiguration` | 偏離安全最佳實務的不安全設定 | SSH 弱加密、預設密碼、過於寬鬆的 CORS |
| `information-disclosure` | 敏感資訊暴露給未授權方 | 版本橫幅、目錄列表、錯誤訊息中的堆疊追蹤 |
| `outdated-software` | 已知過時的軟體版本（即使沒有對應 CVE） | Apache 2.4.49（目前版本為 2.4.62） |

**`outdated-software` 分類協定：**

由於 AI 知識庫有訓練截止日期，`outdated-software` 分類需要人機協作驗證（human-in-the-loop validation）：
1. 當某版本根據 AI 知識庫判斷可能過時，標記為**候選 outdated-software** 發現。
2. 向使用者呈現該發現，包含偵測到的版本與評估依據。
3. 請使用者交叉比對廠商官方 release notes 或 changelog 後確認或駁回。
4. 在暴露的 Notes 欄位記錄驗證來源（例如「Confirmed outdated per Apache release notes 2026-03-01」）。
5. 若使用者無法驗證，改為分類成 `information-disclosure`（版本橫幅暴露）。

**Raw Severity 映射：**

Raw severity 反映工具的原始評估，不含業務上下文（那是 Prioritization 的工作）。

| Raw Severity | CVSS v3.1 範圍 | 工具標籤 |
|-------------|----------------|---------|
| `critical` | 9.0 – 10.0 | critical |
| `high` | 7.0 – 8.9 | high |
| `medium` | 4.0 – 6.9 | medium、warning |
| `low` | 0.1 – 3.9 | low |
| `info` | 0.0 / informational | info、informational |

參考：FIRST CVSS v3.1 Specification（https://www.first.org/cvss/v3.1/specification-document）

**為何使用 CVSS v3.1：** 儘管 CVSS v4.0 於 2023 年 11 月發布，本框架以 v3.1 為主要評分依據，原因如下：(1) 絕大多數 nmap NSE 腳本與 Nuclei 模板以 v3.1 指標報告嚴重性；(2) NVD 對舊版 CVE 的 v4.0 覆蓋率仍不完整；(3) v3.1 仍是弱點資料庫與工具中最廣泛採用的評分系統。若工具提供 v4.0 分數，記錄在 Notes 欄位供參考，但 Raw Severity 映射以 v3.1 等值為準。

工具提供 CVSS 分數時，使用分數範圍。僅提供標籤時，直接映射。兩者皆有且矛盾時，以 CVSS 分數為準。

**手動發現處理：**

使用者以自然語言描述發現時：
1. 詢問受影響的服務/端口（若未說明）。
2. 請使用者從四種類型中選擇暴露類型。
3. 詢問預估嚴重性（或提供 CVE 編號以推導）。
4. 若使用者提供 CVE，使用其 CVSS 分數作為 raw severity。

### Step 3 — 暴露彙整與登錄

去重、比對歷史記錄、指定 ID，並寫入持久化儲存。

#### 3a — 去重

- 同一 CVE 被多個工具發現 → 合併為一筆記錄；在 `Source Tool` 列出所有來源（例如「nuclei, nmap-script」）。
- 同一服務上的不同 CVE → 保持為獨立記錄。
- 不同工具以不同方式描述的相同問題（無 CVE，描述相似）→ 呈現給使用者，詢問是否需要合併。
- **同類別 info 級別發現**（例如同一服務的多個缺失安全標頭）：向使用者提供選擇：
  - **保持獨立** — 每項發現一筆暴露（最細粒度，適合詳細分析）
  - **合併** — 合併為單一暴露（例如「Missing Security Headers (5 items)」），個別項目列在 Notes 欄位（適合管理層報告）

  記錄使用者的選擇。若未表達偏好，預設為獨立記錄。

#### 3b — 跨 Session 比對（僅限回歸輪次）

讀取 asset 檔案的 Exposure Registry，與目前發現進行比對。

**比對基準：**
- 有 CVE → 以 `CVE + 受影響服務（端口）` 比對
- 無 CVE → 以 `標題 + 受影響服務（端口）` 近似比對 — 呈現給使用者確認

**狀態標記：**

比對始終基於 **Raw Severity**（工具報告的原始嚴重性）。不與 Prioritization 的調整後嚴重性比較 — 那是在 Exposure Registry 的 `Adjusted Severity` 欄位中單獨追蹤的。

| 狀態 | 條件 |
|------|------|
| `new` | Exposure Registry 中無匹配記錄 |
| `known` | 有匹配記錄，Raw Severity 不變 |
| `escalated` | 有匹配記錄，Raw Severity 較上次記錄的 Raw Severity 升高 |
| `reopened` | 有匹配記錄且 `Current Status = mitigated`，但本輪掃描再次偵測到 — 修復失效或已回歸 |
| `mitigated` | Registry 中有記錄，但本輪掃描未發現 — **需使用者確認** |

**mitigated 判定流程：**
1. 處理完所有目前發現後，列出 Registry 中存在但本輪未偵測到的暴露。
2. 向使用者呈現每一項：
   > *「以下來自先前輪次的暴露在本次掃描中未被偵測到。請逐項確認：此暴露是否已解決，還是可能仍然存在？」*
3. 使用者確認已解決 → 標記 `mitigated`
4. 使用者不確定或表示未解決 → 保持狀態為 `open`，加註：「Session [ID] 未偵測到但未確認已修復」

**首輪**：跳過整個步驟 — 所有發現皆為 `new`。

#### 3c — Exposure ID 指定

- 格式：`EXP-NNN`（三位數補零）。
- 範圍：Exposure ID 為 **per-asset**（每個資產檔案有獨立的編號序列）。在目前單主機實作中是明確的。若擴展至多主機場景，使用複合鍵 `<Asset ID>/<Exposure ID>`（例如 `ASSET-001/EXP-001`）確保唯一性。
- 掃描 asset 檔案的 Exposure Registry 判斷下一個可用編號。
- 若無既有記錄 → 從 `EXP-001` 開始。
- `known`、`escalated` 和 `reopened` 暴露 → **沿用**既有 Exposure ID。
- `new` 暴露 → 指定下一個可用 ID。

#### 3d — 寫入 Asset 檔案

更新 `reports/assets/<id>.md`：

**Exposure Registry 表：**
- `new` → 新增一行：Exposure ID、Title、First Seen = 目前 Session ID、Last Seen = 目前 Session ID、Severity History = 目前 severity、Current Status = `open`
- `known` → 更新 `Last Seen (Session)` 為目前 Session ID
- `escalated` → 更新 `Last Seen (Session)` + 附加至 `Severity History`（例如「medium (S-001) → high (S-002)」）
- `reopened` → 更新 `Current Status` 為 `reopened`，更新 `Last Seen (Session)`，若 Raw Severity 變更則附加至 `Severity History`
- `mitigated` → 更新 `Current Status` 為 `mitigated`，更新 `Last Seen (Session)`

**Last Assessed**：更新為目前 Session ID + 日期。

**不更新 Current Risk Summary** — 那是 Prioritization 的職責。

#### 3e — 寫入 Discovery Summary

將摘要寫入 `ctem-state.md` 的 `## Phase Summaries` 之下，標題為 `### Discovery Summary`。若已存在先前的 Discovery Summary（因回溯），**替換**而非附加。

---

## 產出：Discovery Summary

```markdown
### Discovery Summary

**Scan Date**: <ISO 8601 日期>
**Tools Used**: <工具名稱與版本>
**Input Method**: <paste / file / mixed>

#### Open Services Detected

| # | Port | Protocol | Service | Version | In-Scope |
|---|------|----------|---------|---------|----------|
| 1 | 22   | TCP      | SSH     | OpenSSH 8.9 | Yes |
| 2 | 80   | TCP      | HTTP    | Apache 2.4.52 | Yes |
| 3 | 3306 | TCP      | MySQL   | 8.0.30 | User-added |

#### Exposures Found

| # | Exposure ID | Title | Type | Raw Severity | CVE | Affected Service | Source Tool | Status |
|---|-------------|-------|------|-------------|-----|------------------|-------------|--------|
| 1 | EXP-001 | Apache Path Traversal | vulnerability | high | CVE-2021-41773 | HTTP/80 | nuclei | new |
| 2 | EXP-002 | SSH Weak Ciphers | misconfiguration | medium | — | SSH/22 | nmap-script | new |

#### Summary Statistics

| Metric | Value |
|--------|-------|
| Total Exposures | X |
| By Severity | critical: N, high: N, medium: N, low: N, info: N |
| By Status | new: N, known: N, escalated: N, reopened: N, mitigated: N |
| Unexpected Services Found | N |
```

此摘要是傳遞給第 3 階段（Prioritization）的**主要交接資料**。保持值簡潔且可機器解析。

---

## 完成 Checklist

結束前向使用者呈現此清單，所有項目必須勾選。

```
## Discovery 完成檢查清單

- [ ] 已讀取 Scoping Summary（確認目標與邊界）
- [ ] 已推薦掃描指令，使用者已執行掃描
- [ ] 已解析服務與端口掃描結果
- [ ] 已解析弱掃結果（或確認未使用弱掃工具）
- [ ] 所有暴露已分類並指定 raw severity
- [ ] Exposure Registry 已更新（reports/assets/）
- [ ] 超出範圍的服務已處理（納入、排除或回溯）
- [ ] Discovery Summary 已寫入 ctem-state.md
```

所有項目滿足後：

1. 更新 `ctem-state.md`：將 **Phase Status** 中 Discovery 該列設為 `completed`，填入 `Key Findings Summary` 和 `Last Updated` 欄位。
2. 在 Transition Log 新增一筆記錄。
3. 告知使用者 Discovery 已完成，可以進入第 3 階段。

範例結束訊息：
> **Discovery 完成。** 共發現 N 項暴露（critical: X, high: X, medium: X, low: X, info: X）。Exposure Registry 與 Discovery Summary 已寫入。準備好後可進入 Phase 3 — Prioritization。

---

## 參考資料（按需載入）

| 檔案 | 何時載入 | 優先級 |
|------|---------|---------|
| [tool-commands.md](./references/tool-commands.md) | Step 0 — 推薦掃描指令給使用者 | Step 0 開始時讀取 |
| [nmap-parsing.md](./references/nmap-parsing.md) | Step 1 或 Step 2 — 使用者提供 nmap 輸出需解析 | 收到 nmap 輸出時讀取 |
| [nuclei-parsing.md](./references/nuclei-parsing.md) | Step 2 — 使用者提供 Nuclei 輸出需解析 | 收到 Nuclei 輸出時讀取 |
