# CTEM Phase 2 — Discovery Skill 完整設計稿

> 本文件統整所有討論決策，作為 `ctem-2-discovery/SKILL.md` 的建置藍圖。

---

## 一、設計前提

- 流程無限貼近 Gartner CTEM 框架（Gartner, 2022, ID: G00766755）
- 本版為**簡單版**：單一機器 per session（多機器/多線程列入未來 TODO）
- Skill 保持獨立，流程控制回歸 `ctem-flow`
- 所有評估框架必須有依據並附來源（論文需求）
- Discovery 的職責：**從 Scoping 定義的邊界內，系統性地識別所有暴露（exposures）**
- Discovery 不做業務上下文排序（那是 Prioritization 的職責），但提供 raw severity 作為初步評級

---

## 二、與 Scoping 的銜接

### 2.1 讀取上游資訊

Discovery 啟動時：
1. 讀取 `ctem-state.md` → 確認 Scoping 狀態為 `completed`（依 ctem-state-protocol）
2. 讀取 `ctem-state.md` → `### Scoping Summary` 取得：
   - Target Host / Hostname
   - OS / Platform
   - In-Scope Services（決定掃描目標）
   - Out-of-Scope（避免掃到範圍外）
   - Attack Surface Boundary
   - Business Criticality（作為後續 context 參考）
3. 讀取 `reports/assets/<id>.md` 取得 Exposure Registry（用於跨 session 比對）

### 2.2 先決條件

- Scoping 必須為 `completed` 才能啟動 Discovery
- 若先決條件不滿足 → STOP，回報缺少的項目

---

## 三、輸入模式

### 返回 Session 行為：每輪完整重跑（選項 A）

無論首輪或返回輪，**都要求使用者提供新的掃描結果**。理由：
- Discovery 的價值在於發現最新狀態
- 目標環境可能已變更
- 如果使用者想跳過 Discovery，由 ctem-flow 層級處理（非 Discovery skill 職責）

返回輪的額外動作：
- 將本輪發現與 asset 檔案中既有 Exposure Registry 比對
- 標記每筆暴露的狀態（new / known / escalated / mitigated）

### 首輪偵測

若 asset 檔案的 Exposure Registry 為空（無既有暴露記錄）→ 首輪模式：
- 跳過所有跨 session 比對
- 所有發現的暴露一律標記為 `new`

---

## 四、掃描結果輸入方式：混合模式

支援兩種輸入方式，Skill 自動偵測：

1. **直接貼上（paste）**：使用者直接貼 terminal 輸出文字到對話中
2. **檔案路徑讀取**：使用者提供掃描結果檔案的路徑（例如 `nmap -oX scan.xml` 產出的檔案），skill 用 `read_file` 讀取

偵測邏輯：
- 若使用者訊息中包含檔案路徑格式（如 `/path/to/file`、`~/scan.xml`）→ 嘗試 read_file
- 否則 → 視為直接貼上的輸出文字

---

## 五、V1 支援的工具範圍

| 工具 | 輸出類型 | V1 狀態 | 說明 |
|------|---------|---------|------|
| **nmap** (-sV, -sC, --script vuln) | 服務版本 + NSE 弱掃 | ✅ 必要 | 服務偵測與基礎弱掃 |
| **Nuclei** | 弱掃結果（模板驅動） | ✅ 必要 | 高覆蓋率弱掃 |
| **手動發現** | 使用者自行描述的暴露 | ✅ 必要 | 任何來源的保底機制 |
| Nessus | 商業弱掃報告 | ❌ 未來版本 | — |
| OpenVAS | 開源弱掃報告 | ❌ 未來版本 | — |
| nikto | Web 伺服器弱掃 | ❌ 未來版本 | — |

---

## 六、執行步驟

### Step 0：掃描規劃（Scan Planning）

**目的**：根據 Scoping Summary 推薦使用者該執行的掃描指令。

