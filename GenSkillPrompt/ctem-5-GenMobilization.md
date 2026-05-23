# CTEM Phase 5 — Mobilization Skill 完整設計稿

> 本文件統整所有討論決策，作為 `ctem-5-mobilization/SKILL.md` 的建置藍圖。

---

## 一、設計前提

- 流程無限貼近 Gartner CTEM 框架（Gartner, 2022, ID: G00766755）
- 本版為**簡單版**：單一機器 per session（多機器/多線程列入未來 TODO）
- Skill 保持獨立，流程控制回歸 `ctem-flow`
- 所有評估框架必須有依據並附來源（論文需求）
- Mobilization 的職責：**將已確認的威脅暴露轉化為可執行的修復行動計畫，分派責任、訂定時程、追蹤解決**
- Mobilization 不掃描、不驗證、不評分——它規劃、分派、追蹤
- 核心原則（Gartner）：Mobilization 不只是開修復工單——要確保組織層級的對齊與修復的落實
- Mobilization 是 CTEM 五階段中唯一會產出 `mitigated` 和 `accepted` 狀態轉換的修復階段

---

## 二、與上游的銜接

### 2.1 讀取上游資訊

Mobilization 啟動時：
1. 讀取 `ctem-state.md` → 確認 Validation 狀態為 `completed`（依 ctem-state-protocol）
2. 讀取 `ctem-state.md` → `### Scoping Summary` 取得：
   - Target Host / Hostname（修復目標）
   - Business Function（修復優先判斷的業務上下文）
   - Regulatory Context（合規性修復時程考量）
   - Owner / Team（修復行動的預設指派對象）
3. 讀取 `ctem-state.md` → `### Validation Summary` 取得：
   - Validation Results 表格（每筆暴露的 Validation Status：confirmed / false-positive / inconclusive）
   - Newly Discovered During Validation 表格（驗證期間新發現的暴露）
   - Attack Paths Identified 表格（攻擊路徑鏈結，用於 chain-breaker 分析）
   - Validation Statistics（Overall Risk Level、Risk Level Change）
   - Recommendation 欄位（Phase 4 的建議）
4. 讀取 `ctem-state.md` → `### Prioritization Summary` 取得：
   - Prioritized Exposure List（含 Adjusted Severity、Exploitability、Controls Applied、Rationale）
   - Host-Level Compensating Controls（現有控制清單，影響修復建議）
5. 讀取 `ctem-state.md` → `### Discovery Summary` 取得：
   - Open Services Detected（服務清單作為修復指令的上下文）
6. 讀取 `reports/assets/<id>.md` 取得：
   - Exposure Registry（完整暴露記錄含歷史，包含 Current Status）
   - Remediation History（檢查是否有前輪未完成的修復行動）
   - Current Risk Summary（修復後需更新）
7. 讀取 Session ID from `ctem-state.md` → `Session Info`

### 2.2 先決條件

- Validation 必須為 `completed` 才能啟動 Mobilization
- 若先決條件不滿足 → STOP，回報缺少的項目

### 2.3 首輪偵測

首輪/返回輪的區分由 Exposure Registry 的 Remediation History 是否有記錄決定。
- 首輪：Remediation History 為空 → 所有修復行動皆為新規劃
- 返回輪：檢查前輪 Remediation History 中 `pending-verification` 的項目 → 在 Step 0 提醒使用者

---

## 三、修復涵蓋範圍（Q1 決策：B — confirmed + inconclusive）

### 3.1 佇列建立規則

| Validation Status | 行動類型 | 說明 |
|-------------------|---------|------|
| `confirmed` | **Remediation Action** | 完整修復計畫：Quick Fix + 分類 + 工程量估算 + 時程 |
| `inconclusive` | **Investigation Action** | 調查行動：建議重新驗證的方法或額外掃描指令，不做完整修復規劃 |
| `false-positive` | 排除 | 已排除於風險計算外，不需行動 |

### 3.2 設計理由

- `confirmed` 暴露有明確的利用證據，可以制定精確的修復方案
- `inconclusive` 暴露不應被忽略——Gartner CTEM 強調不放過潛在風險——但因無法確認可利用性，僅提出調查建議（重新掃描、從其他網路位置驗證等），資源集中在已確認的暴露上
- `false-positive` 已在 Validation 中排除，不進入修復佇列

---

## 四、修復建議深度（Q2 決策：C — 分層式）

### 4.1 分層模式

| 層級 | 內容 | 呈現時機 |
|------|------|---------|
| **Quick Fix（主表格）** | 一句話修復建議 + Action Type + Effort 估算 + Priority | Step 1 自動產出 |
| **Detailed Steps（展開式）** | Step-by-step 修復指令與指引 | 使用者對個別暴露要求展開時，AI 查詢 `remediation-playbooks.md` 生成 |

