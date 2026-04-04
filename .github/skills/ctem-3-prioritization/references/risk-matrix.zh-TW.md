# 風險評估矩陣 — 中文閱讀版

> 此檔案為 `risk-matrix.md` 的繁體中文翻譯，僅供閱讀參考。AI 實際執行時讀取的是英文版 `risk-matrix.md`。

---

此參考檔定義 CTEM Prioritization（Phase 3）的完整風險評估框架，包含風險矩陣、情境調整規則、補償控制定義與排序邏輯。

## 框架基礎

本矩陣結合暴露層級的技術嚴重性與資產層級的業務關鍵性，產出具風險意識的優先順序：

| 層級 | 框架 | 角色 |
|------|------|------|
| **X 軸** — Raw Severity | **CVSS v3.1**（FIRST, 2019） | 來自 Discovery 掃描工具的技術嚴重性 |
| **Y 軸** — Business Criticality | **FIPS 199** / **NIST SP 800-30 Rev. 1** | 來自 Scoping CIA 評估的業務影響 |
| **輸出** — Base Adjusted Severity | 綜合結果 | 風險導向的優先排序嚴重性 |

### 主要來源

1. **FIRST — CVSS v3.1 Specification Document**（2019）
   - URL：https://www.first.org/cvss/v3.1/specification-document
   - 角色：定義 Raw Severity 使用的嚴重性等級（critical/high/medium/low）。

2. **NIST SP 800-30 Rev. 1 — Guide for Conducting Risk Assessments**（2012）
   - URL：https://csrc.nist.gov/pubs/sp/800-30/r1/final
   - 引用章節：**Appendix H, Table H-3** — 影響等級評估量表；**Table G-3** — 威脅事件發生可能性評估量表
   - 角色：提供矩陣設計依據 — 結合可能性（以可利用性近似）與影響（業務關鍵性）來判定風險等級。

3. **FIPS 199 — Standards for Security Categorization**（2004）
   - URL：https://csrc.nist.gov/pubs/fips/199/final
   - 角色：定義 Scoping 中使用的 Business Criticality 等級（critical/high/moderate/low）。

### CTEM 對齊

Gartner 的 CTEM 框架要求 Prioritization **超越傳統的漏洞嚴重性**，納入業務影響、可利用性上下文和補償控制（Gartner, 2022, G00766755）。本矩陣將此原則操作化：Raw Severity 單獨不決定優先順序 — 它與業務上下文結合並針對現實因素進行調整。

---

## 第一部分 — 風險矩陣（Base Adjusted Severity）

### 矩陣定義

查找 **Raw Severity**（X 軸，來自 Discovery）與 **Business Criticality**（Y 軸，來自 Scoping）的交叉點，以決定 **Base Adjusted Severity**。

|  | **Raw: critical** | **Raw: high** | **Raw: medium** | **Raw: low** | **Raw: info** |
|---|---|---|---|---|---|
| **Criticality: critical** | critical | critical | high | medium | info |
| **Criticality: high** | critical | high | high | medium | info |
| **Criticality: moderate** | high | high | medium | low | info |
| **Criticality: low** | high | medium | low | low | info |

### 矩陣設計原則

1. **info 保持 info**：資訊性發現不構成可利用風險，無論業務關鍵性如何。始終排除調整，排序至最底部。

2. **關鍵業務資產提升風險**：`medium` 的 Raw Severity 在 `critical` 資產上變為 `high` Base — 反映即使技術難度中等，利用後的業務影響仍然嚴重。

3. **低關鍵性資產降低風險**：`high` 的 Raw Severity 在 `low` 關鍵性資產上變為 `medium` Base — 技術嚴重性是真實的，但利用後的業務影響有限。

4. **最高為 critical**：矩陣永不超過 `critical`。多個 `critical` 輸入仍然產出 `critical`。

5. **對角線對齊**：沿對角線（嚴重性與關鍵性等級匹配時），輸出通常等於 Raw Severity — 當技術評估與業務評估一致時，無需調整。

### 嚴重性等級順序

由低到高：`info` < `low` < `medium` < `high` < `critical`

此排序用於 Prioritization 階段的所有比較、調整和排序。

### 術語澄清