**動作**：
1. 讀取 Scoping Summary 的 In-Scope Services、Target Host、OS/Platform
2. 根據目標特性，從 `references/tool-commands.md` 推薦適合的掃描指令組合
3. 明確告知使用者需要執行哪些掃描並將結果回傳

**推薦邏輯**：
- 一律推薦 nmap 服務掃描（`-sV -sC`）+ 弱掃腳本（`--script vuln`）
- 一律推薦 Nuclei 掃描
- 根據 In-Scope Services 調整 nmap 端口範圍（如已知端口清單可用 `-p` 指定）
- 若 OS 為 Web 伺服器角色 → 額外推薦 Nuclei 的 web-specific 模板

**互動方式**：
- 列出建議指令
- 使用者可以只跑部分工具（例如只有 nmap 沒有 Nuclei）→ skill 接受並繼續
- 使用者也可以額外提供其他工具輸出 → skill 以「手動發現」方式處理

### Step 1：服務與端口分析（Service & Port Analysis）

**目的**：解析 nmap 等工具輸出，確認開放服務與版本，並與 Scoping 邊界比對。

**動作**：
1. 解析使用者提供的 nmap 輸出（文字或檔案）
2. 擷取每個開放端口的：Port、Protocol、Service Name、Version
3. 與 Scoping Summary 的 In-Scope Services 比對：
   - 匹配的 → 標記 `In-Scope = Yes`
   - 不在 Scoping 中的 → 標記 `In-Scope = Unexpected`

**超出範圍的服務處理（決策 10 — 選項 B）**：
- 發現 Unexpected services 時 → **暫停流程，主動提醒使用者**
- 呈現 Unexpected services 清單
- 詢問使用者：
  > 「以下服務不在 Scoping 的 In-Scope 清單中。請選擇：」
  > 1. 將此服務加入本次評估範圍（Discovery 繼續包含此服務）
  > 2. 確認排除（Discovery 不評估此服務的暴露）
  > 3. 回溯到 Scoping 重新調整範圍
- 記錄使用者的決定
- Discovery **不自行修改 Scoping Summary** — 若需要正式 backtrack，由使用者/ctem-flow 決定

**產出**：Open Services 表格（用於 Discovery Summary）

### Step 2：弱點與暴露識別（Vulnerability & Exposure Identification）

**目的**：解析弱掃工具輸出，識別所有暴露並分類。

**動作**：
1. 解析 Nuclei 輸出（若有提供）
2. 解析 nmap NSE script 輸出中的弱掃結果（若使用 `--script vuln`）
3. 接受使用者手動描述的暴露
4. 對每筆暴露進行：
   - **分類**（Type）
   - **初步嚴重性評級**（Raw Severity）
   - **關聯**到對應的 Port/Service

**暴露類型分類（Exposure Type）**：

| Type | 說明 | 範例 |
|------|------|------|
| `vulnerability` | 已知 CVE 或可利用弱點 | CVE-2024-XXXX |
| `misconfiguration` | 不安全的設定 | SSH weak ciphers, default credentials |
| `information-disclosure` | 資訊洩露 | 版本資訊暴露、目錄列表、錯誤訊息洩露 |
| `outdated-software` | 已知過時版本（即使無特定 CVE） | Apache 2.4.49（已有新版） |

**Raw Severity 評級**：

基於工具輸出的原始嚴重性，不加入業務上下文調整（那是 Prioritization 的工作）。

| Raw Severity | 來源依據 |
|-------------|---------|
| critical | CVSS 9.0–10.0 或工具標記為 critical |
| high | CVSS 7.0–8.9 或工具標記為 high |
| medium | CVSS 4.0–6.9 或工具標記為 medium |
| low | CVSS 0.1–3.9 或工具標記為 low |
| info | 純資訊揭露，CVSS 0 或工具標記為 info |

CVSS 參考：FIRST CVSS v3.1 Specification（https://www.first.org/cvss/v3.1/specification-document）