### 4.2 Action Type 分類

| Action Type | 說明 | 典型範例 |
|-------------|------|---------|
| `patch` | 軟體升級或套用安全修補 | 升級 Apache 到最新版本 |
| `configure` | 修改配置以消除暴露 | 停用 SSH 弱密碼套件、關閉匿名 FTP |
| `harden` | 強化防護措施 | 新增 WAF 規則、啟用 MFA |
| `replace` | 替換元件 | 更換已 EOL 的軟體版本 |
| `investigate` | 進一步調查（僅 inconclusive） | 從其他網路位置重新驗證、執行更深度掃描 |

### 4.3 Effort 估算

| Effort Level | 說明 | 典型時間 |
|-------------|------|---------|
| `trivial` | 單一指令或設定變更 | < 30 分鐘 |
| `low` | 幾個步驟，需基本測試 | 30 分鐘 – 2 小時 |
| `medium` | 多步驟，可能需要服務重啟或短暫停機 | 2 – 8 小時 |
| `high` | 重大變更，需規劃維護窗口、備份、回滾計畫 | 1 – 3 天 |
| `critical` | 架構級變更或元件替換 | > 3 天 |

---

## 五、攻擊路徑修復策略（Q3 決策：C — Chain-Breaker + 全部暴露）

### 5.1 策略概述

1. **每個 confirmed 暴露**都有各自獨立的修復行動（Action）
2. 對每條攻擊路徑（Attack Path），分析並識別 **Chain-Breaker**——修復它可中斷整條攻擊鏈的最低成本節點
3. Chain-Breaker 身分成為 **Priority 加分因素**：若修復一個暴露就能中斷攻擊鏈，其修復優先度上調

### 5.2 Chain-Breaker 識別邏輯

對每條攻擊路徑（AP-NNN）：

1. 列出鏈上每個暴露的 Effort Level
2. 選出 Effort 最低的暴露作為 candidate chain-breaker
3. 若 Effort 相同，選擇位於鏈中最早位置的暴露（切斷越早，影響越大）
4. 呈現 chain-breaker 建議給使用者確認

### 5.3 Priority 排序規則

修復行動的最終優先排序依據以下因子（依序）：

1. **Adjusted Severity**（降序）：critical > high > medium > low
2. **Chain-Breaker 加分**：屬於 chain-breaker 的暴露同級時優先
3. **Effort Level**（升序）：同級時 Effort 越低越優先（quick-win）
4. **Exposure ID**（升序）：以上皆相同時按 ID 排序

---

## 六、風險接受工作流（Q4 決策：B — 結構化審批）

### 6.1 觸發時機

當使用者明確表示不修復某個暴露而選擇接受風險時觸發。可發生在 Step 1（修復計畫呈現後）或 Step 3（專門處理）。

### 6.2 必要記錄欄位

| 欄位 | 必要？ | 說明 |
|------|--------|------|
| Justification | 是 | 為何接受此風險（業務理由、成本考量等） |
| Approver | 是 | 風險接受的決策者（人名或角色） |
| Review Date | 是 | 預定重新評估日期（下一輪 CTEM 應重新檢視此接受決策） |

### 6.3 狀態轉換

接受風險時：
- Exposure Registry 中 `Current Status` 從 `confirmed` 變為 `accepted`
- 此為合法的狀態轉換（見 `reports/README.md` 狀態轉換矩陣：`confirmed → accepted` 由 Phase 5 觸發）
- `accepted` 暴露**不從風險計算中移除**——它仍然是一個已知風險，只是組織決定不修復
- 但在 Overall Risk Level 的呈現中應標註哪些風險是 accepted

### 6.4 不設嚴重性門檻

不限制哪些等級的暴露才能被接受——這是組織的治理決策。但 AI 應在高嚴重性暴露（critical / high）被接受時**主動提醒**風險：

> *「注意：EXP-001（Adjusted Severity: critical）被標記為 Accepted。此暴露的攻擊路徑已確認可達成完整系統存取。請確認此決策經過適當授權。」*

### 6.5 Review Date 與下一輪 CTEM

`Review Date` 記錄在 Remediation History 和 Mobilization Summary 中。未來的 CTEM Session（Discovery 或 Scoping）可以比對 Review Date 是否已到期，作為觸發重新評估的參考。此跨輪次邏輯目前以 metadata 形式存在——自動比對留為未來增強功能。

---

## 七、修復時間線 / SLA 模型（Q5 決策：B — 預設 SLA + 使用者覆寫）

### 7.1 預設 SLA 對照表

