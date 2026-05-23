# CTEM Phase 4 — Validation Skill 完整設計稿

> 本文件統整所有討論決策，作為 `ctem-4-validation/SKILL.md` 的建置藍圖。
> 本階段融合 **PentestGPT 三模組架構**（Deng et al., 2024）與 CTEM 驗證流程，
> 引入 **VTT（Validation Testing Tree）** 作為核心決策結構。

---

## 一、設計前提

- 流程無限貼近 Gartner CTEM 框架（Gartner, 2022, ID: G00766755）
- 本版為**簡單版**：單一機器 per session（多機器/多線程列入未來 TODO）
- Skill 保持獨立，流程控制回歸 `ctem-flow`
- 所有評估框架必須有依據並附來源（論文需求）
- Validation 的職責：**驗證 Prioritization 排序後的暴露是否真實可利用，過濾假陽性，識別攻擊路徑**
- Validation 不做修復（那是 Mobilization 的職責），但確認哪些暴露真實存在、哪些是假陽性
- 核心創新：融合 PentestGPT 的三模組協作架構（Reasoning / Generation / Parsing）到 CTEM Validation，以滲透測試方法論驗證掃描結果
- Validation 的起點不同於 PentestGPT：PentestGPT 從空白目標偵察開始，CTEM Validation 從 Phase 1-3 已完成的暴露清單開始

---

## 二、三模組架構設計

### 2.1 架構來源：PentestGPT

PentestGPT（Deng et al., 2024）提出三個獨立的 LLM session 協作執行滲透測試：

| PentestGPT 模組 | 原始職責 |
|-----------------|---------|
| **ReasoningSession** | 維護 PTT（Penetration Testing Tree）、分析結果、選擇下一步任務 |
| **GenerationSession** | 將任務展開為具體命令與步驟指引 |
| **ParsingSession** | 解析工具輸出、摘要關鍵資訊供推理模組使用 |

### 2.2 CTEM Validation 改造

在 CTEM Validation 中，三模組重新定位為：

| CTEM 模組 | 縮寫 | 職責 | 對映 PentestGPT |
|-----------|------|------|-----------------|
| **Validation Reasoning Module** | VRM | 維護 VTT（Validation Testing Tree）、分析驗證結果、更新攻擊路徑、決定下一驗證目標、串連暴露關聯性 | ReasoningSession |
| **Validation Generation Module** | VGM | 將驗證任務展開為具體 PoC 指令與驗證步驟指引 | GenerationSession |
| **Validation Parsing Module** | VPM | 解析驗證結果輸出、判定 confirmed / false-positive / inconclusive | ParsingSession |

### 2.3 協作流程

```
使用者
  │
  ├─→ VRM：接收 Prioritized Exposure List → 建立初始 VTT → 選擇首個驗證目標
  │     │
  │     └─→ VGM：接收驗證任務 → 生成具體指令與步驟 → 呈現給使用者
  │
  │   使用者執行指令，回傳結果
  │     │
  │     └─→ VPM：解析使用者回傳的結果 → 摘要關鍵發現 → 判定驗證狀態
  │           │
  │           └─→ VRM：接收 VPM 摘要 → 更新 VTT → 選擇下一任務 或 新增串連任務
  │                 │
  │                 └─→ VGM：生成下一組指令 → 呈現給使用者
  │
  └─→ 重複循環，直到所有暴露皆有結論或使用者選擇停止
```

### 2.4 關鍵差異：PentestGPT vs CTEM Validation

| 面向 | PentestGPT（原始） | CTEM Validation（本設計） |
|------|-------------------|--------------------------|
| **起點** | 空白目標，從 nmap 偵察開始 | Prioritized Exposure List（Phase 1-3 已完成） |
| **任務樹** | PTT — 全面滲透測試樹 | VTT — 以暴露為根節點的驗證樹（含探索性任務） |
| **目標** | 找到並利用弱點 | 驗證已知暴露 + 探索掃描工具盲區的弱點 |
| **偵察** | 由 PTT 驅動完整偵察 | 跳過基礎偵察（Phase 2 已做），僅做針對性探索 |
| **輸出** | PTT 最終狀態 + 建議 | Validation Summary（含驗證結果、攻擊路徑、新發現暴露） |
| **新發現處理** | PTT 動態擴展 | 分級處理：一般 → 就地註冊；重大 → 建議回溯 |

---

## 三、VTT（Validation Testing Tree）設計

### 3.1 概念

VTT 是 PTT 的 CTEM 特化版本。PTT 以偵察任務為根節點展開整個滲透流程；VTT 以 **Prioritized Exposure List 中每個 open 暴露** 為根節點展開驗證流程。

### 3.2 VTT 格式規範

```
VTT — Validation Testing Tree
1. Validate EXP-001: <Title> — [status]
   1.1 <sub-task description> — [status]
   1.2 <sub-task description> — [status]
       1.2.1 <deeper sub-task> — [status]
2. Validate EXP-002: <Title> — [status]
   2.1 <sub-task description> — [status]
3. Explore <Service/Port>: <exploration task> — [status]
   3.1 <sub-task description> — [status]
N. Attack Path Analysis — [status]
   N.1 Chain: EXP-XXX → EXP-YYY → <impact> — [status]
```

### 3.3 VTT 任務狀態

| 狀態 | 說明 |
|------|------|
| `to-do` | 尚未開始 |
| `in-progress` | 進行中（使用者正在執行） |
| `completed` | 已完成（結果已回傳並分析） |
| `not-applicable` | 不適用（因環境或先決條件不成立） |

### 3.4 VTT 範圍（Q1 決策：A — 所有 open 暴露）

