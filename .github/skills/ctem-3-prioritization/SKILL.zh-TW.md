# CTEM 第 3 階段 — 優先排序（Prioritization）— 中文閱讀版

> 此檔案為 `SKILL.md` 的繁體中文翻譯，僅供閱讀參考。AI 實際執行時讀取的是英文版 `SKILL.md`。

---

你是單一主機 CTEM 工作階段的優先排序分析師。
你的任務是**評估並排序** Phase 2 發現的每一項暴露，結合技術嚴重性與業務上下文 — 不只看 CVSS 分數，還要考量可利用性情報與補償控制。
你不執行技術驗證（不跑 PoC、不做滲透測試）— 那是 Phase 4 的職責。你負責評估、排序與記錄。

## 範圍限制聲明

本實作聚焦於**單一主機、基礎設施層級的優先排序**，採用三層評估模型：

| 層級 | 輸入 | 輸出 |
|------|------|------|
| **風險矩陣** | Raw Severity × Business Criticality | Base Adjusted Severity |
| **情境調整** | 可利用性情境 + 補償控制 | 淨調整（±1 級） |
| **最終結果** | Base + 淨調整 | Adjusted Severity（逐暴露） |

所有輸入皆來自 Scoping（Phase 1）和 Discovery（Phase 2）產出的結構化資料。不需要使用者提供外部工具輸出。

## 互動風格

採用**混合模式**：
1. 自動讀取所有上游階段資料，自動計算風險矩陣結果。
2. 以結構化表格呈現結果供使用者確認。
3. 可利用性分類：AI 提供建議與理由，使用者確認或覆寫。
4. 補償控制：一次性主機層級清單勾選。
5. 僅在使用者需要覆寫 AI 建議時才逐項互動。
6. 不重複詢問前序階段已確認的資訊。

---

## 先決條件

開始 Prioritization 工作流程前：

1. 從專案根目錄讀取 `ctem-state.md`。
2. 確認 Phase Status 表中 Discovery 狀態為 `completed`。若否 → **停止**並回報：*「必須先完成 Discovery 才能開始 Prioritization。」*
3. 將 Prioritization 狀態設為 `in_progress`。
4. 從 `ctem-state.md` 讀取 `### Scoping Summary` 以擷取：
   - Business Criticality（風險矩陣核心輸入）
   - Business Function（業務影響分析上下文）
   - Regulatory Context（合規性考量）
   - Attack Surface Boundary（範圍確認）
5. 從 `ctem-state.md` 讀取 `### Discovery Summary` 以擷取：
   - Exposures Found 表格（所有暴露的 Raw Severity、CVE、Affected Service、Type、Status）
   - Open Services Detected 表格（用於補償控制映射）
   - Summary Statistics（暴露統計概覽）
6. 從 `ctem-state.md` → `Session Info` 讀取 Session ID。
7. 讀取 `reports/assets/<hostname-or-ip>.md` 以擷取（檔名對應 Scoping Summary 中的 Target Host 或 Hostname）：
   - Exposure Registry（含歷史 Adjusted Severity，用於跨 session 比對）
   - Current Risk Summary（前輪風險狀態，用於趨勢比對）

### 首輪偵測

若 asset 檔案的 Exposure Registry 中**所有暴露均無 `Adjusted Severity` 值**（欄位為空），為**首輪**：
- 跳過跨 session Adjusted Severity 比對（Step 3）。
- Risk Changes 表標記為「N/A — first session」。
- Current Risk Summary 為首次寫入。

---

## 工作流程

### Step 0 — 上下文載入

在進行任何評估前，載入所有上游資料並向使用者呈現摘要。

**動作**：
1. 讀取 `ctem-state.md` → 確認 Discovery 狀態 `completed`。
2. 設定 Prioritization 狀態為 `in_progress`。
3. 讀取 Scoping Summary → 擷取 Business Criticality、Business Function、Regulatory Context。
4. 讀取 Discovery Summary → 擷取 Exposures Found 表格。
5. 讀取 `reports/assets/<id>.md` → 擷取 Exposure Registry。
6. 建立評估佇列：所有 Current Status 為 `open`、`new`、`known`、`escalated` 或 `reopened` 的暴露。
7. **零暴露檢查**：若評估佇列為空（無 open 暴露），跳過 Step 1–3。寫入一份空 Prioritized Exposure List 的 Prioritization Summary，設定 Overall Risk Level 為 `info`、Trend 為 `→ stable`（或 `→ first assessment`），直接進入 Step 4 產出。告知使用者：*「無 open 暴露需要評估。Prioritization Summary 將反映乾淨狀態。」*
8. 向使用者呈現摘要：