| Adjusted Severity | 預設 SLA | 說明 |
|-------------------|---------|------|
| `critical` | 24 小時 | 需立即處理；若無法在 24h 內修復，需有臨時緩解措施 |
| `high` | 7 天 | 一週內完成修復 |
| `medium` | 30 天 | 一個月內完成修復 |
| `low` | 90 天 | 一季內完成修復 |
| `info` | N/A | 資訊性發現，無 SLA |

> 此 SLA 預設值參考業界常見實務。完整定義與計算邏輯存放於 `references/remediation-sla.md`。

### 7.2 覆寫機制

使用者可在兩個層級覆寫 SLA：

| 層級 | 說明 | 範例 |
|------|------|------|
| **全域覆寫** | 修改整個 SLA 對照表 | 「我們的 critical SLA 是 48 小時」 |
| **逐項覆寫** | 針對個別暴露設定不同的 deadline | 「EXP-003 延長到 14 天，因為需要維護窗口」 |

### 7.3 Deadline 計算

- `Deadline = Plan Date + SLA Duration`
- 使用 ISO 8601 日期格式
- AI 自動套用預設 SLA 計算 deadline，呈現給使用者確認/覆寫

### 7.4 Regulatory Context 影響

若 Scoping Summary 包含非空的 `Regulatory Context`（如 PCI-DSS、GDPR、ISO 27001），在呈現 SLA 時加註合規性影響：

> *「注意：Regulatory Context 為 PCI-DSS。PCI-DSS 要求高風險漏洞在 30 天內修復。EXP-001 的預設 SLA（24h）已符合此要求。」*

Regulatory Context **不改變** SLA 預設值——它提供額外的合規性上下文供使用者參考。

---

## 八、Session 內快速修復驗證（Q6 決策：B 修改版 — 可選、使用者觸發制）

### 8.1 設計原則

- Mobilization 的核心職責是**規劃與動員**（plan & mobilize），不是**立即修復所有暴露**
- In-session verification 為**可選功能**，由使用者主動觸發
- **預設建議為 `planned`**——AI 在呈現修復計畫後，預設所有 action 的 Status 為 `planned`
- 使用者主動說「我修好了 EXP-XXX」或「我要現在修 EXP-XXX」時，才觸發驗證流程

### 8.2 論文實驗策略考量

為什麼 `planned` 是更好的預設：
- 同一輪 session 中全部修完 → 下一輪 Discovery 掃描結果全為 `mitigated`，失去縱向對比數據
- 適度保留 `planned` 項目 → 跨 session 可展示 risk trend 下降、exposure status 遷移矩陣等實驗資料
- 建議的論文實驗策略：Session 1 修 1–2 個 quick-win（展示 in-session verification），其餘標 `planned`；Session 2 驗收 `planned` 項目的修復成效（展示持續性追蹤能力）

### 8.3 驗證觸發流程

```
Use 者：「我已經修好 EXP-003」
  │
  ├─ AI 讀取 EXP-003 的修復建議（Quick Fix + Detailed Steps）
  │
  ├─ AI 建議驗證指令：
  │   「請執行以下指令驗證修復是否生效：」
  │   > curl -s --path-as-is http://10.10.10.5/icons/.%2e/%2e%2e/etc/passwd
  │   「若回傳 403 Forbidden 或找不到頁面 → 修復成功」
  │   「若仍回傳 /etc/passwd 內容 → 修復未生效」
  │
  ├─ 使用者執行並回報結果
  │
  ├─ AI 判定：
  │   ├─ 修復成功 → Status: `verified`
  │   │   → Current Status: confirmed → mitigated
  │   │   → Remediation History: Result = resolved
  │   │
  │   └─ 修復未生效 → Status 維持 `planned`
  │       → 建議替代修復方案或排除原因
  │
  └─ 更新 Asset Profile 的 Current Risk Summary（若有狀態變更）
```

### 8.4 Action Status 定義

| Status | 說明 | 觸發方式 |
|--------|------|---------|
| `planned` | 已規劃但尚未執行 | AI 預設（Step 1 輸出時） |
| `in-progress` | 使用者表示正在修復 | 使用者主動告知 |
| `verified` | 修復已完成且驗證通過 | 使用者提供驗證結果，AI 判定成功 |
| `deferred` | 使用者選擇延遲到下一輪 | 使用者主動決定 |

### 8.5 Exposure Status 轉換（由 Mobilization 觸發）

| 修復結果 | Exposure Current Status 變化 |
|---------|------------------------------|
| `verified`（修復成功） | `confirmed → mitigated` |
| `accepted`（接受風險） | `confirmed → accepted` |
| `planned` / `in-progress` / `deferred` | 維持 `confirmed`（不變） |

---

## 九、執行步驟

### Step 0：上下文載入與修復佇列建立（Context Loading & Remediation Queue）

**目的**：讀取所有前序階段資料，建立待處理修復佇列。

