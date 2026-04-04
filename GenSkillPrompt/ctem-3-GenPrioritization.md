# CTEM Phase 3 — Prioritization Skill 完整設計稿

> 本文件統整所有討論決策，作為 `ctem-3-prioritization/SKILL.md` 的建置藍圖。

---

## 一、設計前提

- 流程無限貼近 Gartner CTEM 框架（Gartner, 2022, ID: G00766755）
- 本版為**簡單版**：單一機器 per session（多機器/多線程列入未來 TODO）
- Skill 保持獨立，流程控制回歸 `ctem-flow`
- 所有評估框架必須有依據並附來源（論文需求）
- Prioritization 的職責：**基於業務上下文，對 Discovery 發現的暴露進行風險評估與排序**
- Prioritization 不做技術驗證（那是 Validation 的職責），但評估可利用性情境作為排序參考
- 核心原則（Gartner）：Prioritization 不只看 CVSS，必須結合業務影響、可利用性、補償控制來排序

---

## 二、與上游的銜接

### 2.1 讀取上游資訊

Prioritization 啟動時：
1. 讀取 `ctem-state.md` → 確認 Discovery 狀態為 `completed`（依 ctem-state-protocol）
2. 讀取 `ctem-state.md` → `### Scoping Summary` 取得：
   - Business Criticality（風險矩陣的核心輸入）
   - Business Function（業務影響分析的上下文）
   - Regulatory Context（合規性影響考量）
   - Attack Surface Boundary（評估範圍確認）
3. 讀取 `ctem-state.md` → `### Discovery Summary` 取得：
   - Exposures Found 表格（所有暴露清單、Raw Severity、CVE、Affected Service、Status）
   - Open Services Detected 表格（服務清單，用於補償控制映射）
   - Summary Statistics（暴露統計概覽）
4. 讀取 `reports/assets/<id>.md` 取得：
   - Exposure Registry（含歷史 Adjusted Severity，用於跨 session 比對）
   - Current Risk Summary（前輪風險狀態，用於趨勢比對）

### 2.2 先決條件

- Discovery 必須為 `completed` 才能啟動 Prioritization
- 若先決條件不滿足 → STOP，回報缺少的項目

### 2.3 首輪偵測

若 Asset Profile 的 Exposure Registry 中所有暴露均無 `Adjusted Severity` 記錄（欄位為空）→ 首輪模式：
- 跳過跨 session Adjusted Severity 比對（Step 3）
- Risk Changes 表格留空
- Current Risk Summary 為首次寫入

---

## 三、輸入模式

Prioritization **不需要使用者提供外部工具輸出**。所有輸入皆來自前序階段已寫入的結構化資料：

| 資料來源 | 資訊 | 用途 |
|----------|------|------|
| Scoping Summary | Business Criticality | 風險矩陣 Y 軸 |
| Discovery Summary | Exposures Found (Raw Severity) | 風險矩陣 X 軸 |
| Discovery Summary | Exposures Found (CVE) | 可利用性分類參考 |
| Discovery Summary | Exposures Found (Type, Affected Service) | 補償控制映射 |
| Asset Profile | Exposure Registry (Adjusted Severity) | 跨 session 比對 |
| Asset Profile | Current Risk Summary | 趨勢比對 |

Prioritization 需要使用者**確認或決定**的事項：
- 每筆暴露的可利用性分類（AI 建議，使用者確認）
- 主機層級的補償控制盤點（使用者勾選）
- 最終排序結果的確認

---

## 四、評分模型

### 4.1 三層評估架構

```
Layer 1: Risk Matrix           Layer 2: Contextual Adjustment     Layer 3: Final
┌──────────────────────┐      ┌────────────────────────────┐     ┌──────────────────┐
│ Raw Severity         │      │ Exploitability Context      │     │ Adjusted Severity│
│ × Business Criticality│ ──→ │ + Compensating Controls     │ ──→ │ (per exposure)   │
│ = Base Severity      │      │ = Net Adjustment (±1 max)   │     │                  │
└──────────────────────┘      └────────────────────────────┘     └──────────────────┘
```