**手動發現處理**：
- 使用者以自然語言描述暴露
- Skill 提問補齊：受影響的服務/端口、暴露類型、預估嚴重性
- 若使用者提供 CVE 編號 → 直接使用對應 CVSS 分數

### Step 3：暴露彙整與登錄（Exposure Consolidation & Registration）

**目的**：去重、合併、比對歷史記錄，寫入 asset 檔案與 Discovery Summary。

**動作**：

#### 3a. 去重與合併
- 同一 CVE 被多個工具發現 → 合併為單筆，Source Tool 列出所有來源
- 同一服務的相關暴露保持獨立記錄（不合併不同 CVE）

#### 3b. 跨 Session 暴露比對（返回輪限定）

比對基準：
- 有 CVE → 以 CVE 編號 + Port/Service 為 matching key
- 無 CVE → 以 Title + Port/Service 近似比對，**需使用者確認**

狀態標記：

| Status | 條件 |
|--------|------|
| `new` | 本輪首次發現，asset 檔案 Exposure Registry 無匹配記錄 |
| `known` | asset 檔案已有記錄，severity 不變 |
| `escalated` | asset 檔案已有記錄，severity 升高 |
| `mitigated` | asset 檔案有記錄但本輪掃描未再現 — **需使用者確認** |

**mitigated 判定流程**：
1. 列出「上輪有但本輪未見」的暴露清單
2. 詢問使用者：每筆是否確認已修復？
3. 使用者確認 → 標記 `mitigated`
4. 使用者不確認 → 保留為 `open`，在 Notes 加註「本輪未偵測到但未確認修復」

首輪模式（Exposure Registry 為空）：跳過比對，全部標記 `new`。

#### 3c. 指定 Exposure ID
- 格式：`EXP-NNN`（零填充三位數）
- 掃描 asset 檔案的 Exposure Registry 判斷下一個可用編號
- 若無既有記錄 → 從 `EXP-001` 開始
- `known` 和 `escalated` 狀態的暴露 → 沿用既有 Exposure ID
- `new` 狀態的暴露 → 指定新 Exposure ID

#### 3d. 寫入 Asset 檔案

更新 `reports/assets/<id>.md` 的以下區塊：

- **Exposure Registry 表**：
  - `new` → 新增一行
  - `known` → 更新 `Last Seen (Session)` 欄位
  - `escalated` → 更新 `Last Seen (Session)` + `Severity History` 欄位
  - `mitigated` → 更新 `Current Status` 為 `mitigated`
- **Last Assessed**：更新為當前 Session ID + 日期

**不寫入 Current Risk Summary** — 由 Prioritization 階段負責。

#### 3e. 寫入 Discovery Summary（至 ctem-state.md）

格式見「七、資訊傳遞機制」。

---

## 七、資訊傳遞機制

### 7.1 持久化位置

| 位置 | 內容 | 生命週期 |
|------|------|---------|
| `ctem-state.md` → `## Phase Summaries` 下的 `### Discovery Summary` | 本輪 Discovery 結果摘要 | 單輪 session |
| `reports/assets/<id>.md` → Exposure Registry | 暴露完整記錄 | 跨輪次長期維護 |

### 7.2 Discovery Summary 格式（寫入 ctem-state.md）

```markdown
### Discovery Summary

**Scan Date**: <ISO 8601 日期>
**Tools Used**: <工具名稱 + 版本>
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
| 1 | EXP-001 | Apache CVE-2024-XXXX | vulnerability | high | CVE-2024-XXXX | HTTP/80 | nuclei | new |
| 2 | EXP-002 | SSH weak ciphers | misconfiguration | medium | — | SSH/22 | nmap-script | new |

#### Summary Statistics

| Metric | Value |
|--------|-------|
| Total Exposures | X |
| By Severity | critical: N, high: N, medium: N, low: N, info: N |
| By Status | new: N, known: N, escalated: N, mitigated: N |
| Unexpected Services Found | N (detail above) |
```

### 7.3 下游階段如何取得資訊