**動作**：
1. 讀取 `ctem-state.md` → 確認 Validation 狀態 `completed`
2. 設定 Mobilization 狀態為 `in_progress`
3. 讀取所有 Phase Summaries（Scoping、Discovery、Prioritization、Validation）
4. 讀取 `reports/assets/<id>.md` → 擷取 Exposure Registry、Remediation History、Current Risk Summary
5. **讀取 [remediation-sla.md](./references/remediation-sla.md)** — 載入 SLA 預設值與計算邏輯（**Required**）
6. 建立修復佇列：
   - 所有 Validation Status 為 `confirmed` 的暴露 → Remediation Action
   - 所有 Validation Status 為 `inconclusive` 的暴露 → Investigation Action
7. **返回輪檢查**：若 Remediation History 中有 `pending-verification` 的前輪行動 → 提醒使用者
8. **零佇列檢查**：若修復佇列為空（所有暴露皆為 false-positive） → 跳過 Steps 1–5，寫入空的 Mobilization Summary（「No remediation actions required — all exposures were false-positives」），直接進入 Step 6
9. 向使用者呈現佇列摘要：

> *「本輪 Mobilization 將處理以下暴露：」*
>
> **Remediation Actions（confirmed）：**
>
> | # | Exposure ID | Title | Adjusted Severity | Attack Path |
> |---|-------------|-------|-------------------|-------------|
> | 1 | EXP-001 | Apache Path Traversal | critical | AP-001 |
> | 2 | EXP-003 | FTP Anonymous Login | high | AP-001 |
>
> **Investigation Actions（inconclusive）：**
>
> | # | Exposure ID | Title | Adjusted Severity | Reason |
> |---|-------------|-------|-------------------|--------|
> | 1 | EXP-004 | ... | medium | Connection timeout during PoC |
>
> *「接下來將為每個暴露生成修復計畫。」*

### Step 1：修復計畫生成（Remediation Plan Generation）

**目的**：為每個 confirmed 暴露產出 Quick Fix 建議，為 inconclusive 暴露產出 investigation 建議。

**動作**：
1. 對每個 `confirmed` 暴露：
   - 根據暴露的 Type（vulnerability / misconfiguration / information-disclosure / outdated-software）、CVE、Affected Service 生成修復建議
   - 填入 Quick Fix（一句話）、Action Type（patch / configure / harden / replace）、Effort Level
   - 若需要更深入的修復指引，可參考 **[remediation-playbooks.md](./references/remediation-playbooks.md)**
2. 對每個 `inconclusive` 暴露：
   - 生成一條 Investigation Action（建議重新驗證的方法或額外掃描指令）
   - Action Type 固定為 `investigate`
   - Effort Level 通常為 `low`（重新掃描/驗證的成本較低）
3. 所有 Action 的預設 Status 為 `planned`
4. 呈現修復計畫表格供使用者確認或修改：

> *「修復計畫：」*
>
> | # | Action ID | Exposure ID | Title | Adjusted Severity | Action Type | Quick Fix | Effort |
> |---|-----------|-------------|-------|-------------------|-------------|-----------|--------|
> | 1 | ACT-001 | EXP-001 | Apache Path Traversal | critical | patch | Upgrade Apache to ≥ 2.4.51 | low |
> | 2 | ACT-002 | EXP-003 | FTP Anonymous Login | high | configure | Disable anonymous FTP access | trivial |
> | 3 | ACT-003 | EXP-005 | Credential File on FTP | critical | configure | Remove credential file + rotate exposed passwords | medium |
> | 4 | ACT-004 | EXP-004 | SSH Service (inconclusive) | medium | investigate | Re-run PoC from alternate network position | low |
>
> *「如需任何暴露的詳細修復步驟，請告訴我（例如『展開 ACT-001 的修復步驟』）。」*
> *「若有暴露您決定接受風險不修復，也請告訴我。」*

**Action ID 格式**：`ACT-NNN`（零補齊三位數），從 ACT-001 開始。每個 session 獨立編號。

**展開詳細步驟（使用者觸發）**：

當使用者要求展開某個 Action 的詳細修復步驟：
1. AI 讀取 `remediation-playbooks.md` 中對應暴露類型的修復模板
2. 根據目標主機的具體環境（OS、Service version、配置）填入具體指令
3. 呈現 step-by-step 修復指引

### Step 2：攻擊路徑修復策略（Attack Path Remediation Strategy）

**目的**：分析攻擊路徑，識別 chain-breaker，調整修復優先級。

**觸發條件**：僅在 Validation Summary 中存在 Attack Paths 時執行。若無攻擊路徑 → 跳過此步驟。

**動作**：
1. 讀取 Validation Summary 的 Attack Paths Identified 表格
2. 對每條攻擊路徑（AP-NNN）：
   a. 列出鏈上每個暴露及其 Action ID 與 Effort Level
   b. 識別 chain-breaker：Effort 最低者；相同則取鏈中最早位置者
   c. 標記 chain-breaker 身分