### 4.2 Layer 1 — 風險矩陣（簡單查表）

以 **Raw Severity**（Discovery 提供）為 X 軸、**Business Criticality**（Scoping 提供）為 Y 軸，查表得出 **Base Adjusted Severity**。

矩陣定義詳見 `references/risk-matrix.md`。

**等級順序**（由低到高）：`info` < `low` < `medium` < `high` < `critical`

> 注意：Severity 等級用 `medium`（對齊 CVSS v3.1 慣例），Business Criticality 用 `moderate`（對齊 FIPS 199 / Scoping 慣例）。兩者為不同維度的命名，不互通。

### 4.3 Layer 2 — 情境調整

基於兩個因子對 Base Severity 做 ±1 級調整：

#### 4.3.1 可利用性情境（Exploitability Context）

三級定性分類，每筆暴露逐一評估：

| 等級 | 定義 | 調整 | 判斷依據 |
|------|------|------|---------|
| `confirmed-in-wild` | 已有真實攻擊活動或已被積極利用 | **+1** | CISA KEV 收錄、CVE 描述標註 "exploited in the wild"、廠商公告已觀察到攻擊 |
| `poc-available` | 有公開的概念驗證（PoC）程式碼，但未觀察到實際攻擊 | **0** | Exploit-DB、GitHub PoC、Nuclei 模板標記 verified |
| `theoretical` | 僅有理論可利用性，無公開 PoC 或已知攻擊 | **−1** | CVE 描述僅述可能性、無公開利用程式碼、需特定條件才可觸發 |

**評估流程**：
1. 對有 CVE 的暴露：AI 根據 CVE 資訊建議分類，呈現給使用者確認
2. 對無 CVE 的暴露（misconfiguration、information-disclosure 等）：由 AI 根據暴露類型建議，使用者確認
3. 使用者可覆寫 AI 建議

**注意**：此步驟不做技術驗證（不執行 PoC、不做滲透測試）。僅基於公開情報判斷。技術驗證屬 Phase 4 (Validation) 職責。

#### 4.3.2 補償控制（Compensating Controls）

採結構化控制清單，分**主機層級盤點**與**逐暴露映射**兩步進行。

**主機層級控制清單**（一次性盤點）：

| # | 控制類別 | 說明 | 適用服務類型 |
|---|---------|------|-------------|
| CC-01 | Network Segmentation | 主機位於網路分段區域，存取受限 | 所有 |
| CC-02 | WAF (Web Application Firewall) | 過濾惡意 HTTP/HTTPS 請求 | Web 服務 (HTTP/HTTPS) |
| CC-03 | IDS/IPS | 入侵偵測/防禦系統監控主機流量 | 所有 |
| CC-04 | Firewall Rules / ACL | 特定服務的存取限制（來源 IP 白名單等） | 特定端口/服務 |
| CC-05 | MFA / Strong Authentication | 存取服務需多因子驗證 | 認證類服務 (SSH, Admin Panel) |
| CC-06 | TLS / Encryption in Transit | 傳輸資料加密 | 傳輸敏感資料的服務 |
| CC-07 | EDR / Endpoint Protection | 端點偵測與回應代理程式運行中 | 主機層級 |

**評估流程**：
1. **Step 2a — 主機層級盤點**：呈現控制清單，請使用者逐項確認 Yes / No
2. **Step 2b — 逐暴露映射**（自動）：根據每筆暴露的 Type 和 Affected Service，自動判斷哪些已確認的控制與之相關
3. **調整規則**：若該暴露有**至少一項**相關且已確認的補償控制 → **−1**；否則 → **0**

**控制-暴露映射邏輯**：