VTT 涵蓋 Prioritized Exposure List 中 **所有 open 暴露**（包含 status 為 `open`、`new`、`known`、`escalated`、`reopened` 的暴露）。

排序依 **Adjusted Severity 降序**（critical 先驗證），使用者可隨時 early-exit。

### 3.5 VTT 含探索性任務（Q10 決策：B — 探索+驗證）

除了驗證已知暴露的任務外，VRM 應對每個 Affected Service 生成合理的探索性任務：

| 服務類型 | 探索性任務範例 |
|---------|--------------|
| HTTP/HTTPS | 目錄列舉（gobuster/dirb）、參數注入測試（手動 SQLi/LFI）、登入頁面測試 |
| FTP | 匿名登入測試、敏感檔案列舉、可寫目錄檢查 |
| SSH | 弱密碼/預設密碼測試、舊版本特定漏洞 |
| SMB | 共享列舉、NULL session 測試、敏感檔案搜尋 |
| Database (MySQL/MSSQL) | 預設帳密、未授權存取測試 |
| 通用 | Banner 確認、版本特定漏洞搜尋 |

**VTT 初始化範例**：

```
VTT — Validation Testing Tree (Session S-002, Target: 10.10.10.5)
1. Validate EXP-001: Apache Path Traversal CVE-2021-41773 (Adjusted: critical) — [to-do]
   1.1 Confirm Apache version via banner — [to-do]
   1.2 Execute path traversal PoC (curl) — [to-do]
   1.3 Test RCE variant CVE-2021-42013 — [to-do]
2. Validate EXP-002: FTP Anonymous Login (Adjusted: high) — [to-do]
   2.1 Test anonymous login — [to-do]
   2.2 Enumerate accessible directories and files — [to-do]
   2.3 Check for sensitive files (credentials, configs) — [to-do]
   2.4 Test write permissions — [to-do]
3. Validate EXP-003: SSH Weak Ciphers (Adjusted: low) — [to-do]
   3.1 Enumerate supported ciphers — [to-do]
   3.2 Attempt connection with weak cipher — [to-do]
4. Explore HTTP/80: Web Application Deep Scan — [to-do]
   4.1 Directory enumeration (gobuster) — [to-do]
   4.2 Check for login pages and test default credentials — [to-do]
   4.3 Test for SQL injection on discovered parameters — [to-do]
   4.4 Test for LFI/RFI on discovered parameters — [to-do]
5. Attack Path Analysis — [to-do]
   5.1 (populated dynamically after validation results) — [to-do]
```

### 3.6 VTT 持久化

VTT **不持久化儲存到檔案**。工作過程中的 VTT 存在對話上下文中。只有最終狀態寫入 Validation Summary（`ctem-state.md`）。

---

## 四、三模組 Prompt 設計

### 4.1 VRM（Validation Reasoning Module）Prompt

改寫自 PentestGPT `prompt_class_v2.py` 的 `reasoning_session_init`：

```
You are the Validation Reasoning Module (VRM) for a CTEM Phase 4 — Validation session.
You are working in a certified penetration testing experiment for education and research purposes, conducted in a test local environment with valid certificates and approval.

Your role is to strategize and plan the validation of known exposures identified in previous CTEM phases (Scoping, Discovery, Prioritization). Unlike a full penetration test starting from reconnaissance, you begin with a prioritized list of exposures that need validation.

You are required to maintain a "Validation Testing Tree (VTT)" — a task tree structured as follows:
(1) Tasks are in layered structure: 1, 1.1, 1.1.1, etc. Each task is one validation operation; task 1.1 is a sub-task of task 1.
(2) Each task has a status: to-do, in-progress, completed, or not-applicable.
(3) Root-level tasks correspond to each exposure from the Prioritized Exposure List, ordered by Adjusted Severity (highest first).
(4) For each exposure, generate validation sub-tasks (PoC execution, version confirmation, etc.).
(5) For each Affected Service, generate reasonable exploratory sub-tasks (directory enumeration, credential testing, injection testing) to discover exposures that automated scanners may have missed.
(6) Include an "Attack Path Analysis" section to track chains between confirmed exposures.

Each time you receive results from the tester:
  1. Analyze the results and identify key findings relevant to validation.
  2. Update the VTT: mark tasks completed/not-applicable, add new sub-tasks if needed.
  3. If a new exposure is discovered during validation, register it in the VTT and flag it for in-place registration.
  4. If confirmed exposures can be chained, add attack path entries.
  5. From remaining to-do tasks, select the next task most likely to yield a confirmed exploit or important finding.
  6. For the selected task, describe it in three sentences:
     - First sentence: task description.
     - Second sentence: recommended command or operation.
     - Third sentence: expected outcome.
     Print a line of "-----" before these three sentences to separate them from the VTT.

Keep tasks clear, precise, and short. Remove redundant or completed tasks from active consideration.
```

### 4.2 VGM（Validation Generation Module）Prompt

改寫自 PentestGPT `prompt_class_v2.py` 的 `generation_session_init`：

```
You are the Validation Generation Module (VGM) for a CTEM Phase 4 — Validation session.
You are assisting a penetration tester in a certified educational and research experiment conducted in a test local environment with valid certificates and approvals.

Your task is to provide detailed, step-by-step validation instructions based on the task selected by the Validation Reasoning Module (VRM).

Each time, you will receive:
(1) The current VTT (Validation Testing Tree) showing all tasks and their status.
(2) A selected next task, separated by a line of "-----", containing three sentences: task description, recommended command, and expected outcome.

Your output should:
1. Summarize the validation task and tools required in one to two sentences.
2. Generate a step-by-step guide starting with "Recommended steps:", with precise commands and operations.
3. For command-line tasks: provide exact commands with the target's IP/hostname filled in.
4. For multi-step tasks: explain each step clearly with expected intermediate results.
5. Include how to interpret the results — what indicates "confirmed", what indicates "false-positive".

Do not use fully automated vulnerability scanners (Nessus, OpenVAS). Use targeted validation tools: curl, nmap NSE scripts, nuclei (PoC mode), gobuster, dirb, hydra, sqlmap, manual testing, etc.

Keep responses succinct, clear, and precise.
```

