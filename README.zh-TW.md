# CTEM Kit

一個純提示工程（Prompt Engineering）的自動化工具組，利用 GitHub Copilot 的 AI 代理技能，自動化 **Gartner CTEM（持續威脅暴露管理）** 五階段工作流程。

**零程式碼。純 Markdown。完全由 AI 驅動。**

## 什麼是 CTEM？

CTEM（Continuous Threat Exposure Management，持續威脅暴露管理）是 Gartner 提出的五階段框架，用於主動管理組織的威脅暴露：

1. **範圍界定（Scoping）** — 定義範圍：資產、業務關鍵性、攻擊面邊界
2. **發現（Discovery）** — 找出暴露：漏洞、錯誤配置、攻擊面缺口
3. **優先排序（Prioritization）** — 依風險排序：可利用性 × 業務影響 × 情境
4. **驗證（Validation）** — 驗證暴露是否真實且可被利用，過濾誤報
5. **動員（Mobilization）** — 產生修復計畫、指派行動、追蹤解決進度

## 快速開始

### 前置需求

- [VS Code](https://code.visualstudio.com/) 並安裝 [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) 擴充套件
- 啟用 Copilot Chat 的 Agent 模式

### 使用方式

#### 方式 A：完整自動化流程（推薦）

1. 在 VS Code 中開啟本專案
2. 開啟 Copilot Chat 並選擇 **`@ctem-coordinator`** 代理
3. 告訴它你的目標：
   ```
   @ctem-coordinator 對目標 192.168.1.0/24 啟動新的 CTEM 工作階段
   ```
4. 協調器將引導你完成所有五個階段，自動處理階段轉換和回溯

#### 方式 B：執行個別階段

你可以將任何階段作為斜線指令獨立執行：

| 指令 | 階段 | 功能 |
|------|------|------|
| `/ctem-scoping` | 1. 範圍界定 | 定義目標範圍與資產清冊 |
| `/ctem-discovery` | 2. 發現 | 解析工具輸出、識別暴露 |
| `/ctem-prioritization` | 3. 優先排序 | 依風險評分和排序暴露 |
| `/ctem-validation` | 4. 驗證 | 驗證可利用性、過濾誤報 |
| `/ctem-mobilization` | 5. 動員 | 產生修復計畫和行動項目 |

> **注意**：單獨執行階段時，你需要自行管理階段順序並更新 `ctem-state.md`。使用方式 A 時，協調器會自動處理。

#### 方式 C：恢復工作階段

如果你中途停止了工作階段：

1. 開啟 Copilot Chat → 選擇 **`@ctem-coordinator`**
2. 輸入：
   ```
   @ctem-coordinator 恢復目前的工作階段
   ```
3. 協調器會讀取 `ctem-state.md` 並從上次中斷的地方繼續

## 專案結構

```
ctem-kit/
├── .github/
│   ├── copilot-instructions.md          # 所有 CTEM 互動的全域規則
│   ├── instructions/
│   │   └── ctem-state-protocol.instructions.md  # 狀態檔案讀寫規則
│   ├── agents/
│   │   └── ctem-coordinator.agent.md    # 工作流程協調器（管理階段）
│   └── skills/
│       ├── ctem-scoping/SKILL.md        # 階段 1：範圍界定
│       ├── ctem-discovery/SKILL.md      # 階段 2：發現
│       ├── ctem-prioritization/SKILL.md # 階段 3：優先排序
│       ├── ctem-validation/SKILL.md     # 階段 4：驗證
│       └── ctem-mobilization/SKILL.md   # 階段 5：動員
├── ctem-state.md                        # 工作階段狀態追蹤（AI 管理）
└── README.md                            # 英文說明文件
```

### 檔案角色說明

| 檔案 | 角色 | 使用者 |
|------|------|--------|
| `copilot-instructions.md` | 定義 CTEM 情境與全域規則。每次對話自動載入。 | AI（自動載入） |
| `ctem-state-protocol.instructions.md` | `ctem-state.md` 的讀寫規則。存取狀態檔案時自動載入。 | AI（相關時自動載入） |
| `ctem-coordinator.agent.md` | 工作流程管理器。決定階段順序、處理回溯、追蹤進度。不執行分析。 | 使用者透過 `@ctem-coordinator` 呼叫 |
| `ctem-scoping/SKILL.md` | 階段 1 邏輯。定義範圍、盤點資產、繪製攻擊面。 | 使用者透過 `/ctem-scoping` 呼叫或由協調器委派 |
| `ctem-discovery/SKILL.md` | 階段 2 邏輯。解析掃描輸出、識別暴露。 | 使用者透過 `/ctem-discovery` 呼叫或由協調器委派 |
| `ctem-prioritization/SKILL.md` | 階段 3 邏輯。評分和排序暴露。 | 使用者透過 `/ctem-prioritization` 呼叫或由協調器委派 |
| `ctem-validation/SKILL.md` | 階段 4 邏輯。透過三模組方法驗證可利用性（推理 / 生成 / 解析）。 | 使用者透過 `/ctem-validation` 呼叫或由協調器委派 |
| `ctem-mobilization/SKILL.md` | 階段 5 邏輯。產生修復計畫並追蹤修復進度。 | 使用者透過 `/ctem-mobilization` 呼叫或由協調器委派 |
| `ctem-state.md` | 即時工作階段狀態。追蹤已完成的階段、發現摘要和回溯歷史。 | AI 讀寫；使用者可檢視 |

## 回溯機制

不同於線性工作流程，CTEM 需要**非線性階段轉換**。例如，驗證暴露時可能發現新的攻擊面，需要重新進入發現階段。

### 運作方式

- 每個階段結束後，協調器會執行**回溯檢查**
- 將新發現與 `ctem-state.md` 中前一階段的輸出進行比對
- 如需回溯，協調器會建議回到哪個階段及原因
- 使用者也可隨時手動要求回溯
- 每個工作階段最多 **3 次回溯**，避免無限迴圈

### 回溯流程

```
範圍界定 → 發現 → 優先排序 → 驗證 ──→ 動員
                                  │
          ┌───────────────────────┘
          │（驗證時發現新暴露）
          ▼
        發現 → 優先排序 → 驗證 → 動員
```

## 擴展工具組

### 為階段技能新增細節

編輯 `.github/skills/ctem-<phase>/` 下對應的 `SKILL.md` 檔案。標有 `<!-- TODO -->` 的技能是等待完整提示實作的預留位置。

### 新增參考資料

在任何技能目錄中建立 `references/` 資料夾：

```
.github/skills/ctem-validation/
├── SKILL.md
└── references/
    ├── attack-path-reasoning.md
    ├── exploit-validation.md
    └── result-analysis.md
```

在 `SKILL.md` 中用相對連結引用：`[攻擊路徑推理](./references/attack-path-reasoning.md)`

### 新增素材 / 範本

在任何技能目錄中建立 `assets/` 資料夾，放置可重複使用的範本：

```
.github/skills/ctem-mobilization/
├── SKILL.md
└── assets/
    └── remediation-report-template.md
```