| 暴露 Affected Service | 可能相關的控制 |
|----------------------|---------------|
| HTTP/HTTPS 相關 | CC-02 (WAF), CC-04 (ACL), CC-06 (TLS) |
| SSH 相關 | CC-04 (ACL), CC-05 (MFA) |
| 其他 TCP/UDP 服務 | CC-01 (Segmentation), CC-03 (IDS/IPS), CC-04 (ACL) |
| 主機層級（任何） | CC-01 (Segmentation), CC-03 (IDS/IPS), CC-07 (EDR) |

> 若自動映射結果不確定（例如非典型服務），呈現映射結果請使用者確認。

### 4.4 Layer 3 — 最終調整規則

```
Net Adjustment = Exploitability Adj. + Controls Adj.
Net Adjustment = clamp(Net, -1, +1)        ← 淨調整上限 ±1 級
Final Adjusted Severity = Base + Net Adjustment
Final Adjusted Severity = clamp(Final, info, critical)  ← 不超出等級範圍
```

**特殊規則**：
- `info` 級暴露**不參與調整** — 維持 `info`，排序置底。理由：info 級為資訊性揭露，不構成可利用風險。
- 當淨調整為 0 時，Adjusted Severity = Base Severity。
- Rationale 欄位必須記錄調整推導過程（例如：`Base high → +1 confirmed-in-wild, -1 WAF → net 0 → Adjusted high`）

### 4.5 評估完整公式摘要

```
1. Base = RiskMatrix[Raw Severity][Business Criticality]
2. Exploitability Adj. = { confirmed-in-wild: +1, poc-available: 0, theoretical: -1 }
3. Controls Adj. = { has_relevant_control: -1, no_relevant_control: 0 }
4. Net = clamp(Exploitability Adj. + Controls Adj., -1, +1)
5. Adjusted Severity = clamp(Base + Net, info, critical)
   Exception: if Raw Severity == info → Adjusted Severity = info (skip adjustment)
```

---

## 五、執行步驟

### Step 0：上下文載入（Context Loading）

**目的**：讀取所有前序階段資料，彙整待評估暴露清單。

**動作**：
1. 讀取 `ctem-state.md` → 確認 Discovery 狀態 `completed`
2. 設定 Prioritization 狀態為 `in_progress`
3. 讀取 Scoping Summary → 擷取 Business Criticality、Business Function、Regulatory Context
4. 讀取 Discovery Summary → 擷取 Exposures Found 表格
5. 讀取 `reports/assets/<id>.md` → 擷取 Exposure Registry
6. 彙整待評估清單：所有 Current Status 為 `open`、`new`、`known`、`escalated`、`reopened` 的暴露
7. 向使用者呈現摘要：

> *「本輪 Prioritization 將評估以下暴露：」*
>
> | # | Exposure ID | Title | Raw Severity | Type | Affected Service |
> |---|-------------|-------|-------------|------|------------------|
> | 1 | EXP-001 | ... | high | vulnerability | HTTP/80 |
> | ... | | | | | |
>
> *「Business Criticality：<level>（來自 Scoping）。接下來將進行風險矩陣評估。」*

### Step 1：基線風險評估（Base Risk Assessment）

**目的**：用風險矩陣為每筆暴露計算 Base Adjusted Severity。

**動作**：
1. **讀取 [risk-matrix.md](./references/risk-matrix.md)**——此為必要步驟，包含完整矩陣定義
2. 對每筆暴露，以 Raw Severity × Business Criticality 查表 → Base Adjusted Severity
3. 呈現結果表格供使用者確認：

> *「風險矩陣評估結果（Business Criticality = <level>）：」*
>
> | # | Exposure ID | Title | Raw Severity | Base Adjusted Severity |
> |---|-------------|-------|-------------|----------------------|
> | 1 | EXP-001 | ... | high | <result> |
> | ... | | | | |

**注意**：此步驟為純機械式查表，不涉及主觀判斷。

### Step 2：情境調整（Contextual Adjustment）

**目的**：基於可利用性情境與補償控制，對 Base Severity 做 ±1 級調整。