- **Severity** 等級：`info`、`low`、`medium`、`high`、`critical` — 對齊 CVSS v3.1。
- **Business Criticality** 等級：`low`、`moderate`、`high`、`critical` — 對齊 FIPS 199 / Scoping。
- 注意：Severity 使用 `medium`，而 Criticality 使用 `moderate`。這些是不同框架的慣例，不互通。

---

## 第二部分 — 情境調整規則

從矩陣計算出 Base Adjusted Severity 後，根據兩個因子進行情境調整：可利用性情境與補償控制。

### 可利用性情境

三級定性分類，逐暴露評估：

| 等級 | 定義 | 調整 | 判斷依據 |
|------|------|------|---------|
| `confirmed-in-wild` | 已在真實攻擊中被積極利用 | **+1** | 收錄於 CISA KEV 目錄；CVE 描述或廠商公告確認積極利用；威脅情報報告有活躍攻擊活動 |
| `poc-available` | 有公開的概念驗證程式碼，但未觀察到真實攻擊 | **0** | Exploit-DB 項目存在；GitHub PoC 儲存庫；Nuclei 模板標記為 verified；Metasploit 模組可用 |
| `theoretical` | 僅有理論可利用性；無公開 PoC 或已知攻擊 | **−1** | CVE 描述僅述可能性；無公開利用程式碼；需特定或不太可能的前提條件；複雜度高且無已知繞過方式 |

**評估指引**：
- **有 CVE** 的暴露：根據公開可用的 CVE 資訊（NVD、廠商公告、CISA KEV）分類。
- **無 CVE** 的暴露（misconfiguration、information-disclosure、outdated-software）：根據暴露性質分類：
  - 預設憑證或已知弱設定 → 通常為 `poc-available`（工具可直接利用）
  - 資訊洩露（版本標語、目錄列表） → 通常為 `theoretical`（資訊本身不可直接利用）
  - 無特定 CVE 的過時軟體 → 通常為 `theoretical`，除非已知攻擊活動針對該版本範圍

**重要**：此分類僅基於公開情報。不執行技術驗證（PoC 執行、滲透測試）— 那是 Phase 4 的職責。

### 補償控制調整

根據該暴露是否已有降低可利用性的相關控制。

**規則**：若暴露有**至少一項相關且已確認**的補償控制 → **−1**；否則 → **0**。

控制相關性由控制-暴露映射決定（見第三部分）。

### 淨調整計算

```
淨調整 = 可利用性調整 + 控制調整
淨調整 = clamp(Net, -1, +1)          ← 上限 ±1 級
Adjusted Severity = clamp(Base + Net, info, critical)   ← 不超出等級範圍
```

**例外**：若 Raw Severity = `info` → Adjusted Severity = `info`（跳過所有調整）。

**計算範例**：

| Base | 可利用性 | 控制 | 原始 Net | 限制後 Net | Adjusted |
|------|---------|------|---------|-----------|----------|
| high | confirmed-in-wild (+1) | 無 (0) | +1 | +1 | critical |
| high | confirmed-in-wild (+1) | WAF (-1) | 0 | 0 | high |
| medium | poc-available (0) | ACL (-1) | -1 | -1 | low |
| medium | theoretical (-1) | Segmentation (-1) | -2 | **-1**（限制） | low |
| critical | confirmed-in-wild (+1) | 無 (0) | +1 | +1 | **critical**（上限） |
| low | theoretical (-1) | EDR (-1) | -2 | **-1**（限制） | **low**（底線規則） |

**`low` → `info` 降級的底線規則**：當調整將 `low` Base Severity 向下推至 `info` 時，適用以下規則：
- 若暴露 Type 為 `vulnerability` 或 `misconfiguration` → **底線為 `low`**（這些類型具有固有的可利用風險，`info` 會誤判其風險）。在 Rationale 中記錄 "floor applied — vulnerability/misconfiguration cannot be demoted to info"。
- 若暴露 Type 為 `information-disclosure` 或 `outdated-software` → 允許降級至 `info`（這些類型在考慮情境後可能確實為資訊性）。在 Rationale 中記錄降級原因。

---

## 第三部分 — 補償控制參考

### 主機層級控制清單