- **Prioritization** 啟動時 → 讀取 `ctem-state.md` 的 `### Discovery Summary` → 取得暴露清單、raw severity、狀態
- 需要更多細節 → 讀取 `reports/assets/<id>.md` 的 Exposure Registry

---

## 八、完成條件與 Checklist

Discovery skill 在所有步驟做完後，產出以下 checklist：

```markdown
## Discovery Completion Checklist

- [ ] Scoping Summary 已讀取（確認目標與邊界）
- [ ] 掃描指令已推薦並由使用者執行
- [ ] 服務與端口掃描結果已解析
- [ ] 弱掃結果已解析（或確認無弱掃工具使用）
- [ ] 所有暴露已分類並指定 raw severity
- [ ] Exposure Registry 已更新（reports/assets/）
- [ ] 超出範圍的服務已處理（標記或確認排除）
- [ ] Discovery Summary 已寫入 ctem-state.md
```

**職責劃分**：
- Discovery skill：執行分析、產出 checklist、寫入 Exposure Registry 與 Discovery Summary
- ctem-flow：驗證 checklist、更新 Phase Status、執行階段轉換、Backtrack Check

---

## 九、互動風格

採**混合模式**（與 Scoping 一致）：
1. 接受使用者在訊息中直接提供的掃描結果（貼上或檔案路徑）
2. 自動解析已提供的輸出
3. 僅針對缺失的必要資訊提問
4. 不重複詢問已知資訊

額外互動原則：
- 解析結果以結構化表格呈現，讓使用者確認
- 遇到不確定的解析結果（例如無法判斷 severity）→ 明確標出並詢問使用者
- 手動發現的暴露 → 以引導式問答補齊所需欄位

---

## 十、參考工具指令與解析指南

獨立存放於 `references/` 目錄，SKILL.md 透過相對連結引用。

### 10.1 references/tool-commands.md

Discovery 階段的掃描指令參考，包含：
- nmap 服務掃描 + 弱掃腳本指令
- Nuclei 掃描指令與常用模板分類
- 輸出格式建議（便於解析的輸出選項）

### 10.2 references/nmap-parsing.md

nmap 輸出的解析規則，包含：
- 標準文字輸出格式解析（`-sV -sC` 輸出）
- NSE script 弱掃結果解析（`--script vuln` 輸出）
- XML 格式解析（`-oX` 輸出）— 若使用者提供檔案
- 如何從 nmap 輸出提取：Port/Service/Version、CVE、弱掃發現

### 10.3 references/nuclei-parsing.md

Nuclei 輸出的解析規則，包含：
- 標準 terminal 輸出格式解析
- JSON 輸出格式解析（`-json` 輸出）— 若使用者提供檔案
- Nuclei severity 等級到 raw severity 的映射
- 如何從 Nuclei 輸出提取：Template ID、CVE、Severity、受影響 URL/Service

---

## 十一、檔案影響範圍

建置 Discovery skill 時需要建立/修改的檔案：

| 檔案 | 動作 | 說明 |
|------|------|------|
| `.github/skills/ctem-2-discovery/SKILL.md` | 重寫 | 主要 prompt 邏輯 |
| `.github/skills/ctem-2-discovery/SKILL.zh-TW.md` | 新增 | 繁體中文版（輔助閱讀） |
| `.github/skills/ctem-2-discovery/references/tool-commands.md` | 新增 | 掃描指令參考 |
| `.github/skills/ctem-2-discovery/references/tool-commands.zh-TW.md` | 新增 | 繁體中文版 |
| `.github/skills/ctem-2-discovery/references/nmap-parsing.md` | 新增 | nmap 輸出解析指南 |
| `.github/skills/ctem-2-discovery/references/nmap-parsing.zh-TW.md` | 新增 | 繁體中文版 |
| `.github/skills/ctem-2-discovery/references/nuclei-parsing.md` | 新增 | Nuclei 輸出解析指南 |
| `.github/skills/ctem-2-discovery/references/nuclei-parsing.zh-TW.md` | 新增 | 繁體中文版 |

