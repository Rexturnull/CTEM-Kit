# 工具指令參考 — Discovery 階段（中文閱讀版）

> 此檔案為 `tool-commands.md` 的繁體中文翻譯，僅供閱讀參考。

Discovery 階段使用者可執行的掃描指令參考。
此檔案由 Discovery skill 在 Step 0 推薦掃描時按需載入。

---

## nmap — 服務偵測 + 弱點掃描

### 基本服務掃描

在範圍內的端口識別開放服務與版本：

```bash
nmap -sV -sC -p <ports> <target>
```

- `-sV` — 探測開放端口的服務/版本資訊
- `-sC` — 執行預設 NSE 腳本（含基礎列舉）
- `-p <ports>` — 指定 Scoping 中 In-Scope Services 的端口（例如 `-p 22,80,443`）
- 完整發現可使用 `-p-`（全部 65535 端口），但提醒使用者耗時較長

### 弱點掃描

對已知開放端口執行 NSE 弱掃腳本：

```bash
nmap -sV --script vuln -p <ports> <target>
```

- `--script vuln` — 執行「vuln」類別中的所有腳本
- 偵測已知 CVE、錯誤配置與常見弱點
- 需要先有開放端口清單 — 如需要可先執行基本服務掃描

### 服務 + 弱掃合併指令（推薦）

單一指令涵蓋服務偵測與弱點掃描：

```bash
sudo nmap -sV -sC --script vuln -p <ports> -oN nmap-discovery.txt <target>
```

- `-oN nmap-discovery.txt` — 將一般輸出存檔（方便貼上或閱讀）
- 部分腳本操作需要 sudo
- 這是 Discovery 的**推薦預設指令**

### 全端口 + 弱掃

當 Scoping 邊界為「僅主機層級」且需檢查所有端口時：

```bash
sudo nmap -sV -sC --script vuln -p- -oN nmap-full.txt <target>
```

- `-p-` — 掃描全部 65535 端口
- 明顯較慢；僅在需要完整端口掃描時使用

### UDP 服務掃描

當 In-Scope Services 包含 UDP 端口（例如 DNS/53、SNMP/161、NTP/123）：

```bash
sudo nmap -sU -sV --top-ports 50 -oN nmap-udp.txt <target>
```

- `-sU` — UDP 掃描（需要 root/sudo）
- `--top-ports 50` — 掃描最常見的 50 個 UDP 端口（平衡覆蓋率與速度）
- 指定特定 UDP 端口：`sudo nmap -sU -sV -p 53,161,123 -oN nmap-udp.txt <target>`
- UDP 掃描明顯慢於 TCP — 提醒使用者預期較長的掃描時間
- 可與 TCP 合併：`sudo nmap -sS -sU -sV -p T:22,80,443,U:53,161 <target>`

### 輸出格式

| 旗標 | 格式 | 適用場景 |
|------|------|---------|
| `-oN <file>` | 一般文字 | 貼入對話、人工閱讀 |
| `-oX <file>` | XML | 結構化解析、大型輸出 |
| `-oG <file>` | Grepable | 使用 grep/awk 快速篩選 |
| `-oA <base>` | 全部三種 | 完整歸檔 |

大多數情況推薦 `-oN`（容易閱讀和貼上）。大型掃描建議 `-oX` 以利結構化解析。

---

## Nuclei — 模板驅動弱點掃描

### 基本掃描

對目標執行所有預設模板：

```bash
nuclei -u <target> -o nuclei-results.txt
```

- `-u <target>` — 單一目標 URL 或 IP
- `-o <file>` — 將結果存檔
- 預設執行所有啟用的模板

### 依嚴重性篩選掃描

聚焦於特定嚴重性等級以上的發現：

```bash
nuclei -u <target> -severity critical,high,medium -o nuclei-results.txt
```

- `-severity` — 依嚴重性篩選（逗號分隔）
- 減少初步評估時的資訊雜訊

### Web 專用掃描

當目標有 Web 服務（HTTP/HTTPS）時：

```bash
nuclei -u http://<target> -t http/ -o nuclei-web.txt
nuclei -u https://<target> -t http/ -o nuclei-web-ssl.txt
```

- `-t http/` — 僅使用 HTTP 相關模板
- 若兩者都在範圍內，對 HTTP 和 HTTPS 都執行
- 涵蓋：錯誤配置、暴露的管理面板、預設密碼、Web CVE

### JSON 輸出（推薦用於解析）

```bash
nuclei -u <target> -jsonl -o nuclei-results.jsonl
```

- `-jsonl` — 每行一個 JSON 物件（JSON Lines 格式）
- 比純文字更容易結構化解析
- 每行包含：template-id、severity、matched-at、CVE 資訊、描述

### CVE 專用掃描

針對已知的 CVE 進行偵測：

```bash
nuclei -u <target> -t cves/ -o nuclei-cves.txt
```

- `-t cves/` — 僅執行 CVE 偵測模板
- 適合驗證特定已知弱點

### 更新模板

掃描前確保模板為最新：

```bash
nuclei -update-templates
```

- 若使用者近期未更新，建議在掃描前執行

---

## 推薦掃描順序

典型 Discovery 工作階段的建議順序：

### 第 1 步：nmap 服務 + 弱掃
```bash
sudo nmap -sV -sC --script vuln -p <scoped-ports> -oN nmap-discovery.txt <target>
```

### 第 2 步：Nuclei 完整掃描
```bash
nuclei -update-templates && nuclei -u <target> -jsonl -o nuclei-results.jsonl
```

### 第 3 步（Web 服務在範圍內時）：Nuclei Web 掃描
```bash
nuclei -u http://<target> -t http/ -jsonl -o nuclei-web.jsonl
```

告知使用者：
> *「完成後請分享結果 — 直接貼上輸出或告訴我檔案路徑。如果只有其中一個工具也沒關係 — 分享手邊有的，我會據此作業。」*