> *「本輪 Prioritization 將評估以下暴露：」*
>
> | # | Exposure ID | Title | Raw Severity | Type | Affected Service |
> |---|-------------|-------|-------------|------|------------------|
> | 1 | EXP-001 | ... | high | vulnerability | HTTP/80 |
>
> *「Business Criticality：\<level\>（來自 Scoping）。接下來將進行風險矩陣評估。」*

### Step 1 — 基線風險評估

用風險矩陣為每筆暴露計算 Base Adjusted Severity。

**在開始此步驟前，必須讀取 [risk-matrix.md](./references/risk-matrix.md)。** 該檔包含完整的矩陣定義、調整規則和補償控制參考。

**動作**：
1. 對每筆暴露，查表 `RiskMatrix[Raw Severity][Business Criticality]` → Base Adjusted Severity。
2. `info` 級暴露**不參與調整** — 維持 `info` 並排序至最底部。理由：info 級為資訊性揭露，不構成可利用風險。
3. 呈現結果表格供使用者確認：

> *「風險矩陣評估結果（Business Criticality = \<level\>）：」*
>
> | # | Exposure ID | Title | Raw Severity | Base Adjusted Severity |
> |---|-------------|-------|-------------|----------------------|
> | 1 | EXP-001 | ... | high | \<result\> |

此步驟為純機械式查表，不涉及主觀判斷。

### Step 2 — 情境調整

基於可利用性情境與補償控制，對 Base Severity 做最多 ±1 級調整。

#### Step 2a — 主機層級補償控制盤點（一次性）

呈現結構化控制清單，請使用者確認哪些控制已就位：

> *「請確認目標主機目前已部署哪些補償控制：」*
>
> | # | 控制項 | 說明 | 是否就位？ |
> |---|--------|------|-----------|
> | CC-01 | 網路分段 | 主機位於分段網路區域，存取受限 | Yes / No |
> | CC-02 | WAF | Web 應用防火牆，過濾惡意 HTTP/HTTPS 請求 | Yes / No |
> | CC-03 | IDS/IPS | 入侵偵測/防禦系統，監控主機流量 | Yes / No |
> | CC-04 | 防火牆規則 / ACL | 特定服務的存取限制（來源 IP 白名單等） | Yes / No |
> | CC-05 | MFA / 強認證 | 存取服務需多因子驗證 | Yes / No |
> | CC-06 | TLS / 傳輸加密 | 傳輸中資料加密 | Yes / No |
> | CC-07 | EDR / 端點防護 | 端點偵測與回應代理程式運行中 | Yes / No |

記錄使用者回答，建立主機控制盤點結果。

#### Step 2b — 可利用性分類 + 控制映射（逐暴露）

對每筆非 `info` 級暴露：

1. **可利用性分類**：根據 CVE 資訊或暴露特性建議三級分類之一，呈現建議理由供使用者確認或覆寫。

   | 等級 | 定義 | 調整 | 判斷依據 |
   |------|------|------|---------|
   | `confirmed-in-wild` | 已觀察到真實攻擊活動 | **+1** | 收錄於 CISA KEV、CVE 描述標註 "exploited in the wild"、廠商公告確認已觀察到攻擊 |
   | `poc-available` | 有公開的概念驗證程式碼，但未觀察到實際攻擊 | **0** | Exploit-DB 項目、GitHub PoC、Nuclei 模板標記為 verified |
   | `theoretical` | 僅有理論可利用性，無公開 PoC 或已知攻擊 | **−1** | CVE 描述僅述可能性、無公開利用程式碼、需特定條件才可觸發 |

   **評估流程**：
   - 有 CVE 的暴露：AI 根據 CVE 資訊建議分類，呈現理由。
   - 無 CVE 的暴露（misconfiguration、information-disclosure 等）：AI 根據暴露類型與特性建議。
   - 使用者可覆寫任何 AI 建議。

   **重要**：此步驟不做技術驗證（不執行 PoC、不做滲透測試）。僅基於公開情報判斷。技術驗證屬 Phase 4 職責。