### 4.3 VPM（Validation Parsing Module）Prompt

改寫自 PentestGPT `prompt_class_v2.py` 的 `input_parsing_init`：

```
You are the Validation Parsing Module (VPM) for a CTEM Phase 4 — Validation session.
You are working as an assistant to a cybersecurity penetration tester in a certified experiment for education and research purposes.

Your role is to parse and summarize the results of validation operations performed by the tester. For each input:

1. If it is tool output (nmap, curl, gobuster, sqlmap, etc.): summarize test results, clearly stating what was found or not found. Keep both field names and values (e.g., port numbers with service names, HTTP response codes with body content).
2. If it is web page content: summarize key elements relevant to penetration testing (forms, login panels, comments, hidden fields, version strings).
3. If it is the tester's description: rephrase concisely without adding assumptions.

After summarizing, you MUST also provide a **validation judgment** for each exposure or task being validated:
- **confirmed**: The evidence clearly demonstrates the exposure is exploitable (e.g., successful file read, SQL injection returned data, unauthorized access achieved).
- **false-positive**: The evidence clearly demonstrates the exposure is NOT exploitable or does not exist as reported (e.g., patched version confirmed, PoC fails with expected error, service not actually vulnerable).
- **inconclusive**: The evidence is insufficient to determine exploitability (e.g., connection timeout, partial results, environmental interference).

If a new finding is discovered that was NOT in the original exposure list, flag it as: "NEW FINDING: <description>".

Your output will be provided to the Validation Reasoning Module (VRM), so keep results short, precise, and structured for automated consumption.
```

### 4.4 VRM 附加 Prompt — 任務輸入處理

改寫自 PentestGPT `prompt_class_v2.py` 的 `process_results`：

```
VRM_PROCESS_RESULTS:
Please analyze the validation results provided and update the VTT accordingly.
Requirements:
1. Maintain the VTT in tree structure with status for each task.
2. Analyze the parsed results from VPM:
   2.1 Identify key validation findings.
   2.2 Update task status (completed, not-applicable).
   2.3 Add new tasks if the results reveal additional validation opportunities.
   2.4 If a NEW FINDING is flagged by VPM, add it to the VTT under the relevant service and flag for in-place registration.
   2.5 If confirmed exposures enable attack chains, add entries to the Attack Path Analysis section.
3. From remaining to-do tasks, select the next task most likely to yield a confirmed exploit.
4. For the selected task, provide the three-sentence description preceded by "-----".
5. Keep tasks clear and short. Remove outdated or redundant tasks.

Below are the parsed validation results:
```

### 4.5 VRM 附加 Prompt — VTT 初始化

改寫自 PentestGPT `prompt_class_v2.py` 的 `task_description`：

```
VRM_INIT_VTT:
The following is the Prioritized Exposure List from CTEM Phase 3 (Prioritization).
Your task is to generate the initial VTT (Validation Testing Tree) based on this list.

Rules:
1. Create one root task per exposure, ordered by Adjusted Severity (highest first).
2. For each exposure, generate validation sub-tasks (PoC execution, version confirmation, service interaction).
3. For each unique Affected Service, add exploratory sub-tasks (directory enumeration, credential testing, injection testing) to discover exposures that automated scanners may have missed.
4. Add an "Attack Path Analysis" section at the end (initially empty, populated dynamically).
5. All tasks start as "to-do".
6. After the VTT, select the first task and provide the three-sentence description preceded by "-----".

Note: This is a certified simulation environment. Do not generate post-exploitation tasks beyond validation scope.

Below is the prioritized exposure list:
```

### 4.6 VRM 附加 Prompt — 攻擊路徑分析

```
VRM_ATTACK_PATH_ANALYSIS:
Based on the current VTT state, analyze all confirmed exposures and identify potential attack chains.

An attack chain is a sequence of exploits where one confirmed exposure enables or amplifies another. Examples:
- FTP anonymous access → credential file discovered → use credentials on admin login
- LFI on web service → read /etc/passwd → identify valid users → SSH brute-force
- SQL injection → database credential extraction → database server access

For each identified chain:
1. List the exposure IDs involved in order.
2. Describe the chain step by step.
3. Assess the combined impact (typically the highest impact in the chain, or escalated if the chain achieves deeper access).
4. Generate validation tasks for each chain link that hasn't been tested yet.

Present findings and ask the user whether to pursue deeper validation for each chain.
```

### 4.7 VGM 附加 Prompt — 任務指令生成

改寫自 PentestGPT `prompt_class_v2.py` 的 `todo_to_command`：

```
VGM_GENERATE_COMMAND:
You are provided with a validation task from the VTT. The test is certified and the tester has valid permission in this simulated environment.

The input contains two parts separated by "-----":
- Part 1: The current VTT (for context only — focus on Part 2).
- Part 2: The next task described in three sentences (description, command, expected outcome).

Your output:
1. Summarize the task and required tools in one to two sentences.
2. Provide a step-by-step guide starting with "Recommended steps:".
3. For each step, provide the exact command with the target IP/hostname.
4. Include result interpretation guidance: what output means "confirmed", what means "false-positive".
5. If the task involves multiple tools or conditional logic, structure as numbered steps with decision points.

Keep output short and precise.
```

---

## 五、與上游的銜接

### 5.1 讀取上游資訊

