# 業務關鍵性評估矩陣

本參考文件定義 CTEM Scoping（Step 3）中用於評估目標主機業務關鍵性的框架。

## 框架依據

本矩陣結合兩個 NIST 框架，以分層方式運作：

| 層級 | 框架 | 角色 |
|------|------|------|
| **輸入層** — 各維度影響評級 | **FIPS 199**（NIST, 2004）§3 | 定義 CIA 各維度的 Low / Moderate / High 影響標準 |
| **輸出層** — 整體關鍵性推導 | **NIST SP 800-30 Rev. 1**（NIST, 2012）Table H-3 | 提供五級定性影響量表，推導單一整體關鍵性等級 |

### 主要來源

1. **FIPS 199** — *Standards for Security Categorization of Federal Information and Information Systems*
   - 發布者：NIST, 2004 年 2 月
   - 連結：https://csrc.nist.gov/pubs/fips/199/final
   - 引用章節：**§3** — 機密性、完整性、可用性之潛在影響定義

2. **NIST SP 800-30 Rev. 1** — *Guide for Conducting Risk Assessments*
   - 發布者：NIST, 2012 年 9 月
   - 連結：https://csrc.nist.gov/pubs/sp/800-30/r1/final
   - 引用章節：**Appendix H, Table H-3** — 影響等級評估量表（五級定性量表）；**Appendix I, Table I-2** — 各等級不利影響範例

### 輔助來源

3. **NIST SP 800-60 Vol. 1 Rev. 1** — *Guide for Mapping Types of Information and Information Systems to Security Categories*
   - 發布者：NIST, 2008 年 8 月
   - 連結：https://csrc.nist.gov/pubs/sp/800-60/v1/r1/final
   - 角色：為 CIA 問答中的操作範例提供依據（資訊類型對應影響等級）

### CTEM 對齊

Gartner CTEM 框架要求範圍界定從**業務成果與風險**出發，而非從技術資產清冊開始（Gartner, 2022, G00766755）。本矩陣將此原則具體化：每項評級均以 NIST 正式定義的業務影響為基礎。

---

## Step 1 — CIA 影響評級（FIPS 199 §3）

依序向使用者提出以下三個問題。每個維度記錄答案為 **high**、**moderate** 或 **low**。

**FIPS 199 定義**欄位包含 FIPS 199 §3 的正式判定標準 — 這是每項評級的權威依據。**操作範例**欄位提供源自 NIST SP 800-60 Vol. 1 資訊類型對應的實務參考；範例用於輔助理解，不取代正式定義。

### 問題 1 — 機密性（Confidentiality）

> 若這台機器被入侵，未經授權揭露其資訊的影響程度？

| 評級 | FIPS 199 §3 定義 | 操作範例 |
|------|-----------------|---------|
| **高（High）** | 未經授權之揭露預期將對組織運作、組織資產或個人造成**嚴重或災難性的不利影響**（severe or catastrophic adverse effect） | 客戶個資、財務記錄、營業秘密、身分驗證憑證、加密金鑰 |
| **中（Moderate）** | 未經授權之揭露預期將對組織運作、組織資產或個人造成**重大的不利影響**（serious adverse effect） | 內部文件、營運資料 — 無受規範或高敏感資訊 |
| **低（Low）** | 未經授權之揭露預期將對組織運作、組織資產或個人造成**有限的不利影響**（limited adverse effect） | 僅有公開或無敏感性質的資訊 |

### 問題 2 — 完整性（Integrity）

> 若這台機器的資料被竄改，業務影響程度？

| 評級 | FIPS 199 §3 定義 | 操作範例 |
|------|-----------------|---------|
| **高（High）** | 未經授權之修改或破壞預期將對組織運作、組織資產或個人造成**嚴重或災難性的不利影響**（severe or catastrophic adverse effect） | 直接影響客戶服務、財務正確性或安全關鍵作業 |
| **中（Moderate）** | 未經授權之修改或破壞預期將對組織運作、組織資產或個人造成**重大的不利影響**（serious adverse effect） | 影響內部決策或報告；事後可偵測並修正 |
| **低（Low）** | 未經授權之修改或破壞預期將對組織運作、組織資產或個人造成**有限的不利影響**（limited adverse effect） | 影響極小；容易偵測與復原 |

### 問題 3 — 可用性（Availability）

> 若這台機器離線 24 小時，業務影響程度？

| 評級 | FIPS 199 §3 定義 | 操作範例 |
|------|-----------------|---------|
| **高（High）** | 對資訊或系統之存取或使用中斷，預期將對組織運作、組織資產或個人造成**嚴重或災難性的不利影響**（severe or catastrophic adverse effect） | 核心業務完全中斷；造成營收損失或 SLA 違約 |
| **中（Moderate）** | 對資訊或系統之存取或使用中斷，預期將對組織運作、組織資產或個人造成**重大的不利影響**（serious adverse effect） | 部分服務降級但有替代方案；無即時營收影響 |
| **低（Low）** | 對資訊或系統之存取或使用中斷，預期將對組織運作、組織資產或個人造成**有限的不利影響**（limited adverse effect） | 對日常運作幾乎無影響 |