3. 呈現攻擊路徑修復策略供使用者確認：

> *「攻擊路徑修復策略：」*
>
> **AP-001**: EXP-003 (FTP Anonymous) → EXP-005 (Credential File) → EXP-001 (Admin Login via leaked creds)
>
> | Position | Exposure ID | Action | Effort | Chain-Breaker? |
> |----------|-------------|--------|--------|----------------|
> | 1 | EXP-003 | Disable anonymous FTP | trivial | **✓ Yes** |
> | 2 | EXP-005 | Remove credential file | medium | No |
> | 3 | EXP-001 | Rotate admin password | low | No |
>
> *「建議：修復 EXP-003（停用匿名 FTP，Effort: trivial）即可中斷整條攻擊鏈。」*
> *「但仍建議逐一修復鏈上所有暴露以建立縱深防禦。」*

4. 調整修復優先級：chain-breaker 暴露在同級 Adjusted Severity 中優先排序

### Step 3：風險接受處理（Risk Acceptance Processing）

**目的**：處理使用者決定不修復的暴露。

**觸發條件**：使用者在 Step 1 或後續對話中表示接受某暴露的風險時觸發。若使用者未提出接受風險 → 跳過此步驟。

**動作**：
1. 對每個被標記為接受風險的暴露，蒐集必要資訊：

> *「EXP-003 將被標記為 Accepted（風險接受）。請提供以下資訊：」*
>
> | 欄位 | 說明 |
> |------|------|
> | Justification | 為何接受此風險？ |
> | Approver | 風險接受的決策者（人名或角色） |
> | Review Date | 何時重新評估此決策？（建議：下一輪 CTEM 或特定日期） |

2. 若暴露為 critical 或 high → AI 主動提醒：

> *「⚠ 注意：EXP-001（Adjusted Severity: critical）被標記為 Accepted。此暴露已確認可利用，且位於攻擊路徑 AP-001 中。請確認此決策經過適當授權。」*

3. 記錄接受決策
4. 更新 Exposure Status：`confirmed → accepted`

### Step 4：行動指派與時程（Action Assignment & Timeline）

**目的**：為每個修復行動指派負責人和截止日期。

**動作**：
1. 讀取 `remediation-sla.md` 的預設 SLA 對照表
2. 為每個 Action 自動計算 Deadline（Plan Date + SLA Duration）
3. Pre-fill Owner：若 Scoping Summary 有 Owner / Team → 使用該值作為預設；否則留空讓使用者填入
4. 呈現完整的行動分派表格，供使用者確認或覆寫：

> *「行動分派與時程（SLA 預設值已套用，可逐項覆寫）：」*
>
> | # | Action ID | Exposure ID | Priority | Quick Fix | Owner | Deadline | Status |
> |---|-----------|-------------|----------|-----------|-------|----------|--------|
> | 1 | ACT-001 | EXP-001 | 1 | Upgrade Apache to ≥ 2.4.51 | <Owner> | 2026-04-06 | planned |
> | 2 | ACT-002 | EXP-003 | 2 | Disable anonymous FTP [chain-breaker: AP-001] | <Owner> | 2026-04-12 | planned |
> | 3 | ACT-003 | EXP-005 | 3 | Remove credential file + rotate passwords | <Owner> | 2026-04-12 | planned |
> | 4 | ACT-004 | EXP-004 | 4 | Re-run PoC from alternate network (investigation) | <Owner> | 2026-05-05 | planned |
>
> *「如需調整任一項的 Owner、Deadline 或 Priority，請告訴我。」*
> *「如需修改全域 SLA，也請告訴我。」*

5. 使用者確認後，鎖定最終排序與時程

**優先級排序**依據（見第五節 5.3）：
1. Adjusted Severity（降序）
2. Chain-Breaker 加分
3. Effort Level（升序）
4. Exposure ID（升序）

### Step 5：Session 內快速修復驗證（In-Session Quick-Fix Verification）— 可選

**目的**：使用者在 session 中完成的快速修復，提供驗證流程。

**觸發條件**：使用者主動表示已修復某個暴露。**不是流程中的固定步驟**——為使用者觸發的互動循環。

**互動循環**：

```
REPEAT (user-triggered):
  1. 使用者：「我修好了 EXP-XXX」或「我要現在修 ACT-NNN」
  2. AI 根據該暴露的修復建議，生成驗證指令
  3. 使用者執行並回報結果
  4. AI 判定修復是否成功：
     - 成功 → Action Status = verified, Exposure Status = mitigated
     - 失敗 → 維持 planned, 建議替代方案
  5. 若有 Status 變更 → 立即更新 Asset Profile
UNTIL:
  - 使用者無更多修復報告
  - 或使用者選擇完成 Mobilization
```