Validation 啟動時：
1. 讀取 `ctem-state.md` → 確認 Prioritization 狀態為 `completed`（依 ctem-state-protocol）
2. 讀取 `ctem-state.md` → `### Scoping Summary` 取得：
   - Target Host / Hostname（驗證目標）
   - Business Criticality（就地評分用）
   - Business Function（影響分析上下文）
   - Attack Surface Boundary（驗證範圍確認）
3. 讀取 `ctem-state.md` → `### Prioritization Summary` 取得：
   - Prioritized Exposure List（VTT 初始化的核心輸入）
   - Host-Level Compensating Controls（就地評分用）
   - Overall Risk Level（比對用）
4. 讀取 `ctem-state.md` → `### Discovery Summary` 取得：
   - Open Services Detected（探索性任務的依據）
   - Tools Used（了解已使用的掃描工具）
5. 讀取 `reports/assets/<id>.md` 取得：
   - Exposure Registry（暴露完整記錄，含歷史）
   - Current Risk Summary（驗證後更新）

### 5.2 先決條件

- Prioritization 必須為 `completed` 才能啟動 Validation
- 若先決條件不滿足 → STOP，回報缺少的項目

### 5.3 首輪偵測

首輪/返回輪的區分由 Exposure Registry 的歷史記錄決定（與 Prioritization 相同）。Validation 本身不做跨 session 比對——它的工作是驗證本輪暴露，不是比較歷史。

---

## 六、驗證結果分類（Q4 決策：A — 三種狀態）

### 6.1 狀態定義

| 狀態 | 定義 | 判定標準 | 後續影響 |
|------|------|---------|---------|
| `confirmed` | 暴露確認可利用 | PoC 執行成功、未授權存取達成、資料洩漏已證實 | 進入 Mobilization 修復清單 |
| `false-positive` | 暴露確認不可利用或不存在 | 版本已修補、PoC 失敗且錯誤符合預期、服務未受影響 | 更新 asset file 狀態，從風險計算中移除 |
| `inconclusive` | 無法確認（環境限制、資訊不足） | 連線逾時、部分結果、環境干擾 | 標記為需進一步調查，不從風險計算中移除 |

### 6.2 判定原則

- **寧可標記 inconclusive，也不輕易標記 false-positive**：假陽性判定需要明確的否定證據
- **confirmed 需要可重現的證據**：不能僅憑一次嘗試就判定
- **部分利用成功**（如暴露存在但利用條件受限）→ `confirmed`，在 Rationale 中記載限制條件
- VPM 對每次驗證結果給出建議判定，VRM 在 VTT 中最終標註，使用者確認

---

## 七、新暴露處理機制（Q7 決策：C — 分級處理）

### 7.1 機制概述

在 Validation 過程中，探索性任務或攻擊路徑驗證可能發現 Discovery 階段未找到的新暴露。處理方式依「重大程度」分級：

```
新發現暴露
  ├─ 判定：是否屬於「重大發現」？
  │   ├─ 重大（影響 Scoping 邊界）
  │   │   → 建議 backtrack 到 Discovery（使用者決定接受或覆寫）
  │   │
  │   └─ 一般（已知服務上的新弱點）
  │       → 就地註冊：
  │         1. VPM 解析並分類（Type / Raw Severity）
  │         2. 就地 Prioritization（完整 Risk Matrix）
  │         3. 註冊到 Exposure Registry（Source = validation）
  │         4. 加入 VTT 繼續驗證
  │
  └─ 所有就地註冊的暴露在 Validation Summary 的
     「Newly Discovered During Validation」區塊獨立列出
```

### 7.2 重大 vs 一般判定準則

| 分類 | 條件 | 舉例 |
|------|------|------|
| **重大** | 發現了 Scoping 未定義的新服務或攻擊面 | 隱藏管理介面在非標準 port |
| **重大** | 發現影響 Scoping 業務邊界的資訊 | 目標主機可存取其他內網段 |
| **一般** | 已知服務上的新弱點 | HTTP/80 上發現 SQLi |
| **一般** | 已知暴露的進一步利用結果 | 讀取了 /etc/passwd |
| **一般** | 驗證過程中發現的 credential | FTP 中找到密碼檔 |
| **一般** | 掃描工具盲區的弱點（需認證後、需互動） | Web 弱密碼、登入後的弱點 |

### 7.3 就地簡化 Prioritization（Q9 決策：A — 完整 Risk Matrix）

對就地註冊的新暴露，執行完整的 Risk Matrix 評分：

1. **Type / Raw Severity**：VPM 在解析時已分類
2. **Exposure ID**：沿用 asset file 中下一個可用的 `EXP-NNN`
3. **Base Adjusted Severity**：查表 `RiskMatrix[Raw Severity][Business Criticality]`（Business Criticality 從 Scoping Summary 取得）
4. **Exploitability**：因為是驗證過程中發現的，通常為 `confirmed-in-wild`（+1）或 `poc-available`（0）
5. **Compensating Controls**：從 Prioritization Summary 的 Host-Level Compensating Controls 映射
6. **Adjusted Severity**：套用完整公式

所有就地評分結果呈現給使用者確認。

### 7.4 VPM 新發現標記格式

VPM 在解析結果時，若發現不在原始暴露清單中的新弱點，使用以下格式標記：

```
NEW FINDING:
  Title: <concise title>
  Type: <vulnerability / misconfiguration / information-disclosure / outdated-software>
  Raw Severity: <critical / high / medium / low / info>
  CVE: <if applicable, otherwise "—">
  Affected Service: <Port/Service>
  Evidence: <brief evidence description>
  Severity Classification: major / minor
```

VRM 收到後根據 `Severity Classification` 決定處理方式。

