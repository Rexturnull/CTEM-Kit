# 工具指令參考 — Scoping 階段

使用者在 Scoping 過程中可執行的主機資訊蒐集指令。
此檔案由 Scoping skill 按需載入，當使用者需要指令指引時讀取。

## 主機發現

確認目標是否可連線：

```bash
nmap -sn <target>
```

- `-sn` — 僅 ping 掃描，不做端口掃描
- 確認主機存活後再進行後續步驟

## 作業系統偵測

識別作業系統：

```bash
nmap -O <target>
```

- 需要 root/sudo 權限
- 回傳作業系統類型、版本及可信度百分比
- 若準確度偏低，加上 `--osscan-guess` 進行最佳猜測

## 服務列舉

列出開放端口並識別執行中的服務：

```bash
nmap -sV -p- <target>
```

- `-sV` — 探測開放端口以確定服務/版本
- `-p-` — 掃描全部 65535 個端口（較慢但全面）
- 若需較快的初步掃描，使用 `-p 1-1024` 或 `--top-ports 1000`

## 主機名稱解析

IP 與主機名稱互查：

```bash
# IP → 主機名稱
nslookup <ip>
dig -x <ip>

# 主機名稱 → IP
nslookup <hostname>
dig <hostname>
```

## 快速綜合掃描

若使用者希望一道指令同時涵蓋作業系統偵測與服務識別：

```bash
sudo nmap -O -sV --top-ports 1000 <target>
```

此指令可在一次掃描中取得作業系統偵測及最常見 1000 個端口的服務版本資訊。