**驗證指令生成**：參考 Phase 4 (Validation) 使用的相同驗證方法——驗證修復就是驗證暴露是否還存在。可參考 `remediation-playbooks.md` 中每種修復類型的驗證步驟。

### Step 6：寫入與產出（Write & Output）

**目的**：將 Mobilization 結果寫入所有目標位置。

#### 6a — 更新 Asset Profile (`reports/assets/<id>.md`)

**Remediation History 表**：
- 為每個 Action 新增一列：
  - Exposure ID
  - Action Taken（Quick Fix 描述）
  - Date（Plan Date 或 Verified Date）
  - Verified In (Session)：若 in-session verified → 填入當前 Session ID；否則留空
  - Result：`resolved`（verified 成功）、`pending-verification`（planned/in-progress/deferred）、`accepted`（風險接受）

**Exposure Registry 表 — Current Status 更新**：
- `verified` 修復 → `confirmed → mitigated`
- 風險接受 → `confirmed → accepted`
- `planned` / `in-progress` / `deferred` → 維持 `confirmed`（不變）

**Current Risk Summary 表**：
- 在所有 Status 變更後重算：
  - `Overall Risk Level`：所有非 `false-positive`、非 `mitigated` 的 open 暴露中最高 Adjusted Severity
  - `Open Exposures`：重新計數
  - `Trend`：與 Validation 後的 Overall Risk Level 比較

> **注意**：`accepted` 暴露仍計入 Overall Risk Level 的計算——它是已知風險，只是不修復。若要呈現排除 accepted 後的風險等級，在 Mobilization Summary 的 Risk Overview 中提供 `Risk Level (excl. accepted)` 作為參考。

#### 6b — 寫入 Mobilization Summary (`ctem-state.md`)

格式見「十、資訊傳遞機制」。

#### 6c — 更新 Phase Status (`ctem-state.md`)

- 設定 Mobilization 為 `completed`
- 填入 Key Findings Summary 和 Last Updated
- 新增 Transition Log 項目

---

## 十、資訊傳遞機制

### 10.1 持久化位置

| 位置 | 內容 | 生命週期 |
|------|------|---------|
| `ctem-state.md` → `### Mobilization Summary` | 本輪 Mobilization 結果摘要 | 單輪 session |
| `reports/assets/<id>.md` → Remediation History | 修復行動紀錄 | 跨輪次長期維護 |
| `reports/assets/<id>.md` → Exposure Registry → Current Status | 暴露狀態更新（mitigated / accepted） | 跨輪次長期維護 |
| `reports/assets/<id>.md` → Current Risk Summary | 修復後風險狀態 | 跨輪次（每輪覆寫最新值） |

### 10.2 Mobilization Summary 格式（寫入 ctem-state.md）

```markdown
### Mobilization Summary

**Plan Date**: <ISO 8601 date>
**Remediation Model**: Severity-based SLA + Attack Path Chain-Breaker Analysis

#### Remediation Actions

| # | Action ID | Exposure ID | Title | Adjusted Severity | Action Type | Quick Fix | Effort | Owner | Deadline | Status |
|---|-----------|-------------|-------|-------------------|-------------|-----------|--------|-------|----------|--------|
| 1 | ACT-001 | EXP-001 | Apache Path Traversal | critical | patch | Upgrade Apache to ≥ 2.4.51 | low | sysadmin | 2026-04-06 | planned |
| 2 | ACT-002 | EXP-003 | FTP Anonymous Login | high | configure | Disable anonymous FTP | trivial | sysadmin | 2026-04-12 | verified |

#### Attack Path Remediation

| # | Path ID | Chain | Chain-Breaker | Fix Action | Impact |
|---|---------|-------|---------------|------------|--------|
| 1 | AP-001 | EXP-003 → EXP-005 → EXP-001 | EXP-003 (ACT-002) | Disable anonymous FTP (trivial) | Breaks full kill chain to admin access |

#### Risk Acceptance Log

<!-- If no acceptances: "No risk acceptances in this session." -->

| # | Exposure ID | Title | Adjusted Severity | Justification | Approver | Review Date |
|---|-------------|-------|-------------------|---------------|----------|-------------|
| 1 | EXP-006 | ... | medium | Cost of fix exceeds business impact | CTO | 2026-07-01 |

#### Verification Results

<!-- If no in-session verifications: "No in-session verifications performed." -->

| # | Action ID | Exposure ID | Verification Method | Result | New Status |
|---|-----------|-------------|---------------------|--------|------------|
| 1 | ACT-002 | EXP-003 | FTP anonymous login attempt → rejected | resolved | mitigated |

#### Mobilization Statistics

| Metric | Value |
|--------|-------|
| Total Actions Planned | N |
| By Action Type | patch: N, configure: N, harden: N, replace: N, investigate: N |
| By Status | planned: N, in-progress: N, verified: N, deferred: N |
| Risk Acceptances | N |
| Attack Paths Addressed | N chain-breakers identified |
| Quick Fixes Verified In-Session | N |
| Overall Risk Level (post-mobilization) | <recalculated> |
| Risk Level (excl. accepted) | <risk level excluding accepted exposures> |
| Risk Level Change | <pre-mobilization> → <post-mobilization> |
```