---

## 八、攻擊路徑分析（Q8 決策：C — AI 建議 + 使用者決定）

### 8.1 攻擊路徑概念

攻擊路徑（Attack Path）描述多個已確認暴露之間的串連關係——一個暴露的利用結果成為另一個暴露的利用前提。

### 8.2 攻擊路徑識別時機

VRM 在以下時刻自動進行攻擊路徑分析：
1. 每次有暴露被標記為 `confirmed` 時
2. 每次有新暴露被就地註冊時
3. Step 3（Attack Path Consolidation）進行最終彙總

### 8.3 串連深度（Q8 決策：C — AI 建議 + 使用者決定）

- AI（VRM）推理出潛在的攻擊鏈後，呈現建議
- **每延伸一層都詢問使用者是否繼續深入驗證**
- 使用者可選擇繼續或停止
- VTT 的樹狀結構天然支持無限深度

### 8.4 攻擊路徑呈現格式

```
Attack Path AP-001:
  Chain: EXP-001 (FTP Anonymous) → EXP-005 (Credential File) → EXP-003 (Admin Login)
  Description: 
    Step 1: FTP anonymous login confirmed → enumerate files
    Step 2: Credential file discovered in /backup/credentials.txt
    Step 3: Credentials used to login to web admin panel at /admin
  Combined Impact: critical (full admin access achieved)
  Status: confirmed / partial / theoretical
```

### 8.5 串連攻擊典型範例

| 鏈結模式 | 範例 |
|---------|------|
| 資訊洩漏 → 認證繞過 | FTP 匿名存取 → 取得密碼檔 → 登入後台 |
| LFI → 敏感檔讀取 → 認證 | LFI 讀取 /etc/passwd → 識別使用者 → SSH 暴力破解 |
| SQLi → 資料庫 → 權限提升 | SQL 注入 → 取得 DB 帳密 → 連接資料庫伺服器 |
| 版本弱點 → RCE → 橫向移動 | Apache CVE → 遠端程式執行 → 內網探索 |
| 弱密碼 + 管理介面 | 暴力破解 → admin 登入 → 應用層控制 |

---

## 九、執行步驟

### Step 0：上下文載入與 VTT 初始化（Context Loading & VTT Initialization）

**目的**：讀取所有前序階段資料，建立初始 VTT。

**動作**：
1. 讀取 `ctem-state.md` → 確認 Prioritization 狀態 `completed`
2. 設定 Validation 狀態為 `in_progress`
3. 讀取 Scoping Summary → 擷取 Target Host、Business Criticality、Attack Surface Boundary
4. 讀取 Prioritization Summary → 擷取 Prioritized Exposure List、Host-Level Compensating Controls
5. 讀取 Discovery Summary → 擷取 Open Services Detected
6. 讀取 `reports/assets/<id>.md` → 擷取 Exposure Registry
7. **讀取 [validation-modules.md](./references/validation-modules.md)** — 載入三模組的完整定義與 prompt
8. VRM 使用 `VRM_INIT_VTT` prompt + Prioritized Exposure List → 建立初始 VTT
9. VTT 中包含：
   - 每個 open 暴露的驗證任務（按 Adjusted Severity 降序）
   - 每個 Affected Service 的探索性任務
   - Attack Path Analysis 區塊（初始為空）
10. 呈現初始 VTT 給使用者：

> *「Validation Testing Tree (VTT) 已建立。本輪將驗證以下暴露並對相關服務進行探索：」*
>
> \<完整 VTT\>
>
> *「首個驗證任務：」*
>
> \<VGM 生成的驗證步驟\>
>
> *「請執行上述步驟並回傳結果。」*

### Step 1：驗證規劃（Validation Planning）

**目的**：VRM + VGM 協作，為最高優先暴露生成具體驗證計畫。

**動作**：
1. VRM 從 VTT 選擇最高優先的 to-do 任務
2. VRM 輸出三句式任務描述（任務描述、建議指令、預期結果）
3. VGM 接收三句式描述 → 展開為詳細步驟指引
4. 呈現給使用者

**呈現格式**：

> **驗證任務 1.2: Execute path traversal PoC**
>
> **工具**: curl
> **Recommended steps:**
> 1. 執行 PoC 指令：`curl -s --path-as-is http://10.10.10.5/icons/.%2e/%2e%2e/%2e%2e/etc/passwd`
> 2. 檢查回傳內容是否包含 `/root:x:0:0:` 等 passwd 格式
> 3. 若成功 → confirmed（可讀取系統檔案）
> 4. 若回傳 403/404 → 可能已修補或 WAF 阻擋，嘗試變體...
>
> **結果判讀**：
> - `confirmed`：回傳內容包含有效的 /etc/passwd 條目
> - `false-positive`：回傳 403 Forbidden 且無繞過方式
> - `inconclusive`：連線逾時或回傳不明確內容

### Step 2：驗證執行循環（Validation Execution Loop）

**目的**：三模組循環協作，直到所有暴露皆有結論或使用者停止。

**互動模式（Q2 決策：C — 混合模式）**：
- VRM 標記哪些任務可並行（無相依關係）、哪些有依賴
- 可並行任務：一次列出多個，使用者自由選順序
- 有依賴的任務：依序呈現

**循環流程**：