#### Step 2a — 補償控制盤點（一次性）

呈現結構化控制清單，請使用者盤點主機層級的補償控制：

> *「請確認目標主機目前已部署哪些補償控制：」*
>
> | # | 控制類別 | 是否就位？ |
> |---|---------|-----------|
> | CC-01 | Network Segmentation（網路分段） | Yes / No |
> | CC-02 | WAF（Web 應用防火牆） | Yes / No |
> | CC-03 | IDS/IPS（入侵偵測/防禦） | Yes / No |
> | CC-04 | Firewall Rules / ACL（防火牆規則） | Yes / No |
> | CC-05 | MFA / Strong Auth（多因子驗證） | Yes / No |
> | CC-06 | TLS / Encryption（傳輸加密） | Yes / No |
> | CC-07 | EDR / Endpoint Protection（端點防護） | Yes / No |

記錄使用者回答，建立主機控制盤點結果。

#### Step 2b — 可利用性分類 + 控制映射（逐暴露）

對每筆非 `info` 級暴露：

1. **可利用性分類**：AI 根據 CVE 資訊或暴露特性建議分類（`confirmed-in-wild` / `poc-available` / `theoretical`），呈現建議理由，請使用者確認或覆寫。
2. **補償控制映射**：自動根據 Affected Service 和暴露 Type，從 Step 2a 的盤點中映射相關控制。若有相關且已確認的控制 → Controls Adj = −1。
3. **計算**：套用最終調整規則（見 4.4）→ 得出 Adjusted Severity。

**呈現方式**：可以批次處理多筆暴露，以表格呈現 AI 建議，使用者一次確認：

> *「以下為每筆暴露的情境評估建議，請確認或修改：」*
>
> | # | Exposure ID | Title | Base | Exploitability (建議) | Relevant Controls | Net Adj. | Adjusted |
> |---|-------------|-------|------|----------------------|-------------------|----------|----------|
> | 1 | EXP-001 | ... | critical | confirmed-in-wild (+1) | None | +1→capped | critical |
> | 2 | EXP-002 | ... | medium | poc-available (0) | WAF (-1) | -1 | low |

使用者可針對個別暴露修改 Exploitability 分類或補償控制映射。

#### Step 2c — 確認最終 Adjusted Severity

所有調整完成後，呈現最終結果供使用者確認：

> *「最終 Adjusted Severity 結果：」*
>
> | Priority | Exposure ID | Title | Raw Severity | Adjusted Severity | Exploitability | Controls | Rationale |
> |----------|-------------|-------|-------------|-------------------|----------------|----------|-----------|
> | 1 | EXP-001 | ... | high | critical | confirmed-in-wild | None | Base=critical, +1 exploit, net +1→capped critical |
> | 2 | EXP-002 | ... | medium | low | poc-available | WAF | Base=medium, 0 exploit, -1 WAF, net -1 |
>
> *（按 Adjusted Severity 由高到低排序，同級依 Raw Severity 排序）*

### Step 3：跨 Session 風險比對（Cross-Session Risk Comparison）

**目的**：與前輪 Adjusted Severity 比對，偵測風險升降。

**首輪模式**：跳過此步驟。

**返回輪動作**：
1. 從 Asset Profile 的 Exposure Registry 讀取每筆暴露的前輪 `Adjusted Severity`
2. 與本輪 Adjusted Severity 比較
3. 記錄變化：

| 變化類型 | 條件 | 說明 |
|---------|------|------|
| `escalated` | 本輪 Adjusted > 前輪 Adjusted | 風險升級 |
| `de-escalated` | 本輪 Adjusted < 前輪 Adjusted | 風險降級 |
| `unchanged` | 本輪 Adjusted = 前輪 Adjusted | 無變化 |

4. 呈現風險變化摘要：