> 注意：`ctem-state.md` 和 `ctem-state-protocol.instructions.md` 不需要修改 — 已有 `### Discovery Summary` 的預留位置和通用規範。

---

## 十二、與 ctem-flow 的互動點

Discovery skill 完成後，ctem-flow 的 Backtrack Check 會檢查：

| 檢查項 | 來源 | 處理 |
|--------|------|------|
| 4a — 新資產不在 Scoping | Discovery 不會觸發此項（單一 host） | N/A（未來多 host 版本才需要） |
| 4b — 新暴露不在 Discovery | 後續階段若發現新暴露 → backtrack 回 Discovery | ctem-flow 負責 |
| 4c — 風險側寫顯著變化 | 比較 asset 的 Severity History | ctem-flow 負責，Discovery 提供 Status 欄位（escalated 等） |

Discovery skill 在 Summary 中提供 `Status` 欄位（new/known/escalated/mitigated），讓 ctem-flow 有足夠資訊判斷是否需要 backtrack。

---

## 十三、參考來源

| 來源 | 用途 |
|------|------|
| Gartner, 2022 — *Implement a CTEM Program* (G00766755) | CTEM 框架定義、Discovery 階段職責 |
| FIRST — *CVSS v3.1 Specification Document* | Raw Severity 分級依據（CVSS base score 映射） |
| nmap.org — *Nmap Reference Guide* | nmap 指令參考與輸出格式 |
| ProjectDiscovery — *Nuclei Documentation* | Nuclei 指令參考與輸出格式 |
| NIST NVD — *National Vulnerability Database* | CVE 資訊查詢 |
| OWASP — *Testing Guide v4* | 暴露類型分類參考 |

---

## 十四、設計決策記錄

以下記錄本設計稿的所有關鍵決策及理由，供後續追溯：

| # | 決策 | 結果 | 理由 |
|---|------|------|------|
| 1 | 步驟數量 | 4 步驟（含 Step 0 Scan Planning） | 使用者可能不知道該用什麼工具，與 Scoping 的 tool-commands 模式一致 |
| 2 | V1 工具範圍 | nmap + Nuclei + 手動發現 | 開源工具鏈覆蓋面最廣，手動發現保底 |
| 3 | Exposure 記錄位置 | 嵌入 Asset 檔案的 Exposure Registry | 單一 host 暴露數量有限，降低檔案管理複雜度 |
| 4 | Discovery 給 raw severity | 是 | 基於工具輸出的原始嚴重性，Prioritization 再做業務上下文調整 |
| 5 | 暴露類型分類 | 4 類（vulnerability / misconfiguration / information-disclosure / outdated-software） | 覆蓋常見場景且對應 Gartner exposure 分類 |
| 6 | references 拆分 | 按工具拆成多個檔案（nmap-parsing.md、nuclei-parsing.md） | 各工具解析邏輯獨立，方便維護與擴展 |
| 7 | 返回 Session 行為 | 每輪完整重跑 | Discovery 的價值在於發現最新狀態，跳過由 ctem-flow 處理 |
| 8 | 跨 session 暴露比對 | CVE + Port/Service 為 matching key，mitigated 需使用者確認 | 掃不到可能是掃描參數不同，不應自動假設已修復 |
| 9 | Current Risk Summary 寫入者 | Prioritization（非 Discovery） | 涉及業務上下文排序，是 Prioritization 職責 |
| 10 | 超出 scope 的服務處理 | 暫停提醒使用者，提供三選項 | 重要安全決策不該靜默通過，即時處理比事後 backtrack 效率高 |
| 11 | 掃描結果輸入方式 | 混合模式（貼上 + 檔案路徑） | 靈活性最高，短輸出貼方便、大檔案用路徑 |
| 12 | Completion Checklist | 8 項（見第八節） | 覆蓋所有關鍵產出，與 Scoping checklist 模式一致 |