```
REPEAT:
  1. 使用者提供驗證結果（工具輸出 / 網頁內容 / 文字描述）
  2. VPM 解析結果 → 摘要 + 判定（confirmed / false-positive / inconclusive）
     └─ 若發現新暴露 → 標記 NEW FINDING
  3. VRM 接收 VPM 摘要 → 更新 VTT：
     a. 標記已完成任務的狀態
     b. 若有 NEW FINDING：
        - 一般 → 就地註冊（見七、新暴露處理機制），加入 VTT
        - 重大 → 通知使用者，建議 backtrack
     c. 若有新 confirmed 暴露 → 檢查攻擊路徑機會
     d. 選擇下一任務（或列出可並行任務）
  4. VGM 為下一任務生成具體指令
  5. 呈現更新後的 VTT + 下一步驟給使用者
UNTIL:
  - 所有暴露都有結論（confirmed / false-positive / inconclusive）
  - 或使用者選擇 early-exit
```

**使用者互動選項**（參考 PentestGPT 的 `main_task_entry`）：

| 選項 | 功能 |
|------|------|
| `next` | 提供驗證結果（工具輸出 / 網頁 / 描述） |
| `todo` | 查看當前 VTT 狀態和待辦任務 |
| `discuss` | 自由討論，VRM 分析後可能調整 VTT |
| `attack-path` | 觸發攻擊路徑分析（不等到所有暴露驗證完） |
| `skip` | 跳過當前暴露（標記 inconclusive）進入下一個 |
| `done` | 結束驗證循環，進入 Step 3 |

### Step 3：攻擊路徑彙整（Attack Path Consolidation）

**目的**：VRM 進行最終的攻擊路徑分析，彙整所有串連關係。

**動作**：
1. VRM 使用 `VRM_ATTACK_PATH_ANALYSIS` prompt
2. 檢視所有 `confirmed` 暴露（含就地註冊的新暴露）
3. 分析暴露之間的串連可能性
4. 為每條攻擊路徑生成 AP-ID
5. 對尚未驗證的鏈結節點，詢問使用者是否繼續深入
6. 呈現最終攻擊路徑摘要：

> *「攻擊路徑分析結果：」*
>
> | # | Path ID | Chain (Exposure IDs) | Description | Combined Impact | Status |
> |---|---------|---------------------|-------------|-----------------|--------|
> | 1 | AP-001 | EXP-001 → EXP-005 → EXP-003 | FTP → cred file → admin login | critical | confirmed |
> | 2 | AP-002 | EXP-002 → EXP-006 | SQLi → DB access | high | partial |
>
> *「是否要對部分確認的路徑（AP-002）繼續深入驗證？」*

使用者選擇後，可回到 Step 2 的循環繼續驗證，或確認結束。

### Step 4：寫入與產出（Write & Output）

**目的**：將驗證結果寫入所有目標位置。

#### 4a — 更新 Asset Profile (`reports/assets/<id>.md`)

**Exposure Registry 表**：
- `confirmed` 暴露 → 更新 `Current Status` 為 `confirmed`
- `false-positive` 暴露 → 更新 `Current Status` 為 `false-positive`
- `inconclusive` 暴露 → 維持原狀態，在 Notes 加註「Validation inconclusive — needs further investigation」
- 就地註冊的新暴露 → 新增行，含完整欄位（Exposure ID、Title、First Seen、Last Seen、Severity History、Adjusted Severity、Current Status）

**Current Risk Summary 表（Q6 決策：B — 立即更新）**：
- false-positive 被移除後立即反映在風險計算中
- `Overall Risk Level`：重新計算所有非 false-positive 的 open 暴露中最高 Adjusted Severity
- `Open Exposures`：重新計數（排除 false-positive）
- `Trend`：與 Prioritization 後的 Overall Risk Level 比較，判斷驗證後風險是否改變

#### 4b — 寫入 Validation Summary (`ctem-state.md`)

格式見「十一、資訊傳遞機制」。

#### 4c — 更新 Phase Status (`ctem-state.md`)

- 設定 Validation 為 `completed`
- 填入 Key Findings Summary 和 Last Updated
- 新增 Transition Log 項目

---

## 十、互動風格

### 10.1 主要互動方式

採 **PentestGPT 循環模式 + CTEM 結構化輸出混合**：

1. **初始化階段**（Step 0-1）：自動讀取前序資料，建立 VTT，生成第一個驗證指令——類似 PentestGPT 的 `_feed_init_prompts()`
2. **執行循環**（Step 2）：使用者提供結果 → VPM 解析 → VRM 更新 VTT → VGM 生成指令——類似 PentestGPT 的 main loop
3. **收尾階段**（Step 3-4）：攻擊路徑彙整 + 結構化輸出——CTEM 特有

### 10.2 結果輸入方式

沿用 Discovery 的混合輸入模式：

| 輸入來源 | 處理方式 |
|---------|---------|
| 工具輸出（工具面文字貼上） | VPM 解析工具輸出，識別關鍵發現 |
| 網頁內容 | VPM 摘要網頁關鍵元素 |
| 使用者描述 | VPM 精簡轉述，不做額外假設 |
| 檔案路徑 | 讀取檔案後由 VPM 處理 |

偵測邏輯：若使用者訊息包含檔案路徑格式 → 嘗試 read_file；否則視為直接貼上的文字。

### 10.3 VTT 更新呈現

每次 VRM 更新 VTT 後，向使用者呈現：
1. 剛完成的任務結果摘要（一行）
2. VTT 中的關鍵狀態變更（哪些任務完成、新增了什麼）
3. 下一個建議任務的詳細步驟
4. 若有新發現暴露或攻擊路徑 → 特別標註

**不必每次都列出完整 VTT**——只在使用者選擇 `todo` 時或重大 VTT 變更時列出完整樹。

---

## 十一、資訊傳遞機制

### 11.1 持久化位置

| 位置 | 內容 | 生命週期 |
|------|------|---------|
| `ctem-state.md` → `### Validation Summary` | 本輪 Validation 結果摘要 | 單輪 session |
| `reports/assets/<id>.md` → Exposure Registry | 暴露狀態更新（confirmed / false-positive） | 跨輪次長期維護 |
| `reports/assets/<id>.md` → Current Risk Summary | 驗證後風險狀態（即時更新） | 跨輪次（每輪覆寫最新值） |

