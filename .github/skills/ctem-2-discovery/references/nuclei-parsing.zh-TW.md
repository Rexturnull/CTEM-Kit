# Nuclei 輸出解析指南 — Discovery 階段（中文閱讀版）

> 此檔案為 `nuclei-parsing.md` 的繁體中文翻譯，僅供閱讀參考。

從 Nuclei 掃描輸出擷取弱點資訊的規則。
使用者提供 Nuclei 掃描結果時按需載入。

---

## 目錄

1. [純文字輸出](#純文字輸出)
2. [JSON Lines 輸出 (-jsonl)](#json-lines-輸出--jsonl)
3. [嚴重性映射](#嚴重性映射)
4. [擷取規則總結](#擷取規則總結)

---

## 純文字輸出

### 格式辨識

Nuclei 的預設終端機輸出格式如下：

```
[2026-04-03T10:15:30] [apache-path-traversal] [http] [critical] http://10.0.0.5:80/cgi-bin/.%2e/%2e%2e/etc/passwd
[2026-04-03T10:15:31] [cve-2021-41773] [http] [high] http://10.0.0.5:80/icons/.%2e/%2e%2e/etc/passwd
[2026-04-03T10:15:32] [ssh-weak-ciphers] [network] [medium] 10.0.0.5:22
[2026-04-03T10:15:33] [http-missing-security-headers:x-frame-options] [http] [info] http://10.0.0.5:80
[2026-04-03T10:15:34] [tech-detect:apache] [http] [info] http://10.0.0.5:80
[2026-04-03T10:15:35] [ssl-dns-names] [ssl] [info] https://10.0.0.5:443
```

### 行格式

每行結構如下：
```
[時間戳記] [模板ID] [協定] [嚴重性] 匹配的URL或主機
```

### 擷取規則

| 欄位 | 位置 | 擷取方式 |
|------|------|---------|
| **時間戳記** | 第 1 個方括號 | ISO 8601 時間戳記 |
| **模板 ID** | 第 2 個方括號 | Nuclei 模板識別碼（例如 `cve-2021-41773`、`ssh-weak-ciphers`） |
| **協定** | 第 3 個方括號 | 協定類別（例如 `http`、`network`、`ssl`、`dns`） |
| **嚴重性** | 第 4 個方括號 | 五種之一：`critical`、`high`、`medium`、`low`、`info` |
| **匹配位置** | 方括號之後 | 偵測到發現的 URL 或 host:port |

### 模板 ID 到 CVE 的映射

許多 Nuclei 模板在模板 ID 中編碼了 CVE 資訊：

| 模板 ID 模式 | CVE | 處理方式 |
|-------------|-----|---------|
| `cve-YYYY-NNNNN` | 直接擷取：`CVE-YYYY-NNNNN` | 記錄 CVE |
| `CVE-YYYY-NNNNN` | 同上 — 大寫變體 | 記錄 CVE |
| 其他（例如 `ssh-weak-ciphers`） | 無 CVE | CVE 記錄為「—」 |

### 模板 ID 到暴露類型的映射

| 模板 ID 模式 | 暴露類型 |
|-------------|---------|
| `cve-*` 或 `CVE-*` | `vulnerability` |
| `*-weak-*`、`*-misconfig*`、`*-default-*` | `misconfiguration` |
| `*-detect*`、`tech-detect*`、`*-disclosure*` | `information-disclosure` |
| `*-version*`、`outdated-*` | `outdated-software` |
| `http-missing-security-headers*` | `misconfiguration` |
| `ssl-*`（弱加密、過期） | `misconfiguration` |
| `exposed-*`、`*-panel*` | `information-disclosure` |

當模板 ID 無法明確匹配模式時，從嚴重性和協定推斷：
- `critical` 或 `high` 嚴重性 → 可能為 `vulnerability`
- `info` 嚴重性 → 可能為 `information-disclosure`
- 模糊的情況 → 呈現給使用者分類

### 從匹配 URL 擷取受影響服務

| 匹配位置格式 | 端口 | 服務 |
|-------------|------|------|
| `http://host:80/...` | 80 | HTTP |
| `https://host:443/...` | 443 | HTTPS |
| `http://host:8080/...` | 8080 | HTTP |
| `host:22` | 22 | SSH |
| `host:3306` | 3306 | MySQL |
| `http://host/...`（無端口） | 80（預設） | HTTP |
| `https://host/...`（無端口） | 443（預設） | HTTPS |

### 子模板發現

部分模板以模板 ID 中的 `:` 分隔產生多個發現：

```
[http-missing-security-headers:x-frame-options] [http] [info] http://10.0.0.5
[http-missing-security-headers:x-content-type-options] [http] [info] http://10.0.0.5
```

預設將每個子發現記錄為**獨立暴露** — 它們代表不同的錯誤配置。但在 Step 3a（去重）時，使用者可選擇將同類別子發現合併為單一暴露（見 SKILL.md § 3a）。使用完整模板 ID（含子元件）作為標題的一部分：
- 標題：「Missing Security Header: X-Frame-Options」
- 標題：「Missing Security Header: X-Content-Type-Options」

---

## JSON Lines 輸出 (-jsonl)

### 格式辨識

使用者提供 JSONL 輸出時（來自 `nuclei -jsonl`），每行為一個 JSON 物件：

```json
{
  "template-id": "cve-2021-41773",
  "template-path": "/path/to/templates/cves/2021/CVE-2021-41773.yaml",
  "info": {
    "name": "Apache HTTP Server 2.4.49 - Path Traversal",
    "author": ["pdteam"],
    "tags": ["cve", "cve2021", "lfi", "apache", "misconfig"],
    "description": "A flaw was found in a change made to path normalization in Apache HTTP Server 2.4.49...",
    "severity": "high",
    "classification": {
      "cvss-metrics": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N",
      "cvss-score": 7.5,
      "cve-id": ["CVE-2021-41773"],
      "cwe-id": ["CWE-22"]
    },
    "reference": [
      "https://nvd.nist.gov/vuln/detail/CVE-2021-41773",
      "https://httpd.apache.org/security/vulnerabilities_24.html"
    ]
  },
  "type": "http",
  "host": "http://10.0.0.5:80",
  "matched-at": "http://10.0.0.5:80/cgi-bin/.%2e/%2e%2e/etc/passwd",
  "timestamp": "2026-04-03T10:15:31+08:00",
  "matcher-status": true
}
```

### JSON 欄位擷取

| Discovery 欄位 | JSON 路徑 | 說明 |
|---------------|-----------|------|
| **模板 ID** | `template-id` | 用於去重和類型推斷 |
| **標題** | `info.name` | 直接使用 — Nuclei 模板名稱具描述性 |
| **Raw Severity** | `info.severity` | 直接映射到我們的嚴重性等級 |
| **CVSS 分數** | `info.classification.cvss-score` | 若有，用於嚴重性映射（覆蓋標籤） |
| **CVE** | `info.classification.cve-id[0]` | 陣列中第一個 CVE；若為 null/空則記錄「—」 |
| **CWE** | `info.classification.cwe-id[0]` | 選填 — 對 Prioritization 有用的上下文 |
| **描述** | `info.description` | 對暴露詳情有用 |
| **協定** | `type` | `http`、`network`、`ssl` 等 |
| **匹配位置** | `matched-at` | URL 或 host:port — 從中擷取端口/服務 |
| **主機** | `host` | 目標主機 — 與 Scoping 交叉比對 |
| **標籤** | `info.tags` | 幫助判定暴露類型 |
| **參考連結** | `info.reference` | 更多資訊的外部連結 |
| **匹配狀態** | `matcher-status` | 必須為 `true` — 若為 `false` 則跳過 |

### JSON 標籤到暴露類型的映射

使用 `info.tags` 陣列進行更準確的類型分類：

| 標籤 | 暴露類型 |
|------|---------|
| `cve`、`cve20XX` | `vulnerability` |
| `misconfig`、`misconfiguration` | `misconfiguration` |
| `exposure`、`disclosure` | `information-disclosure` |
| `tech`、`detect` | `information-disclosure` |
| `outdated` | `outdated-software` |
| `default-login`、`default-credential` | `misconfiguration` |
| `lfi`、`rfi`、`sqli`、`xss`、`rce`、`ssrf` | `vulnerability` |
| `panel`、`login` | `information-disclosure` |

優先順序：若標籤包含 `cve` → `vulnerability`（覆蓋其他標籤）。

---

## 嚴重性映射

### Nuclei 嚴重性到 Raw Severity

Nuclei 提供的嚴重性標籤直接映射到我們的等級：

| Nuclei 嚴重性 | Raw Severity |
|--------------|-------------|
| `critical` | `critical` |
| `high` | `high` |
| `medium` | `medium` |
| `low` | `low` |
| `info` | `info` |

### 有 CVSS 分數時（JSON 輸出）

若 `info.classification.cvss-score` 存在 — 當嚴重性標籤看起來不一致時特別有用：

| CVSS 分數 | Raw Severity |
|----------|-------------|
| 9.0 – 10.0 | `critical` |
| 7.0 – 8.9 | `high` |
| 4.0 – 6.9 | `medium` |
| 0.1 – 3.9 | `low` |
| 0.0 | `info` |

**衝突解決：** 若 CVSS 分數映射的嚴重性與標籤不同，**以 CVSS 分數為準**。在暴露記錄中註記差異。

---

## 擷取規則總結

### 處理流程

1. **判斷格式**：根據輸入結構判定為純文字或 JSONL。
2. **對每項發現**（文字中每行、JSONL 中每個 JSON 物件）：
   a. 擷取模板 ID 和嚴重性。
   b. 若 `matcher-status` 為 `false`（JSON）或行格式不符預期模式則跳過。
   c. 判定 CVE（從模板 ID 或 JSON `cve-id` 欄位）。
   d. 判定暴露類型（從模板 ID 模式、標籤或嚴重性）。
   e. 從匹配 URL/主機擷取受影響服務。
   f. 編撰為暴露記錄。
3. **去重**：相同 template-id + 相同 host:port = 同一發現（保留一筆，若 URL 不同則註記有多個匹配）。
4. **呈現**完整表格供使用者確認。

### Nuclei 發現 → Exposure 記錄映射

| Nuclei 欄位 | Exposure 欄位 |
|------------|-------------|
| `info.name` 或 template-id | Title |
| （見類型映射表） | Type |
| `info.severity` 或 CVSS 分數 | Raw Severity |
| `info.classification.cve-id` 或模板 ID 模式 | CVE |
| 從 `matched-at` 擷取 | Affected Service |
| "nuclei" | Source Tool |