> *「跨 Session 風險變化：」*
>
> | # | Exposure ID | Title | Previous Adjusted | Current Adjusted | Change |
> |---|-------------|-------|-------------------|------------------|--------|
> | 1 | EXP-001 | ... | medium (S-001) | critical (S-002) | ↑ escalated |

> **注意**：此處比對的是 **Adjusted Severity**（業務風險調整後的等級），與 Discovery 在 Step 3b 比對 **Raw Severity**（工具原始評級）不同。兩者為不同維度的比較。

### Step 4：寫入與產出（Write & Output）

**目的**：將評估結果寫入所有目標位置。

#### 4a — 寫入 Asset Profile (`reports/assets/<id>.md`)

**Exposure Registry 表**：
- 更新每筆暴露的 `Adjusted Severity` 欄位：格式為 `<level> (S-XXX)`，例如 `high (S-002)`
- 首輪：首次寫入 Adjusted Severity
- 返回輪：覆寫為本輪最新結果（歷史追蹤由 Severity History 欄位負責，該欄位由 Discovery 維護 Raw Severity 歷史；Adjusted Severity 只保留最新值）

**Current Risk Summary 表**：
- `Overall Risk Level`：所有 open 暴露中最高的 Adjusted Severity
- `Open Exposures`：status 為 open / new / known / escalated / reopened 的暴露數量
- `Highest Severity`：同 Overall Risk Level
- `Trend`：與前輪 Overall Risk Level 比較：
  - 本輪 > 前輪 → `↑ escalating`
  - 本輪 = 前輪 → `→ stable`
  - 本輪 < 前輪 → `↓ improving`
  - 首輪 → `→ first assessment`

#### 4b — 寫入 Prioritization Summary (`ctem-state.md`)

格式見「六、資訊傳遞機制」。

#### 4c — 更新 Phase Status (`ctem-state.md`)

- 設定 Prioritization 為 `completed`
- 填入 Key Findings Summary 和 Last Updated
- 新增 Transition Log 項目

---

## 六、資訊傳遞機制

### 6.1 持久化位置

| 位置 | 內容 | 生命週期 |
|------|------|---------|
| `ctem-state.md` → `### Prioritization Summary` | 本輪 Prioritization 結果摘要 | 單輪 session |
| `reports/assets/<id>.md` → Exposure Registry | Adjusted Severity | 跨輪次（每輪覆寫最新值） |
| `reports/assets/<id>.md` → Current Risk Summary | 整體風險狀態 | 跨輪次（每輪覆寫最新值） |

### 6.2 Prioritization Summary 格式（寫入 ctem-state.md）

```markdown
### Prioritization Summary

**Assessment Date**: <ISO 8601 日期>
**Business Criticality**: <from Scoping>
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

| # | Exposure ID | Previous Adjusted | Current Adjusted | Change |
|---|-------------|-------------------|------------------|--------|
| 1 | EXP-001 | medium (S-001) | critical (S-002) | ↑ escalated |

#### Risk Overview

| Metric | Value |
|--------|-------|
| Total Exposures Assessed | X |
| Adjusted Severity Distribution | critical: N, high: N, medium: N, low: N, info: N |
| Risk Changes | N escalated, N de-escalated, N unchanged |
| Overall Risk Level | <highest adjusted severity among open exposures> |
| Trend | ↑ escalating / → stable / ↓ improving / → first assessment |
| Top Priority Action | <brief recommendation for highest-priority exposure> |
```

### 6.3 下游階段如何取得資訊

- **Validation** 啟動時 → 讀取 `ctem-state.md` 的 `### Prioritization Summary` → 取得排序清單、Adjusted Severity、Exploitability 分類
- 需要更多細節 → 讀取 `reports/assets/<id>.md` 的 Exposure Registry（含 Adjusted Severity）和 Current Risk Summary

### 6.4 Session Report 資料

Prioritization 產出的以下資料由報告產生階段（ctem-report-guide）編入 Session Report：
- `Exposure Summary` → `CVSS` 欄位：從 Discovery 的 CVE 資訊帶入 CVSS base score（若可用）
- `Risk Changes Detected` 表：從 Prioritization Summary 的 Risk Changes 表編入