### 10.3 下游銜接

- **Report Generation**（ctem-report-guide）啟動時 → 讀取 `### Mobilization Summary` → 編入 Session Report
- Mobilization Summary 的 Remediation Actions 表 → Session Report 的 `Remediation Actions` 區塊
- Risk Acceptance Log → Session Report 的 Notes 或專屬區塊

### 10.4 Session Report 資料

Mobilization 產出的以下資料由報告產生階段編入 Session Report：
- `Remediation Actions` 表：直接複製 Action 清單
- `Phase Results Summary` → Mobilization 行：填入修復行動統計

Mobilization **不直接寫入 Session Report 檔案** — Session Report 在五階段全部完成後由報告產生流程統一建立。

---

## 十一、互動風格

### 11.1 主要互動方式

採 **hybrid** 模式（沿用前序階段慣例）：

1. **自動化部分**：AI 讀取所有上游資料，自動算好 SLA、Priority、Chain-Breaker，預填 Quick Fix
2. **使用者確認部分**：修復計畫表格、Owner/Deadline 指派、風險接受——全部需使用者確認
3. **按需展開部分**：個別暴露的詳細修復步驟——使用者要求才生成
4. **使用者觸發部分**：in-session quick-fix verification——使用者說「我修好了」才觸發

### 11.2 互動選項

整個 Mobilization 過程中，使用者可隨時：

| 動作 | 說明 |
|------|------|
| 展開詳細步驟 | 「展開 ACT-001 的修復步驟」→ AI 生成 step-by-step 指引 |
| 接受風險 | 「EXP-003 接受風險」→ 觸發風險接受流程（Step 3） |
| 覆寫 SLA | 「EXP-001 延長到 7 天」或「全域 critical SLA 改為 48h」→ 更新 Deadline |
| 修改 Owner | 「ACT-002 指派給 network-team」→ 更新 Owner |
| 報告修復完成 | 「我修好了 EXP-003」→ 觸發 in-session verification（Step 5） |
| 完成 | 「Mobilization 完成」→ 進入 Step 6 寫入產出 |

---

## 十二、完成條件與 Checklist

Mobilization skill 在所有步驟做完後，產出以下 checklist：

```markdown
## Mobilization Completion Checklist

- [ ] Validation Summary and upstream data read
- [ ] Remediation queue built (confirmed + inconclusive exposures)
- [ ] Remediation plan generated for all confirmed exposures (Quick Fix + Action Type + Effort)
- [ ] Investigation actions generated for inconclusive exposures (if any)
- [ ] Attack path remediation strategy applied with chain-breakers identified (if attack paths exist)
- [ ] Risk acceptances processed with structured justification (if any)
- [ ] Action assignment and timeline completed (Owner + Deadline for all actions)
- [ ] In-session quick-fix verifications completed (if any were triggered)
- [ ] Asset Profile Remediation History and Current Status updated (reports/assets/)
- [ ] Current Risk Summary recalculated (reports/assets/)
- [ ] Mobilization Summary written to ctem-state.md
```

**職責劃分**：
- Mobilization skill：執行修復規劃、產出 checklist、寫入修復結果與更新相關檔案
- ctem-flow：驗證 checklist、更新 Phase Status、執行階段轉換、觸發 Report Generation
- Report Generation（ctem-report-guide）：五階段完成後建立 Session Report

**完成訊息範例**：

> **Mobilization 完成。** 共規劃 N 項修復行動（patch: X, configure: X, harden: X, investigate: X）。N 項已在 session 中驗證修復。M 項風險接受。P 條攻擊路徑已識別 chain-breaker。Overall Risk Level: \<level\>。Mobilization Summary 已寫入。
>
> **五階段全部完成。** 準備好後可進行 Report Generation 以建立 Session Report。

---

## 十三、參考檔案

獨立存放於 `references/` 目錄，SKILL.md 透過相對連結引用。

### 13.1 references/remediation-sla.md

SLA 核心參考，包含：
- **Severity → SLA 預設對照表**：critical=24h, high=7d, medium=30d, low=90d, info=N/A
- **SLA 覆寫規則**：全域覆寫與逐項覆寫的邏輯
- **Deadline 計算邏輯**：Plan Date + SLA Duration，ISO 8601 格式
- **Regulatory Context 影響**：常見合規框架（PCI-DSS、GDPR、ISO 27001）對修復時程的建議
- **Effort Level 定義表**：trivial / low / medium / high / critical 的標準與典型時間