2. **補償控制映射**：自動根據 Affected Service 和暴露 Type，從 Step 2a 的盤點中映射相關控制。

   **控制-暴露映射邏輯**：

   | 暴露 Affected Service | 可能相關的控制 |
   |----------------------|---------------|
   | HTTP/HTTPS 相關 | CC-02 (WAF)、CC-04 (ACL) |
   | HTTP/HTTPS — 資訊洩露或明文傳輸 | CC-06 (TLS) — 僅在暴露涉及資料攔截或明文傳輸時相關，不適用於應用層漏洞（如 path traversal 或 XSS） |
   | SSH 相關 | CC-04 (ACL)、CC-05 (MFA) |
   | 其他 TCP/UDP 服務 | CC-01 (網路分段)、CC-03 (IDS/IPS)、CC-04 (ACL) |
   | 主機層級（任何） | CC-01 (網路分段)、CC-03 (IDS/IPS)、CC-07 (EDR) |

   若自動映射結果不確定（例如非標準服務），呈現映射結果請使用者確認。

   **控制調整**：若該暴露有**至少一項**相關且已確認的補償控制 → **−1**；否則 → **0**。

3. **計算 Adjusted Severity**：套用最終調整公式（見下方）。

**呈現方式**：批次處理所有暴露，以單一表格呈現供使用者確認：

> *「以下為每筆暴露的情境評估建議，請確認或修改：」*
>
> | # | Exposure ID | Title | Base | 可利用性（建議） | 相關控制 | 淨調整 | Adjusted |
> |---|-------------|-------|------|-----------------|---------|--------|----------|
> | 1 | EXP-001 | ... | critical | confirmed-in-wild (+1) | 無 (0) | +1→capped | critical |
> | 2 | EXP-002 | ... | medium | poc-available (0) | WAF (-1) | -1 | low |

使用者可針對個別暴露修改可利用性分類或補償控制映射。

#### 最終調整公式

```
1. Base = RiskMatrix[Raw Severity][Business Criticality]
2. Exploitability Adj. = { confirmed-in-wild: +1, poc-available: 0, theoretical: -1 }
3. Controls Adj. = { has_relevant_control: -1, no_relevant_control: 0 }
4. Net = clamp(Exploitability Adj. + Controls Adj., -1, +1)
5. Adjusted Severity = clamp(Base + Net, info, critical)
   例外：若 Raw Severity == info → Adjusted Severity = info（跳過調整）
```

**嚴重性等級順序**（由低到高）：`info` < `low` < `medium` < `high` < `critical`

**Rationale 欄位**：每筆暴露都必須記錄調整推導過程，例如：`Base=high, +1 confirmed-in-wild, -1 WAF, net 0 → Adjusted high`

#### Step 2c — 確認最終 Adjusted Severity

所有調整完成後，呈現最終排序清單供使用者確認：

> *「最終 Adjusted Severity 結果：」*
>
> | Priority | Exposure ID | Title | Raw Severity | Adjusted Severity | Exploitability | Controls | Rationale |
> |----------|-------------|-------|-------------|-------------------|----------------|----------|-----------|
> | 1 | EXP-001 | ... | high | critical | confirmed-in-wild | 無 | Base=critical, +1 exploit, net +1→capped critical |
> | 2 | EXP-002 | ... | medium | low | poc-available | WAF | Base=medium, 0 exploit, -1 WAF, net -1 |
>
> *（按 Adjusted Severity 由高到低排序，同級依 Raw Severity 排序）*

### Step 3 — 跨 Session 風險比對

將本輪 Adjusted Severity 與前輪比對，偵測風險升降。

**首輪模式**：完全跳過此步驟。Risk Changes 標記為「N/A — first session」。

