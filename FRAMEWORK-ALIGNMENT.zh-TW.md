# CTEM 框架對齊說明

本文件將 Gartner 的 CTEM（Continuous Threat Exposure Management，持續性威脅曝險管理）框架定義對映到本工具包的實作方式，提供各階段可追溯性。

## 來源

Gartner, Inc. *Implement a Continuous Threat Exposure Management (CTEM) Program*. Research Note G00766755. July 2022.

---

## 階段對齊

### 第 1 階段 — Scoping（範疇界定）

| 面向 | Gartner 定義 | 本實作 |
|------|--------------|--------|
| **目標** | 依據業務風險與策略優先順序定義攻擊面範圍，而非僅以 IT 資產清單為主 | 以業務成果為核心：透過 Business Function 與 CIA 三元組 Business Criticality 評估（FIPS 199 / NIST SP 800-30）來界定範圍 |
| **範圍類別** | 外部攻擊面、SaaS 姿態、數位身分、數位供應鏈、雲端組態 | **僅限單一主機之基礎設施層**；SaaS 姿態、數位身分、數位供應鏈與雲原生組態皆明確排除（見 SKILL.md 的 Scope Limitation） |
| **業務脈絡** | Scoping 應從業務成果出發，而非技術盤點 | Step 0（Business Context）在任何技術資料前先收集 Business Function、Regulatory Context、Threat Profile |
| **輸出** | 帶有業務標註的已定義攻擊面 | `ctem-state.md` 的 Scoping Summary + `reports/assets/` 的 Asset Profile |

### 第 2 階段 — Discovery（探索）

| 面向 | Gartner 定義 | 本實作 |
|------|--------------|--------|
| **目標** | 在已定義範圍內主動發掘資產、漏洞、錯誤設定與其他暴露 | 於 Scoping 邊界內，使用 nmap、Nuclei 與手動輸入進行系統化暴露識別 |
| **探索對象** | 漏洞、錯誤設定、資產盤點落差、組態漂移 | 四種暴露類型：`vulnerability`、`misconfiguration`、`information-disclosure`、`outdated-software` |
| **持續性** | Discovery 應是持續進行，而非一次性活動 | 透過 Exposure Registry 的跨 session 比對（new / known / escalated / reopened / mitigated）實現縱向追蹤 |
| **工具中立** | 框架本身不綁定工具 | 原生支援 nmap + Nuclei；手動輸入路徑可接收任意工具發現 |
| **輸出** | 完整的暴露清單 | `ctem-state.md` 的 Discovery Summary + `reports/assets/` 的 Exposure Registry |

### 第 3 階段 — Prioritization（優先排序）

| 面向 | Gartner 定義 | 本實作 |
|------|--------------|--------|
| **目標** | 不只依 CVSS 嚴重性排序，還要納入業務影響、可利用性脈絡與補償控制 | 三層評估：Risk Matrix（Raw Severity × Business Criticality）→ Contextual Adjustment（Exploitability ±1、Compensating Controls ±1，淨值上限 ±1）→ 每筆暴露的 Adjusted Severity |
| **關鍵差異** | 超越傳統漏洞嚴重性，納入業務脈絡、威脅情報與資產關鍵性 | 區分 Raw Severity（Discovery）與 Adjusted Severity（Prioritization）；可利用性分為 `confirmed-in-wild` / `poc-available` / `theoretical`；7 項結構化補償控制清單搭配自動控制-暴露映射 |
| **輸出** | 含風險分數的優先暴露清單 | Prioritized Exposure List（含 Adjusted Severity、Exploitability、Controls Applied、Rationale）寫入 Asset Profile（`Adjusted Severity` + `Current Risk Summary`）與 `ctem-state.md` 的 Prioritization Summary |

### 第 4 階段 — Validation（驗證）

| 面向 | Gartner 定義 | 本實作 |
|------|--------------|--------|
| **目標** | 確認暴露是否可被利用、評估攻擊路徑、過濾誤報 | 規劃中：三模組設計（Attack Path Reasoning、Exploit Validation、Result Analysis） |
| **方法** | 入侵與攻擊模擬、滲透測試、紅隊演練 | 設計上支援 AI 引導的驗證程序產生與結果分析 |
| **輸出** | 已驗證的暴露清單（confirmed/dismissed） | 規劃中：將更新 Exposure Registry 的 Current Status |

### 第 5 階段 — Mobilization（推動落地）

| 面向 | Gartner 定義 | 本實作 |
|------|--------------|--------|
| **目標** | 透過跨團隊修復流程，確保發現事項被實際處置 | 規劃中：修復計畫生成、行動指派、時程追蹤 |
| **關鍵差異** | 不只是開票，而是確保組織對齊與修復落地 | 設計包含 Asset Profile 的 Remediation History 追蹤 |
| **輸出** | 可執行且有負責人與期限的修復計畫 | 規劃中：將寫入 Session Report 與 Asset Profile |

---

## 範圍決策與理由

| 決策 | 理由 |
|------|------|
| 單一主機焦點（非整網段/子網） | 以深度優先於廣度，便於細緻的逐資產縱向追蹤。多主機支援為後續擴充。 |
| 僅基礎設施層（不含 SaaS、身分、供應鏈） | 對剛啟動曝險管理計畫的組織而言，最常見且務實的 CTEM 起點。 |
| 以 CVSS v3.1 為主要嚴重性基礎 | 截至 2026 年具最廣工具支援（nmap、Nuclei）與 NVD 覆蓋。可記錄 CVSS v4.0，但映射仍以 v3.1 為主。詳見 Discovery SKILL.md 的 CVSS 版本說明。 |
| 4 級 Business Criticality（不含 Very Low） | 需要納入 CTEM 評估的主機至少具 Low 影響。見 `business-criticality-matrix.md` 的說明。 |
| 分離 Raw 與 Adjusted Severity | 維持階段邊界清晰：Discovery 客觀呈現工具結果；Prioritization 加入業務脈絡。見 `reports/README.md` 的 Field Ownership Table。 |
| 每資產獨立 Exposure ID | 單主機 session 下最簡方案。多主機擴充建議採複合鍵（`ASSET-ID/EXP-ID`）。 |
| `outdated-software` 需使用者確認 | AI 訓練截止使版本新舊判斷有限；人機協作可提升準確性。 |

---

## 參考資料

1. Gartner, Inc. *Implement a Continuous Threat Exposure Management (CTEM) Program*. Research Note G00766755. July 2022.
2. NIST. *Standards for Security Categorization of Federal Information and Information Systems*. FIPS PUB 199. February 2004. https://csrc.nist.gov/pubs/fips/199/final
3. NIST. *Guide for Conducting Risk Assessments*. SP 800-30 Rev. 1. September 2012. https://csrc.nist.gov/pubs/sp/800-30/r1/final
4. NIST. *Guide for Mapping Types of Information and Information Systems to Security Categories*. SP 800-60 Vol. 1 Rev. 1. August 2008. https://csrc.nist.gov/pubs/sp/800-60/v1/r1/final
5. FIRST. *Common Vulnerability Scoring System v3.1: Specification Document*. June 2019. https://www.first.org/cvss/v3.1/specification-document