> **SKILL.md 必須在 Step 0 載入此檔案。**

### 13.2 references/remediation-playbooks.md

修復指引參考，按暴露類型組織，包含：
- **vulnerability（漏洞）修復模板**：
  - 軟體升級指引（apt/yum/apk 指令模板）
  - 常見 CVE 的針對性修復建議（Apache、OpenSSH、MySQL 等）
  - 臨時緩解措施（WAF 規則、防火牆 ACL、服務停用）
- **misconfiguration（配置錯誤）修復模板**：
  - SSH 強化配置（ciphers、MACs、key exchange）
  - FTP 安全配置（停用匿名、chroot jail、TLS 啟用）
  - Web server 強化（header 設定、目錄列舉關閉、版本隱藏）
- **information-disclosure（資訊洩漏）修復模板**：
  - 移除敏感檔案（credential files、backup files、.git 目錄）
  - 版本橫幅隱藏（ServerTokens、ServerSignature）
  - 錯誤頁面自訂化（避免堆疊追蹤洩漏）
- **outdated-software（過期軟體）修復模板**：
  - 軟體升級路徑指引
  - EOL 軟體替代方案
  - 無法立即升級時的臨時緩解措施
- **每種修復類型的驗證步驟**：修復後如何驗證是否生效

> SKILL.md 在使用者要求展開詳細步驟（Step 1 展開式）或 in-session verification 時載入此檔案（on-demand）。

---

## 十四、與 ctem-flow 的互動點

Mobilization 完成後，ctem-flow 負責：

| 動作 | 說明 |
|------|------|
| 確認五階段全部 `completed` | 檢查 Phase Status 表 |
| 觸發 Report Generation | 依 `ctem-report-guide.instructions.md` 建立 Session Report |
| 更新 Asset Profile → Risk Trend Log | 新增本輪的一列記錄 |
| 重置 `ctem-state.md`（可選） | 歸檔後重置為空白模板 |

Mobilization **不需要執行回溯檢查**——它是最後一個階段。回溯只可能由使用者主動觸發（例如：在修復過程中發現了新的攻擊面，使用者決定回到 Discovery → 由 ctem-flow 處理）。

---

## 十五、術語表

| 術語 | 等級 | 慣例 |
|------|------|------|
| **Severity** levels | `info` < `low` < `medium` < `high` < `critical` | 對齊 CVSS v3.1 |
| **Business Criticality** levels | `low` < `moderate` < `high` < `critical` | 對齊 FIPS 199 / Scoping |
| **Action Status** | `planned`, `in-progress`, `verified`, `deferred` | Mobilization 修復行動狀態 |
| **Exposure Status** | `open`, `confirmed`, `false-positive`, `mitigated`, `accepted`, `reopened` | Exposure Registry 中的持久狀態 |
| **Action Type** | `patch`, `configure`, `harden`, `replace`, `investigate` | 修復行動分類 |
| **Effort Level** | `trivial`, `low`, `medium`, `high`, `critical` | 工程量估算 |

> **Severity** 用 `medium`；**Business Criticality** 用 `moderate`。兩者為不同維度，不互通。

---

## 十六、檔案影響範圍

建置 Mobilization skill 時需要建立/修改的檔案：

| 檔案 | 動作 | 說明 |
|------|------|------|
| `.github/skills/ctem-5-mobilization/SKILL.md` | 重寫 | 主要 prompt 邏輯 |
| `.github/skills/ctem-5-mobilization/SKILL.zh-TW.md` | 新增 | 繁體中文版（輔助閱讀） |
| `.github/skills/ctem-5-mobilization/references/remediation-sla.md` | 新增 | SLA 預設值與計算邏輯 |
| `.github/skills/ctem-5-mobilization/references/remediation-sla.zh-TW.md` | 新增 | 繁體中文版 |
| `.github/skills/ctem-5-mobilization/references/remediation-playbooks.md` | 新增 | 修復指引參考 |
| `.github/skills/ctem-5-mobilization/references/remediation-playbooks.zh-TW.md` | 新增 | 繁體中文版 |
| `FRAMEWORK-ALIGNMENT.md` | 更新 | Phase 5 區塊從「Planned」更新為實際實作內容 |
| `FRAMEWORK-ALIGNMENT.zh-TW.md` | 更新 | 繁體中文版同步更新 |

> 注意：`ctem-state.md` 和 `reports/` 模板不需要修改 — 已有預留的 `### Mobilization Summary` 位置和對應欄位（Remediation History、accepted 狀態等）。

---
