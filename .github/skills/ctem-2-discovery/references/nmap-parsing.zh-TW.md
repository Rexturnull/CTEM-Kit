# nmap 輸出解析指南 — Discovery 階段（中文閱讀版）

> 此檔案為 `nmap-parsing.md` 的繁體中文翻譯，僅供閱讀參考。

從 nmap 輸出擷取服務與弱點資訊的規則。
使用者提供 nmap 掃描結果時按需載入。

---

## 目錄

1. [一般文字輸出 (-oN)](#一般文字輸出--on)
2. [NSE 弱掃腳本輸出](#nse-弱掃腳本輸出)
3. [XML 輸出 (-oX)](#xml-輸出--ox)
4. [擷取規則總結](#擷取規則總結)

---

## 一般文字輸出 (-oN)

### 端口表格格式

nmap 的標準輸出以表格呈現開放端口：

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.52 ((Ubuntu))
443/tcp  open  ssl/http Apache httpd 2.4.52 ((Ubuntu))
3306/tcp open  mysql    MySQL 8.0.30
```

### 擷取規則

端口表格中每一行：

| 欄位 | 擷取方式 |
|------|---------|
| **端口** | `/` 前的數字（例如 `22`） |
| **協定** | `/` 後到空白前（例如 `tcp`） |
| **狀態** | 第二欄 — 僅處理狀態為 `open` 的行 |
| **服務** | 第三欄（例如 `ssh`、`http`、`ssl/http`） |
| **版本** | 服務欄之後的所有內容（例如 `OpenSSH 8.9p1 Ubuntu 3ubuntu0.6`） |

**特殊情況：**
- `ssl/http` → 服務為 `http`（HTTPS），註記有 SSL 包裝
- `tcpwrapped` → 服務被包裝/過濾，照原樣記錄並加註
- `unknown` → 記錄為「unknown」，可能需要手動調查

### 服務版本清理

為 Discovery Summary 擷取乾淨的版本字串：

| 原始版本字串 | 精簡版本 |
|-------------|---------|
| `OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)` | `OpenSSH 8.9` |
| `Apache httpd 2.4.52 ((Ubuntu))` | `Apache 2.4.52` |
| `MySQL 8.0.30` | `MySQL 8.0.30` |
| `nginx 1.18.0` | `nginx 1.18.0` |

規則：保留產品名稱 + major.minor.patch 版本。移除 OS 特定後綴和括號內的細節。

### 主機狀態

尋找頂部的主機狀態行：

```
Nmap scan report for <hostname> (<ip>)
Host is up (0.0015s latency).
```

擷取 hostname 和 IP 以交叉比對 Scoping Summary。

---

## NSE 弱掃腳本輸出

### 格式辨識

NSE 腳本結果出現在端口表格之後，按端口分組。格式如下：

```
PORT   STATE SERVICE
80/tcp open  http
| http-vuln-cve2021-41773:
|   VULNERABLE:
|   Path Traversal in Apache HTTP Server 2.4.49/2.4.50
|     State: VULNERABLE
|     IDs:  CVE:CVE-2021-41773
|       A flaw was found in a change made to path normalization...
|     Disclosure date: 2021-10-05
|     References:
|       https://nvd.nist.gov/vuln/detail/CVE-2021-41773
|_      https://httpd.apache.org/security/vulnerabilities_24.html
```

### 弱點發現的擷取規則

| 欄位 | 擷取方式 |
|------|---------|
| **受影響端口** | 腳本輸出上方的端口標題行（例如 `80/tcp`） |
| **腳本名稱** | 以 `\|` 開頭加腳本名稱和冒號的行（例如 `http-vuln-cve2021-41773`） |
| **弱點狀態** | 尋找 `State: VULNERABLE` — 僅在狀態為 VULNERABLE 時擷取 |
| **CVE** | 包含 `CVE:` 後接識別碼的行（例如 `CVE-2021-41773`） |
| **標題** | `VULNERABLE:` 之後的描述行（例如「Path Traversal in Apache HTTP Server 2.4.49/2.4.50」） |
| **揭露日期** | 以 `Disclosure date:` 開頭的行 |
| **參考連結** | `References:` 下列出的 URL |

### 弱點狀態映射

| nmap 狀態 | 處理方式 |
|----------|---------|
| `VULNERABLE` | 記錄為暴露 — 類型：`vulnerability` |
| `LIKELY VULNERABLE` | 記錄為暴露 — 加註「likely vulnerable, not confirmed」 |
| `NOT VULNERABLE` | 跳過 — 不記錄 |
| `ERROR` | 跳過 — 若相關則在解析日誌中註記 |

### 常見 NSE 弱掃腳本與暴露類型對應

| 腳本模式 | 暴露類型 |
|---------|---------|
| `*-vuln-*` | `vulnerability` |
| `ssl-*`（弱加密、過期憑證） | `misconfiguration` |
| `http-server-header` | `information-disclosure` |
| `http-title` | `information-disclosure`（若揭露內部資訊） |
| `ssh-auth-methods` | `information-disclosure` |
| `*-brute` | 跳過 — 為主動攻擊腳本，非發現 |

### 非弱點 NSE 腳本輸出

部分預設腳本（`-sC`）產生的資訊輸出可能指示暴露：

```
| ssl-cert: Subject: commonName=example.com
| Not valid after:  2024-01-15T12:00:00
|_ssl-date: TLS randomness does not represent time

| ssh-hostkey:
|   256 SHA256:xxxx (ECDSA)
|_  256 SHA256:yyyy (ED25519)
```

**資訊腳本擷取規則：**
- `ssl-cert` 含過期日期 → 類型：`misconfiguration`，標題：「Expired SSL Certificate」
- `ssl-enum-ciphers` 顯示弱加密（DES、RC4、export）→ 類型：`misconfiguration`，標題：「Weak SSL/TLS Ciphers」
- `http-server-header` 揭露伺服器版本 → 類型：`information-disclosure`，標題：「Server Version Disclosure」

---

## XML 輸出 (-oX)

### 何時使用

使用者提供 `.xml` 檔案路徑時處理 XML。XML 比文字輸出更適合結構化解析。

### 關鍵 XML 元素

```xml
<host>
  <address addr="10.0.0.5" addrtype="ipv4"/>
  <hostnames>
    <hostname name="web-prod-01" type="PTR"/>
  </hostnames>
  <ports>
    <port protocol="tcp" portid="80">
      <state state="open"/>
      <service name="http" product="Apache httpd" version="2.4.52"/>
      <script id="http-vuln-cve2021-41773" output="VULNERABLE:...">
        <table key="CVE-2021-41773">
          <elem key="state">VULNERABLE</elem>
          <elem key="title">Path Traversal in Apache HTTP Server</elem>
        </table>
      </script>
    </port>
  </ports>
</host>
```

### XML 擷取映射

| Discovery 欄位 | XPath / 元素 |
|---------------|-------------|
| IP 位址 | `host/address[@addrtype='ipv4']/@addr` |
| Hostname | `host/hostnames/hostname/@name` |
| 端口 | `host/ports/port/@portid` |
| 協定 | `host/ports/port/@protocol` |
| 服務 | `host/ports/port/service/@name` |
| 產品 | `host/ports/port/service/@product` |
| 版本 | `host/ports/port/service/@version` |
| 腳本 ID | `host/ports/port/script/@id` |
| 腳本輸出 | `host/ports/port/script/@output` |
| CVE | `host/ports/port/script/table/@key`（以 "CVE-" 開頭時） |
| 弱點狀態 | `host/ports/port/script/table/elem[@key='state']` |

---

## 擷取規則總結

### 服務 → Discovery Summary 映射

| nmap 欄位 | Discovery Summary 欄位 |
|----------|----------------------|
| portid | Port |
| protocol | Protocol |
| service name | Service |
| product + version | Version |
| （與 Scoping 比對） | In-Scope |

### 弱點 → Exposure 記錄映射

| nmap 欄位 | Exposure 欄位 |
|----------|-------------|
| 腳本標題/描述 | Title |
| （見類型映射表） | Type |
| （nmap 不提供 — 預設 `medium`，除非 CVE 有 CVSS） | Raw Severity |
| CVE 識別碼 | CVE |
| 端口/服務 | Affected Service |
| "nmap-script" | Source Tool |

**重要：** nmap 不提供 CVSS 分數。當識別到 CVE 時：
- 若能從 AI 知識庫取得 CVSS 分數（大多數知名 CVE 都有文件記錄的 CVSS 分數），則用於 Raw Severity。
- 若 CVSS 分數未知，將 Raw Severity 指定為 `unconfirmed` 並註記：「CVSS score not available — cross-reference with NVD (https://nvd.nist.gov/) required.」向使用者呈現該發現，請其提供 CVSS 分數或接受 `medium` 作為臨時評級。