| # | 控制 ID | 控制類別 | 說明 | 適用服務類型 |
|---|--------|---------|------|-------------|
| 1 | CC-01 | 網路分段 | 主機位於分段網路區域，存取受限（VLAN、微分段、DMZ） | 所有 |
| 2 | CC-02 | WAF（Web 應用防火牆） | 在請求到達應用程式前過濾惡意 HTTP/HTTPS 請求 | Web 服務（HTTP/HTTPS） |
| 3 | CC-03 | IDS/IPS | 入侵偵測/防禦系統，監控及/或阻擋主機惡意流量 | 所有 |
| 4 | CC-04 | 防火牆規則 / ACL | 特定服務的存取限制 — 來源 IP 白名單、端口限制、預設拒絕策略 | 特定端口/服務 |
| 5 | CC-05 | MFA / 強認證 | 存取服務需多因子驗證或強認證機制 | 認證類服務（SSH、管理面板、RDP） |
| 6 | CC-06 | TLS / 傳輸加密 | 傳輸層加密保護傳輸中的資料 | 傳輸敏感資料的服務 |
| 7 | CC-07 | EDR / 端點防護 | 端點偵測與回應代理程式在主機上運行中 | 主機層級（任何） |

### 控制-暴露映射

使用此映射來判定哪些已確認控制與各暴露相關：

| 暴露 Affected Service | 可能相關的控制 |
|----------------------|---------------|
| HTTP / HTTPS 相關 | CC-02 (WAF)、CC-04 (ACL) |
| HTTP / HTTPS — 資訊洩露或明文傳輸 | CC-06 (TLS) — 僅在暴露涉及資料攔截或明文傳輸時相關，不適用於應用層漏洞（如 path traversal 或 XSS） |
| SSH 相關 | CC-04 (ACL)、CC-05 (MFA) |
| 資料庫服務（MySQL、PostgreSQL 等） | CC-01 (網路分段)、CC-04 (ACL) |
| 其他 TCP/UDP 服務 | CC-01 (網路分段)、CC-03 (IDS/IPS)、CC-04 (ACL) |
| 主機層級（任何暴露） | CC-01 (網路分段)、CC-03 (IDS/IPS)、CC-07 (EDR) |

**映射規則**：
1. 每筆暴露根據其 Discovery 的 `Affected Service` 欄位映射到控制。
2. 一筆暴露可能匹配多行 — 取所有適用控制的聯集。
3. 控制視為「相關」的條件為：出現在映射中且在主機層級盤點中確認為「Yes」。
4. 若自動映射不確定（例如非標準服務名稱），呈現映射結果請使用者確認。

---

## 第四部分 — 優先排序規則

計算所有暴露的最終 Adjusted Severity 後，排列成優先順序清單。

### 主要排序：Adjusted Severity（降序）

`critical` → `high` → `medium` → `low` → `info`

### 次要排序（同級打破）：Raw Severity（降序）

當兩筆暴露的 Adjusted Severity 相同時，Raw Severity 較高者排前。確保技術上更嚴重的暴露在同一風險等級內優先獲得關注。

### 第三排序（仍同級）：可利用性（降序）

若 Adjusted Severity 和 Raw Severity 都相同：
- `confirmed-in-wild` → `poc-available` → `theoretical`

### 最終同級打破：Exposure ID（升序）

所有三個標準都相同的暴露，以 Exposure ID 升序排列，確保排序確定性。

---

## 參考來源

1. FIRST。*Common Vulnerability Scoring System v3.1: Specification Document*。2019 年 6 月。https://www.first.org/cvss/v3.1/specification-document
2. NIST。*Guide for Conducting Risk Assessments*。SP 800-30 Rev. 1。2012 年 9 月。https://csrc.nist.gov/pubs/sp/800-30/r1/final
3. NIST。*Standards for Security Categorization of Federal Information and Information Systems*。FIPS PUB 199。2004 年 2 月。https://csrc.nist.gov/pubs/fips/199/final
4. Gartner, Inc。*Implement a Continuous Threat Exposure Management (CTEM) Program*。Research Note G00766755。2022 年 7 月。
5. CISA。*Known Exploited Vulnerabilities Catalog*。https://www.cisa.gov/known-exploited-vulnerabilities-catalog
