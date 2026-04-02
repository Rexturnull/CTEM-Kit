# CTEM Phase 1 — Scoping Skill 完整設計稿

> 本文件統整所有討論決策，作為 `ctem-1-scoping/SKILL.md` 的建置藍圖。

---

## 一、設計前提

- 流程無限貼近 Gartner CTEM 框架（Gartner, 2022, ID: G00766755）
- 本版為**簡單版**：單一機器 per session（多機器/多線程列入未來 TODO）
- Skill 保持獨立，流程控制回歸 `ctem-flow`
- 所有評估框架必須有依據並附來源（論文需求）

---

## 二、輸入模式

Scoping skill 啟動時，根據 `reports/assets/` 是否已存在對應資產檔案，自動分為兩條路徑：

### 路徑 A：首輪（無既有資產）

- 走完整四步驟流程
- 由使用者提供目標資訊，skill 以問答引導補齊

### 路徑 B：第二輪+（已有資產檔案）

- 讀取既有 `reports/assets/<id>.md`，呈現給使用者
- 詢問：資訊是否仍正確？有無變更？
- **無變更 → 快速通過**：直接將 asset 資訊帶入 `ctem-state.md` 的 Scoping Summary
- **有變更 → 更新模式**：進入對應步驟修改
- 使用者也可主動要求「重新開始」，此時走路徑 A

---

## 三、執行步驟

### Step 1：目標定義（Target Definition）

**目的**：確認要評估的單一目標主機。

**收集資訊**：
- 目標 IP 位址或 hostname
- 作業系統 / 平台
- 主要角色 / 服務

**互動方式**：混合模式 — 先解析使用者在啟動指令中已提供的資訊，缺什麼再問。

**工具指令**（引導使用者手動執行）：
| 用途 | 指令範例 |
|------|---------|
| 確認主機存活 | `nmap -sn <target>` |
| 作業系統偵測 | `nmap -O <target>` |
| 開放服務列舉 | `nmap -sV -p- <target>` |
| 確認 hostname | `nslookup <ip>` / `dig -x <ip>` |

> 工具指令拆出至 `references/tool-commands.md` 獨立維護，SKILL.md 透過連結引用。

### Step 2：資產發現與登錄（Asset Discovery & Registration）

**目的**：為目標主機建立或更新資產檔案。

**動作**：
- 首輪：根據 `reports/assets/TEMPLATE.md` 建立新的 `reports/assets/<hostname-or-ip>.md`，填入 Step 1 收集到的資訊
- 第二輪：更新既有資產檔案（若有變更）

**填入欄位**（對應 Asset Profile Template）：
- Asset ID（自動生成，例如 ASSET-001）
- Hostname
- IP Address
- OS / Platform
- Role / Service
- First Seen / Last Assessed

### Step 3：業務關鍵性評估（Business Criticality Assessment）

**目的**：評估目標主機對業務的重要性。

**評估框架**：NIST SP 800-30 Rev. 1 — *Guide for Conducting Risk Assessments*（NIST, 2012）
- 來源：https://csrc.nist.gov/pubs/sp/800-30/r1/final
- 輔助框架：FIPS 199 — CIA 三維度分級（https://csrc.nist.gov/pubs/fips/199/final）

**引導問答**（基於 FIPS 199 CIA 維度）：

> **問題 1 — 機密性（Confidentiality）**
> 若這台機器被入侵，機密資料外洩的影響程度？
> - 高：包含客戶個資、財務資料、營業秘密
> - 中：包含內部文件，但無敏感個資
> - 低：僅有公開或無敏感資訊

> **問題 2 — 完整性（Integrity）**
> 若這台機器的資料被竄改，業務影響程度？
> - 高：直接影響客戶服務或財務正確性
> - 中：影響內部決策但可事後修正
> - 低：影響極小，容易偵測與復原

> **問題 3 — 可用性（Availability）**
> 若這台機器離線 24 小時，業務影響程度？
> - 高：核心業務完全中斷
> - 中：部分業務受影響但有替代方案
> - 低：幾乎不影響日常運作

**分級邏輯**：三維度取最高級別 = 最終 Business Criticality

| 最高維度結果 | Business Criticality |
|-------------|---------------------|
| 高 | Critical 或 High（依 context 判斷） |
| 中 | Medium |
| 低 | Low |

> 若三維度中有任一為「高」且該資產為面向客戶的生產系統 → Critical；其餘「高」→ High。

**分級定義**（對齊 NIST SP 800-30 並簡化為 4 級）：

| 等級 | 定義 | 典型範例 |
|------|------|---------|
| **Critical** | 系統失效將直接導致業務中斷或重大資料外洩 | 面向客戶的生產系統、AD DC、核心 DB |
| **High** | 系統失效將嚴重影響業務運營 | 內部重要服務、備援系統、CI/CD |
| **Medium** | 系統失效會造成不便但不影響核心業務 | 開發環境、監控系統、內部工具 |
| **Low** | 系統失效影響極小 | 測試機、沙盒、已退役但仍在線的服務 |