**回歸輪次動作**：
1. 從 Asset Profile 的 Exposure Registry 讀取每筆暴露的前輪 `Adjusted Severity`。
2. 與本輪 Adjusted Severity 比較。
3. 分類變化：

   | 變化類型 | 條件 | 說明 |
   |---------|------|------|
   | `escalated` | 本輪 Adjusted > 前輪 Adjusted | 風險升級 |
   | `de-escalated` | 本輪 Adjusted < 前輪 Adjusted | 風險降級 |
   | `unchanged` | 本輪 Adjusted = 前輪 Adjusted | 無變化 |

4. 呈現風險變化摘要：

> *「跨 Session 風險變化：」*
>
> | # | Exposure ID | Title | 前輪 Adjusted | 本輪 Adjusted | 變化 | 原因 |
> |---|-------------|-------|---------------|---------------|------|------|
> | 1 | EXP-001 | ... | medium (S-001) | critical (S-002) | ↑ escalated | 可利用性升級為 confirmed-in-wild |

   **變化原因**：記錄驅動變化的主要因素。常見原因包括：
   - "Business Criticality changed from X to Y"（Scoping 重新評估）
   - "Raw Severity escalated from X to Y"（Discovery 重新掃描）
   - "Exploitability upgraded to confirmed-in-wild"（新興威脅情報）
   - "Compensating control removed/added"（環境變更）
   - "New exposure in this session"（本 session 新增的暴露，無前輪 Adjusted Severity）

> **注意**：此處比對的是 **Adjusted Severity**（業務風險調整後的等級），與 Discovery Step 3b 比對 **Raw Severity**（工具原始評級）不同。兩者為不同維度的互補比較。

### Step 4 — 寫入與產出

將評估結果寫入所有目標位置。

#### 4a — 寫入 Asset Profile (`reports/assets/<id>.md`)

**Exposure Registry 表**：
- 更新每筆暴露的 `Adjusted Severity` 欄位：格式為 `<level> (S-XXX)`，例如 `high (S-002)`。
- 首輪：首次寫入 Adjusted Severity。
- 回歸輪次：覆寫為本輪最新結果。（Raw Severity 的歷史追蹤由 Discovery 維護的 `Severity History` 欄位負責；Adjusted Severity 只保留最新值。）

**Current Risk Summary 表**：
- `Overall Risk Level`：所有 open 暴露中最高的 Adjusted Severity。
- `Open Exposures`：status 為 `open` / `new` / `known` / `escalated` / `reopened` 的暴露數量。
- `Highest Severity`：同 Overall Risk Level。
- `Trend`：與前輪 Overall Risk Level 比較：
  - 本輪 > 前輪 → `↑ escalating`
  - 本輪 = 前輪 → `→ stable`
  - 本輪 < 前輪 → `↓ improving`
  - 首輪 → `→ first assessment`

#### 4b — 寫入 Prioritization Summary (`ctem-state.md`)

寫入 `## Phase Summaries` 下方，作為 `### Prioritization Summary`。若已存在前次的 Prioritization Summary（因回溯），**整個替換**。

#### 4c — 更新 Phase Status (`ctem-state.md`)

- 設定 Prioritization 為 `completed`。
- 填入 `Key Findings Summary` 和 `Last Updated`。
- 新增 Transition Log 項目。

---

## 輸出：Prioritization Summary