---

## Step 2 — 推導整體關鍵性（SP 800-30 Rev. 1, Table H-3）

將 Step 1 的三個 CIA 評級，對應到 NIST SP 800-30 Rev. 1 Appendix H, Table H-3 的定性影響量表，推導單一**業務關鍵性**等級。

### 推導規則

| 規則 | 條件 | 關鍵性 | SP 800-30 Table H-3 依據 |
|------|------|--------|--------------------------|
| **R1** | ≥ 2 個 CIA 維度評為 **High** | **Critical** | 多重嚴重/災難性不利影響 → **Very High** 影響 |
| **R1-Q** | 恰好 1 個 CIA 維度評為 **High** 且符合任一 R1 限定條件（見下方） | **Critical** | 主要任務失效、重大財務損失或對個人之嚴重傷害 → **Very High** 影響 |
| **R2** | 恰好 1 個 CIA 維度評為 **High**，未符合 R1 限定條件 | **High** | 嚴重或災難性不利影響 → **High** 影響 |
| **R3** | 最高 CIA 維度 = **Moderate** | **Moderate** | 重大不利影響 → **Moderate** 影響 |
| **R4** | 最高 CIA 維度 = **Low** | **Low** | 有限不利影響 → **Low** 影響 |

### R1 限定條件 — 單一「High」何時升級為「Critical」

當恰好一個 CIA 維度評為 High 時，使用以下限定條件判斷影響是否達到 SP 800-30 的 **Very High** 門檻。若**任一**條件為真，套用 R1-Q（Critical）。

| # | 限定條件 | SP 800-30 Table I-2 依據 |
|---|---------|--------------------------|
| Q1 | 入侵將導致組織**無法執行一項或多項主要任務/業務功能** | "inability to perform current missions/business functions" |
| Q2 | 入侵將導致威脅組織存續的**重大財務損失** | "major financial loss" |
| Q3 | 入侵將對個人造成**嚴重或災難性傷害**（如生命損失、嚴重身體傷害） | "severe or catastrophic harm to individuals" |

> **備註**：若 Q1–Q3 皆不明確適用，預設為 R2（High）。有疑慮時不升級 — 下游 CTEM 階段（Prioritization、Validation）將提供進一步的風險校準。

### 關鍵性等級總覽

| 等級 | SP 800-30 影響等級 | 定義 | 典型範例 |
|------|-------------------|------|---------|
| **Critical** | Very High | 入侵產生多重嚴重/災難性不利影響；組織無法執行主要任務 | 面向客戶的生產伺服器、AD 網域控制站、核心資料庫、支付閘道 |
| **High** | High | 入侵產生嚴重或災難性不利影響；任務能力顯著降低 | 內部重要服務、備援基礎設施、CI/CD 管線 |
| **Moderate** | Moderate | 入侵產生重大不利影響；可察覺的降級但核心任務不受影響 | 開發環境、監控系統、內部工具 |
| **Low** | Low | 入侵產生有限不利影響；對營運影響極小 | 測試機、沙盒、已退役但仍在線的服務 |

## 記錄結果

將以下內容寫入資產檔案（`reports/assets/<id>.md`）：

- **Business Criticality** 欄位 → 最終等級（`critical` / `high` / `moderate` / `low`）
- **Owner / Team** 欄位 → 若對話中有提供

在資產檔案的 `## Notes` 區段以 HTML 註解記錄各維度評級與套用的推導規則，以利完整追溯：

```
<!-- CIA Assessment: C=<rating>, I=<rating>, A=<rating> | Rule: <R1|R1-Q|R2|R3|R4> | Criticality: <level> (<qualifier or justification>) -->
```

範例：
```
<!-- CIA Assessment: C=high, I=high, A=high | Rule: R1 (3 dims High) | Criticality: critical -->
<!-- CIA Assessment: C=high, I=moderate, A=low | Rule: R1-Q (Q1: primary mission failure) | Criticality: critical -->
<!-- CIA Assessment: C=high, I=moderate, A=moderate | Rule: R2 | Criticality: high -->
<!-- CIA Assessment: C=moderate, I=low, A=moderate | Rule: R3 | Criticality: moderate -->
<!-- CIA Assessment: C=low, I=low, A=low | Rule: R4 | Criticality: low -->
```

---

## 參考文獻

1. National Institute of Standards and Technology. *Standards for Security Categorization of Federal Information and Information Systems*. FIPS PUB 199. February 2004. Available: https://csrc.nist.gov/pubs/fips/199/final

2. National Institute of Standards and Technology. *Guide for Conducting Risk Assessments*. NIST Special Publication 800-30 Revision 1. September 2012. Available: https://csrc.nist.gov/pubs/sp/800-30/r1/final

3. National Institute of Standards and Technology. *Guide for Mapping Types of Information and Information Systems to Security Categories*. NIST Special Publication 800-60 Volume 1 Revision 1. August 2008. Available: https://csrc.nist.gov/pubs/sp/800-60/v1/r1/final

4. Gartner, Inc. *Implement a Continuous Threat Exposure Management (CTEM) Program*. Research Note G00766755. July 2022.