Prioritization **不直接寫入 Session Report 檔案** — Session Report 在五階段全部完成後由報告產生流程統一建立。

---

## 七、完成條件與 Checklist

Prioritization skill 在所有步驟做完後，產出以下 checklist：

```markdown
## Prioritization Completion Checklist

- [ ] Scoping Summary 與 Discovery Summary 已讀取
- [ ] 所有暴露已通過風險矩陣計算 Base Adjusted Severity
- [ ] 補償控制已盤點（主機層級控制清單）
- [ ] 每筆暴露的可利用性已分類（使用者確認）
- [ ] 情境調整已套用，最終 Adjusted Severity 已確定（使用者確認）
- [ ] 跨 Session 風險比對已完成（或確認為首輪）
- [ ] Asset Profile 的 Adjusted Severity 與 Current Risk Summary 已更新
- [ ] Prioritization Summary 已寫入 ctem-state.md
```

**職責劃分**：
- Prioritization skill：執行評估、產出 checklist、寫入 Adjusted Severity 與 Current Risk Summary
- ctem-flow：驗證 checklist、更新 Phase Status、執行階段轉換、Backtrack Check

---

## 八、互動風格

採**混合模式**（與 Scoping、Discovery 一致）：
1. 自動讀取前序階段資料，自動完成矩陣計算
2. 以結構化表格呈現 AI 建議，讓使用者批次確認
3. 僅在使用者需要覆寫時逐項互動
4. 不重複詢問前序階段已確認的資訊

額外互動原則：
- 每個評估步驟的結果都以表格呈現，讓使用者看得到推導過程
- 可利用性分類：AI 建議 + 簡短理由 → 使用者確認或覆寫
- 補償控制盤點：一次性清單勾選，非逐暴露重複詢問
- 最終結果呈現排序表格，讓使用者看到完整優先順序後再確認

---

## 九、參考檔案

獨立存放於 `references/` 目錄，SKILL.md 透過相對連結引用。

### 9.1 references/risk-matrix.md

風險矩陣核心參考，包含：
- **完整 5×4 風險矩陣表**：Raw Severity (critical/high/medium/low/info) × Business Criticality (critical/high/moderate/low) → Base Adjusted Severity
- **矩陣設計依據**：NIST SP 800-30 Rev. 1 Table H-3 + CVSS v3.1 severity scale
- **可利用性分類定義**：三級分類的判斷標準與範例
- **補償控制清單**：七項控制類別的完整定義、適用場景、與暴露的映射規則
- **最終調整公式**：完整計算規則與邊界條件
- **排序規則**：同級暴露的次要排序標準

> **SKILL.md 必須在 Step 1 開始前讀取此檔案。**

---

## 十、檔案影響範圍

建置 Prioritization skill 時需要建立/修改的檔案：

| 檔案 | 動作 | 說明 |
|------|------|------|
| `.github/skills/ctem-3-prioritization/SKILL.md` | 重寫 | 主要 prompt 邏輯 |
| `.github/skills/ctem-3-prioritization/SKILL.zh-TW.md` | 新增 | 繁體中文版（輔助閱讀） |
| `.github/skills/ctem-3-prioritization/references/risk-matrix.md` | 新增 | 風險矩陣 + 調整規則 |
| `.github/skills/ctem-3-prioritization/references/risk-matrix.zh-TW.md` | 新增 | 繁體中文版 |

> 注意：`ctem-state.md` 和 `reports/` 模板不需要修改 — 已有預留的 `### Prioritization Summary` 位置和對應欄位。

---

## 十一、與 ctem-flow 的互動點

Prioritization skill 完成後，ctem-flow 的 Backtrack Check 會檢查：