**產出**：
- 將 Business Criticality 寫入 asset 檔案
- 記錄三維度的個別評估結果（供論文引用追溯）

### Step 4：攻擊面邊界確認（Attack Surface Boundary Confirmation）

**目的**：明確界定本次評估的邊界。

**收集資訊**：
- In-Scope Services：哪些服務/端口列入評估
- Out-of-Scope：哪些部分明確排除（例如後端 DB 在不同網段）
- 攻擊面邊界說明（host-level only / 含相鄰網段 / etc.）

**產出**：
- 整理成結構化的 In-Scope / Out-of-Scope 清單
- 寫入 Scoping Summary

---

## 四、資訊傳遞機制

### 4.1 持久化位置

| 位置 | 內容 | 生命週期 |
|------|------|---------|
| `ctem-state.md` → `## Phase Summaries` 下的 `### Scoping Summary` | 本輪 Scoping 結果摘要 | 單輪 session |
| `reports/assets/<id>.md` | 資產完整檔案 | 跨輪次長期維護 |

### 4.2 Scoping Summary 格式（寫入 ctem-state.md）

```markdown
### Scoping Summary

| Field | Value |
|-------|-------|
| Target Host | 10.0.0.5 |
| Hostname | web-prod-01 |
| OS / Platform | Ubuntu 22.04 |
| Role / Service | Web Application Server |
| Business Criticality | high |
| Scoping Trigger | *(本版暫不實作，留空或填 manual)* |
| Attack Surface Boundary | host-level only |
| In-Scope Services | HTTP (80), HTTPS (443), SSH (22) |
| Out-of-Scope | Backend DB on separate segment |
| Notes | — |
```

### 4.3 下游階段如何取得資訊

- Discovery 啟動時 → 讀取 `ctem-state.md` 的 `Scoping Summary` → 知道要掃什麼目標、哪些服務
- 需要更多細節 → 讀取 `reports/assets/<id>.md`

### 4.4 每個階段都有 Summary

`ctem-state.md` 中每個階段完成時都在 `## Phase Summaries` 之下寫入對應子區塊：
- `### Scoping Summary`
- `### Discovery Summary`（Phase 2 完成後寫入）
- `### Prioritization Summary`（Phase 3 完成後寫入）
- `### Validation Summary`（Phase 4 完成後寫入）
- `### Mobilization Summary`（Phase 5 完成後寫入）

每個階段 skill 只需讀 `ctem-state.md` 就能取得前序階段的所有摘要，保持模組獨立。

---

## 五、完成條件與 Checklist

Scoping skill 在所有步驟做完後，產出以下 checklist：

```markdown
## Scoping Completion Checklist

- [ ] 目標主機已識別（IP / Hostname）
- [ ] 作業系統與平台已確認
- [ ] 角色 / 服務已定義
- [ ] 業務關鍵性已評估（含 CIA 三維度）
- [ ] 攻擊面邊界已界定（In-Scope / Out-of-Scope）
- [ ] Scoping Trigger 已記錄
- [ ] 資產檔案已建立 / 更新（reports/assets/）
- [ ] ctem-state.md Scoping Summary 已填寫

✅ 以上全部完成 → 回覆「Scoping 完成」讓 ctem-flow 進行下一步
```

**職責劃分**：
- Scoping skill：執行分析、產出 checklist、寫入 asset 檔案與 Scoping Summary
- ctem-flow：驗證 checklist、更新 Phase Status、執行階段轉換

---

## 六、互動風格

採**混合模式**：
1. 解析使用者在啟動指令中已提供的資訊（例如目標 IP、機器描述等）
2. 自動填入已知欄位
3. 僅針對缺失的必要資訊逐項提問
4. 不重複詢問已知資訊

---

## 七、參考工具指令

獨立存放於 `references/tool-commands.md`，SKILL.md 透過相對連結引用。
未來接 MCP 時，只需修改 reference 檔案或在 SKILL.md 加入 MCP tool 呼叫邏輯。

---

## 八、檔案影響範圍

建置 Scoping skill 時需要建立/修改的檔案：

| 檔案 | 動作 | 說明 |
|------|------|------|
| `.github/skills/ctem-1-scoping/SKILL.md` | 重寫 | 主要 prompt 邏輯 |
| `.github/skills/ctem-1-scoping/references/tool-commands.md` | 新增 | 工具指令參考（模組化） |
| `ctem-state.md` | 修改 | 加入 Scoping Summary 區塊預留位置 |
| `.github/instructions/ctem-state-protocol.instructions.md` | 修改 | 加入各階段 Summary 區塊的格式規範 |

---

## 九、參考來源

| 來源 | 用途 |
|------|------|
| Gartner, 2022 — *Implement a CTEM Program* (G00766755) | CTEM 框架定義、business-aligned scoping |
| NIST SP 800-30 Rev. 1 (2012) — *Guide for Conducting Risk Assessments* | 業務關鍵性分級依據 |
| FIPS 199 (2004) — *Standards for Security Categorization* | CIA 三維度評估法 |