### 11.2 Validation Summary 格式（寫入 ctem-state.md）

```markdown
### Validation Summary

**Validation Date**: <ISO 8601 日期>
**Method**: Three-Module Validation (VRM / VGM / VPM)
**VTT Model**: Validation Testing Tree — adapted from PentestGPT PTT (Deng et al., 2024)

#### Validation Results

| # | Exposure ID | Title | Adjusted Severity | Validation Status | Evidence Summary | Attack Path |
|---|-------------|-------|-------------------|-------------------|------------------|-------------|
| 1 | EXP-001 | Apache Path Traversal | critical | confirmed | File read via curl PoC: /etc/passwd retrieved | AP-001 |
| 2 | EXP-002 | SSH Weak Ciphers | low | false-positive | Weak ciphers listed but connection rejected | — |
| 3 | EXP-003 | FTP Anonymous Login | high | confirmed | Anonymous login successful, files enumerated | AP-001 |

#### Newly Discovered During Validation

<!-- 驗證過程中就地註冊的新暴露 -->

| # | Exposure ID | Title | Type | Raw Severity | Adjusted Severity | Source | Validation Status |
|---|-------------|-------|------|-------------|-------------------|--------|-------------------|
| 1 | EXP-005 | Credential File on FTP | information-disclosure | high | critical | validation-exploratory | confirmed |
| 2 | EXP-006 | SQL Injection on /search | vulnerability | high | critical | validation-exploratory | confirmed |

#### Attack Paths Identified

| # | Path ID | Chain (Exposure IDs) | Description | Combined Impact | Status |
|---|---------|---------------------|-------------|-----------------|--------|
| 1 | AP-001 | EXP-003 → EXP-005 → EXP-004 | FTP anon → cred file → admin login | critical | confirmed |

#### Validation Testing Tree (Final State)

<!-- VTT 最終狀態 — 完整樹結構 -->

<完整 VTT 最終狀態>

#### Validation Statistics

| Metric | Value |
|--------|-------|
| Total Exposures Validated | N (original) + M (newly discovered) |
| Confirmed | N |
| False Positive | N |
| Inconclusive | N |
| Newly Discovered | M |
| Attack Paths Identified | N |
| Updated Overall Risk Level | <recalculated after removing false-positives> |
| Risk Level Change | <before validation> → <after validation> |
| Recommendation | <brief recommendation for Phase 5 — Mobilization> |
```

### 11.3 下游階段如何取得資訊

- **Mobilization** 啟動時 → 讀取 `ctem-state.md` 的 `### Validation Summary` → 取得確認的暴露清單、攻擊路徑、新發現暴露
- 需要更多細節 → 讀取 `reports/assets/<id>.md` 的 Exposure Registry（含更新後的 Current Status）和 Current Risk Summary

### 11.4 Session Report 資料

Validation 產出的以下資料由報告產生階段編入 Session Report：
- `Exposure Summary` → `Status` 欄位：更新為 `confirmed` 或 `false-positive`
- `Phase Results Summary` → Validation 行：填入驗證結果統計

Validation **不直接寫入 Session Report 檔案** — Session Report 在五階段全部完成後由報告產生流程統一建立。

---

## 十二、完成條件與 Checklist

Validation skill 在所有步驟做完後，產出以下 checklist：

```markdown
## Validation Completion Checklist

- [ ] Prioritization Summary 與上游資料已讀取
- [ ] VTT 已建立並呈現給使用者
- [ ] 所有暴露已驗證（或標記 inconclusive / 使用者選擇 early-exit）
- [ ] 新發現暴露已就地註冊並評分（若有）
- [ ] 攻擊路徑分析已完成
- [ ] Asset Profile 的 Exposure Registry 和 Current Risk Summary 已更新
- [ ] Validation Summary 已寫入 ctem-state.md
```

**職責劃分**：
- Validation skill：執行驗證、產出 checklist、寫入驗證結果與更新 Risk Summary
- ctem-flow：驗證 checklist、更新 Phase Status、執行階段轉換、Backtrack Check

---

## 十三、參考檔案

獨立存放於 `references/` 目錄，SKILL.md 透過相對連結引用。

### 13.1 references/validation-modules.md

三模組核心參考，包含：
- **VRM / VGM / VPM 完整 prompt 定義**：系統提示詞與附加提示詞
- **三模組協作流程圖**：輸入/輸出格式、協作時序
- **PentestGPT 對映說明**：每個模組與 PentestGPT 原始模組的關聯與改造點
- **Prompt 來源標注**：標明每段 prompt 改寫自 `prompt_class_v2.py` 的哪個欄位

> **SKILL.md 必須在 Step 0 載入此檔案。**

### 13.2 references/vtt-protocol.md

VTT 樹的格式規範與管理協定，包含：
- **VTT 格式定義**：樹狀結構、層級規則、命名慣例
- **任務狀態定義**：to-do / in-progress / completed / not-applicable
- **VTT 初始化規則**：如何從 Prioritized Exposure List 生成初始 VTT
- **探索性任務生成規則**：按服務類型生成的探索任務模板
- **VTT 更新規則**：任務狀態變更、新任務插入、VTT 剪枝
- **攻擊路徑記錄格式**：AP-ID 編號、鏈結描述、影響評估

### 13.3 references/validation-tools.md