```markdown
### Prioritization Summary

**Assessment Date**: <ISO 8601 日期>
**Business Criticality**: <來自 Scoping>
**Scoring Model**: Risk Matrix (Raw Severity × Business Criticality) ± Contextual Adjustment

#### Host-Level Compensating Controls

| # | Control | In Place |
|---|---------|----------|
| CC-01 | Network Segmentation | Yes/No |
| CC-02 | WAF | Yes/No |
| CC-03 | IDS/IPS | Yes/No |
| CC-04 | Firewall Rules / ACL | Yes/No |
| CC-05 | MFA / Strong Auth | Yes/No |
| CC-06 | TLS / Encryption | Yes/No |
| CC-07 | EDR / Endpoint Protection | Yes/No |

#### Prioritized Exposure List

| Priority | Exposure ID | Title | Raw Severity | Adjusted Severity | Exploitability | Controls Applied | Rationale |
|----------|-------------|-------|-------------|-------------------|----------------|-----------------|-----------|
| 1 | EXP-001 | Apache Path Traversal | high | critical | confirmed-in-wild (+1) | None (0) | Base=critical, net +1→capped critical |
| 2 | EXP-002 | SSH Weak Ciphers | medium | low | theoretical (-1) | ACL (-1) | Base=medium, net -1 (capped) |

#### Risk Changes (Cross-Session)

<!-- 首輪填 "N/A — first session" -->

| # | Exposure ID | Previous Adjusted | Current Adjusted | Change | Reason |
|---|-------------|-------------------|------------------|--------|--------|
| 1 | EXP-001 | medium (S-001) | critical (S-002) | ↑ escalated | Exploitability upgraded to confirmed-in-wild |

#### Risk Overview

| Metric | Value |
|--------|-------|
| Total Exposures Assessed | X |
| Adjusted Severity Distribution | critical: N, high: N, medium: N, low: N, info: N |
| Risk Changes | N escalated, N de-escalated, N unchanged |
| Overall Risk Level | <所有 open 暴露中最高的 adjusted severity> |
| Trend | ↑ escalating / → stable / ↓ improving / → first assessment |
| Top Priority Action | <最高優先暴露的簡要建議> |
```

此摘要為 Phase 4（Validation）的**主要交接資料**。保持值簡潔且可機器解析。

### 法規情境於建議中的應用

若 Scoping Summary 包含非空的 `Regulatory Context`（例如 PCI-DSS、GDPR、ISO 27001），應將此反映於 `Top Priority Action` 欄位及影響法規範圍內服務的暴露 Rationale 中。例如：
- 若最高優先暴露影響付款處理服務且 Regulatory Context 包含 PCI-DSS，註記：*「PCI-DSS 範圍內 — 修復時程可能受合規期限約束。」*
- 若未定義 Regulatory Context（N/A），則省略法規相關引用。

法規情境**不會改變**風險矩陣分數或情境調整 — 它為修復急迫性與合規報告提供額外上下文。

---

## 術語說明

- **Severity** 等級使用 `medium`（對齊 CVSS v3.1 慣例）。
- **Business Criticality** 等級使用 `moderate`（對齊 FIPS 199 / Scoping 慣例）。
- 兩者為不同維度的命名慣例，不互通。

---

## 完成 Checklist

在結束前向使用者呈現此 checklist。每個項目都必須勾選。

```
## Prioritization Completion Checklist

- [ ] Scoping Summary 與 Discovery Summary 已讀取
- [ ] 所有暴露已通過風險矩陣計算 Base Adjusted Severity
- [ ] 補償控制已盤點（主機層級控制清單）
- [ ] 每筆暴露的可利用性已分類（使用者已確認）
- [ ] 情境調整已套用，最終 Adjusted Severity 已確定（使用者已確認）
- [ ] 跨 Session 風險比對已完成（或確認為首輪）
- [ ] Asset Profile 的 Adjusted Severity 與 Current Risk Summary 已更新（reports/assets/）
- [ ] Prioritization Summary 已寫入 ctem-state.md
```

所有項目滿足後：

1. 更新 `ctem-state.md`：將 Phase Status 中 Prioritization 設為 `completed`，填入 `Key Findings Summary` 和 `Last Updated`。
2. 新增 Transition Log 項目。
3. 通知使用者 Prioritization 已完成，可進入 Phase 4。

結束訊息範例：
> **Prioritization 完成。** 共評估 N 項暴露（Adjusted Severity — critical: X, high: X, medium: X, low: X, info: X）。Overall Risk Level: \<level\>。Prioritized Exposure List 與 Prioritization Summary 已寫入。準備好後可進入 Phase 4 — Validation。

---

## 參考檔案（按需載入）

| 檔案 | 載入時機 | 優先度 |
|------|---------|--------|
| [risk-matrix.md](./references/risk-matrix.md) | **必須在 Step 1 前讀取** — 包含完整風險矩陣、調整規則、補償控制定義與排序規則 | **必要** — Step 1 前 |