| 檢查項 | 來源 | 處理 |
|--------|------|------|
| 風險側寫顯著變化 | Prioritization Summary → Risk Changes 表 | 若風險升級數量異常多，ctem-flow 可能建議回溯至 Discovery 重新掃描 |
| Adjusted Severity 與 Raw Severity 差距過大 | Prioritization Summary → Prioritized Exposure List | 若多筆暴露的調整幅度都在同方向，可能暗示 Scoping 的 Business Criticality 需要重新評估 |
| 新暴露在後續階段出現 | Phase 4/5 若發現新暴露 | backtrack 回 Discovery，再重跑 Prioritization |

Prioritization 在 Summary 中提供完整的調整推導過程（Rationale 欄位），讓 ctem-flow 有足夠資訊判斷是否需要 backtrack。

---

## 十二、參考來源

| 來源 | 用途 |
|------|------|
| Gartner, 2022 — *Implement a CTEM Program* (G00766755) | CTEM 框架定義、Prioritization 階段職責（不只看 CVSS，要考慮業務影響） |
| NIST SP 800-30 Rev. 1 — *Guide for Conducting Risk Assessments* (2012) | 風險矩陣設計依據（Table H-3 — 5-level impact scale） |
| FIPS 199 — *Standards for Security Categorization* (2004) | Business Criticality 等級定義來源（Scoping 產出） |
| FIRST — *CVSS v3.1 Specification Document* (2019) | Raw Severity 分級依據、severity 等級命名慣例 |
| CISA — *Known Exploited Vulnerabilities Catalog* | 可利用性分類的權威參考（選用） |
| NIST — *National Vulnerability Database* | CVE 資訊、CVSS 分數查詢參考 |

---

## 十三、設計決策記錄

以下記錄本設計稿的所有關鍵決策及理由，供後續追溯：

| # | 決策 | 結果 | 理由 |
|---|------|------|------|
| 1 | 風險評分模型 | A — 簡單矩陣為基礎 | 一致性高、可重複、透明度高，避免黑箱公式 |
| 2 | 可利用性情境 | A — 三級定性分類 | 不依賴外部 API、AI 可從公開資訊推斷、使用者確認、無 CVE 的暴露也適用 |
| 3 | 補償控制 | C — 結構化控制清單 | 系統化、可重複、主機層級一次盤點避免重複詢問 |
| 4 | Workflow 步驟 | 5 步驟（Step 0–4） | 對齊 Discovery 的步驟結構，每步職責清晰 |
| 5 | Reference 檔案 | 1 份（risk-matrix.md） | 核心評分邏輯獨立維護，可利用性定義直接放 SKILL.md 不需獨立檔案 |
| 6 | Summary 格式 | Prioritized Exposure List + Risk Overview | 提供 Phase 4 所需的排序清單與完整評估上下文 |
| 7 | info 級暴露處理 | 不參與調整，維持 info | info 級為資訊揭露，不構成可利用風險，避免無意義的等級調整 |
| 8 | 淨調整上限 | ±1 級 | 避免極端調整、保持可預測性（confirmed-in-wild + 無控制 = +1，theoretical + 有控制 = -1） |
| 9 | 補償控制評估粒度 | 主機層級盤點 + 自動映射到暴露 | 減少使用者互動次數，同時保持暴露層級的精確度 |
| 10 | 跨 session 比對維度 | Adjusted Severity（非 Raw Severity） | 與 Discovery 的 Raw Severity 比對互補，Prioritization 關注業務風險維度的變化 |
| 11 | Current Risk Summary 寫入者 | Prioritization | 需要 Adjusted Severity 才能計算整體風險，Discovery 僅有 Raw Severity 無法計算 |
| 12 | Session Report 寫入時機 | 不直接寫入，由報告產生流程統一處理 | 對齊 Discovery 的模式，Session Report 在五階段完成後統一建立 |
| 13 | Severity vs Criticality 命名 | Severity 用 medium、Criticality 用 moderate | 對齊各自的框架慣例（CVSS 用 medium、FIPS 199 用 moderate），避免混淆 |