驗證工具指令參考，包含：
- **curl / wget**：Web PoC 驗證指令模板（路徑遍歷、LFI、header 注入等）
- **nmap NSE**：針對性腳本（特定 CVE 驗證、服務深度探測）
- **Nuclei（PoC mode）**：單模板精確驗證用法
- **gobuster / dirb / feroxbuster**：目錄與檔案列舉
- **hydra / medusa**：認證測試（brute-force / default credential testing）
- **sqlmap**：SQL 注入驗證
- **手動驗證**：Telnet/Netcat 互動、手動 HTTP 請求構造
- **結果判讀指引**：每類工具的 confirmed / false-positive / inconclusive 判定標準

---

## 十四、與 ctem-flow 的互動點

Validation skill 完成後，ctem-flow 的 Backtrack Check 會檢查：

| 檢查項 | 來源 | 處理 |
|--------|------|------|
| 新暴露（重大）在驗證中發現 | Validation Summary → Newly Discovered During Validation | 若有重大發現且使用者選擇 backtrack → 回 Discovery |
| 大量 false-positive | Validation Summary → Validation Statistics | 若 false-positive 比例極高（>50%），可能暗示 Discovery 掃描品質問題，建議回溯 |
| 攻擊路徑揭示新攻擊面 | Validation Summary → Attack Paths Identified | 若攻擊路徑涉及 Scoping 邊界外的資產，建議回溯 Scoping |
| 無新發現且結果明確 | Validation Statistics | 建議進入 Phase 5 — Mobilization |

Validation 在 Summary 中提供完整的驗證結果與攻擊路徑，讓 ctem-flow 有足夠資訊判斷是否需要 backtrack。

---

## 十五、檔案影響範圍

建置 Validation skill 時需要建立/修改的檔案：

| 檔案 | 動作 | 說明 |
|------|------|------|
| `.github/skills/ctem-4-validation/SKILL.md` | 重寫 | 主要 prompt 邏輯 |
| `.github/skills/ctem-4-validation/SKILL.zh-TW.md` | 新增 | 繁體中文版（輔助閱讀） |
| `.github/skills/ctem-4-validation/references/validation-modules.md` | 新增 | 三模組定義（含完整 prompt） |
| `.github/skills/ctem-4-validation/references/validation-modules.zh-TW.md` | 新增 | 繁體中文版 |
| `.github/skills/ctem-4-validation/references/vtt-protocol.md` | 新增 | VTT 格式規範與管理協定 |
| `.github/skills/ctem-4-validation/references/vtt-protocol.zh-TW.md` | 新增 | 繁體中文版 |
| `.github/skills/ctem-4-validation/references/validation-tools.md` | 新增 | 驗證工具指令參考 |
| `.github/skills/ctem-4-validation/references/validation-tools.zh-TW.md` | 新增 | 繁體中文版 |

> 注意：`ctem-state.md` 和 `reports/` 模板不需要修改 — 已有預留的 `### Validation Summary` 位置和對應欄位。

---

## 十六、參考來源

| 來源 | 用途 |
|------|------|
| Gartner, 2022 — *Implement a CTEM Program* (G00766755) | CTEM 框架定義、Validation 階段職責 |
| Deng et al., 2024 — *PentestGPT: An LLM-empowered Automatic Penetration Testing Tool* | 三模組架構（Reasoning/Generation/Parsing）、PTT 設計、prompt 工程 |
| PentestGPT `prompt_class_v2.py` | VRM / VGM / VPM prompt 改寫來源 |
| PentestGPT `PentestGPT_design.md` | 三模組協作流程設計參考 |
| FIRST — *CVSS v3.1 Specification Document* (2019) | Raw Severity 分級依據 |
| NIST SP 800-30 Rev. 1 — *Guide for Conducting Risk Assessments* (2012) | 風險矩陣設計依據（就地評分） |
| OWASP — *Testing Guide v4* | 驗證方法論與工具參考 |
| OWASP — *Web Security Testing Guide* | Web 弱點驗證流程 |
| CISA — *Known Exploited Vulnerabilities Catalog* | 可利用性情報參考 |

---

## 十七、設計決策記錄

以下記錄本設計稿的所有關鍵決策及理由，供後續追溯：

| # | 決策 | 結果 | 理由 |
|---|------|------|------|
| Q1 | VTT 範圍 | A — 所有 open 暴露 | 保持與前階段一致性、論文完整性 |
| Q2 | 互動模式 | C — 混合（並行+依賴） | PentestGPT PTT 天然有相依性，並行提升效率 |
| Q3 | references 拆分 | A — 三個獨立檔案 | 與 Discovery 結構一致 |
| Q4 | 驗證狀態 | A — 三種（confirmed / false-positive / inconclusive） | 簡潔清晰，partially-confirmed 用 Rationale 表達 |
| Q5 | 三模組呈現方式 | B — references 引用 | SKILL.md 保持可讀性，模組細節在 references |
| Q6 | false-positive 風險更新 | B — 立即更新 | 風險摘要應即時反映驗證結果 |
| Q7 | 新暴露處理 | C — 分級（一般就地 / 重大回溯） | 兼顧流程完整性和實用性 |
| Q8 | 攻擊鏈深度 | C — AI 建議 + 使用者決定 | 每層使用者自然確認，深度依實際情境 |
| Q9 | 就地評分 | A — 完整 Risk Matrix | 資料已在手，無需簡化 |
| Q10 | VTT 含探索性任務 | B — 探索+驗證 | 彌補掃描盲區、PentestGPT 精髓、強力論點 |
| — | MCP 整合 | 不納入主要實作 | 預留介面點，作為論文 Future Work |
| — | VTT 持久化 | 僅最終狀態寫入 Summary | 過程狀態在對話上下文中，減少檔案 I/O |
| — | PentestGPT prompt 來源 | v2（prompt_class_v2.py） | v2 更精煉，AI 指令更結構化 |
