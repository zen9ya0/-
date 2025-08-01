# 資安事件調查與數位鑑識標準作業程序 (SOP)

本專案提供 **資安事件調查與數位鑑識 SOP 文件**，幫助資安團隊快速應對安全事件，並確保證據完整與法律效力。

## 目錄
1. [目的與範疇](#目的與範疇)
2. [流程總覽](#流程總覽)
3. [詳細流程](#詳細流程)
   - [準備 (Preparation)](#準備-preparation)
   - [偵測與識別 (Detection & Identification)](#偵測與識別-detection--identification)
     - [主動威脅狩獵（Threat Hunting）](#主動威脅狩獵-threat-hunting)
   - [遏止 (Containment)](#遏止-containment)
   - [根除 (Eradication)](#根除-eradication)
   - [復原 (Recovery)](#復原-recovery)
   - [鑑識分析 (Digital Forensics)](#鑑識分析-digital-forensics)
   - [事後檢討 (Lessons Learned)](#事後檢討-lessons-learned)
4. [RACI 權責矩陣](#raci-權責矩陣)
5. [事件時間軸模板](#事件時間軸模板)
6. [常用工具與命令範例](#常用工具與命令範例)


---

## 目的與範疇
本 SOP 用於指導資安事件處理與數位鑑識，確保：
- 快速應對與遏止資安事件。
- 完整保存並分析證據，確保其法律效力。
- 從事件中學習並持續改善防禦能力。

---

## 詳細流程

### 準備 (Preparation)
- 建立資安事件應變計畫 (IR Plan) 與鑑識流程 (DFIR Playbook)。
- 準備工具：
  - **鑑識工具**：FTK Imager、Autopsy、Volatility、X-Ways。
  - **封包分析**：tcpdump、Wireshark、Zeek。
  - **惡意碼分析**：strings、PEiD、Cuckoo Sandbox。
- 定期演練與滲透測試，確認響應團隊熟悉流程。
- 定義事件嚴重度 (Critical/High/Medium/Low)。

---

### 偵測與識別 (Detection & Identification)
- **事件來源**：SIEM、EDR、IDS/IPS、SOC 分析、使用者通報。
- **分析方法**：
  - 日誌分析 (Windows Event Log, Syslog)。
  - pcap 封包檢視（`tcpdump` 或 `Wireshark`）。
  - IOC 比對（YARA、Threat Intelligence Feed）。
- **成果**：
  - 確認事件範圍與類型。
  - 建立初步事件時間軸並保存快照。

---

#### 主動威脅狩獵（Threat Hunting）
在沒有 SIEM、EDR、集中日誌的極端情況下，需透過 **網路流量為核心的主動偵測** 方式進行威脅狩獵。

**方法論：**
1. **建立觀測點 (Visibility)**  
   - 使用核心交換器或防火牆的 Port Mirroring，將流量鏡像到分析伺服器。  
   - 工具：`tcpdump + Zeek + Suricata`。

2. **縮小範圍 (Scoping)**  
   - 先分析出站流量的異常行為（長連線、可疑 DNS 查詢、非標準端口）。  
   - 透過 Zeek 的 `conn.log`、`dns.log` 找出可疑內部 IP。

3. **主機層檢查**  
   - 鎖定可疑 IP 後，批量收集 `netstat`、`tasklist` 或 `ps aux`。
   - 尋找可疑程序（非系統路徑、帶有加密或可疑命令列參數）。

4. **取證與保存**  
   - 對鎖定主機進行記憶體抓取與磁碟影像 (`dd`, `winpmem`)。
   - 計算 SHA256/MD5 哈希值以確保完整性。

5. **快速 IOC 獵殺**  
   - 利用 YARA 扫描常見惡意程式路徑：
     ```bash
     yara -r rules.yar /home/
     yara -r rules.yar C:\Users\
     ```

6. **時間軸重建**  
   - 沒有日誌時，利用 MFT 和記憶體取證：
     ```bash
     log2timeline.py evidence.plaso /mnt/evidence/disk.img
     psort.py -o L2tcsv evidence.plaso > timeline.csv
     ```
---


### 遏止 (Containment)
- **短期措施**：隔離主機 (Quarantine)、封鎖來源 IP、停用可疑帳號。
- **長期措施**：更新防火牆/WAF/IPS 規則，強化權限管理。
- **證據保存**：對受影響系統進行影像備份。

---

### 根除 (Eradication)
- 移除惡意檔案、清除後門及惡意排程。
- 修補漏洞並更新系統安全補丁。
- 執行威脅狩獵 (Threat Hunting)，確認橫向移動跡象。

---

### 復原 (Recovery)
- 使用乾淨映像或備份還原。
- 驗證系統服務可正常安全運行。
- 觀察期（7-14天）加強監控 SIEM 與流量異常。

---

### 鑑識分析 (Digital Forensics)
1. **證據收集**  
   - 全磁碟 bit-to-bit 影像 (`dd if=/dev/sda of=/evidence/disk.img bs=4M` 或 FTK Imager)。
   - 記憶體擷取 (`winpmem`、`Belkasoft RAM Capturer`)。
   - 收集 pcap、伺服器日誌、瀏覽器紀錄。
   - 產生 SHA256/MD5 哈希驗證完整性。
2. **證據保存**  
   - 建立 **Chain of Custody** 文件，記錄所有存取。
   - 證據存放於安全加密磁碟與 WORM 儲存設備。
3. **分析**  
   - **時間軸還原 (Timeline Analysis)**：透過 `log2timeline`、MFT 分析事件順序。
   - **惡意程式分析**：靜態分析 (strings、PE header)、動態分析 (沙箱)。
   - **網路鑑識**：分析 C2 流量、可疑 DNS/SSL 握手行為。
4. **報告**  
   - 記錄攻擊手法、受害範圍、資料外洩狀況。
   - 提出防禦建議與修補策略。

---

### 事後檢討 (Lessons Learned)
- **IR Review Meeting**：討論應變流程缺失與可改進項目。
- **更新 IOC / SIEM 規則**：將新發現的 Indicators 加入偵測規則。
- **教育與強化**：更新 SOP 並培訓相關人員。

---

## RACI 權責矩陣
| 流程階段     | R (執行者) | A (負責者)  | C (諮詢)        | I (知會)      |
|--------------|------------|-------------|-----------------|---------------|
| 偵測與識別   | SOC 分析師 | IR Team Lead| 系統管理員      | CISO          |
| 遏止         | IR 工程師  | IR Team Lead| IT 部門         | CIO           |
| 鑑識分析     | DFIR 專家  | IR Team Lead| 法務、稽核      | CISO          |
| 根除與復原   | IT 部門    | 資安經理    | 應用系統負責人  | CEO           |
| 事後檢討     | 資安經理   | CISO        | 全體 IR 團隊    | 管理層        |

---

## 事件時間軸模板
| 時間 (T+N) | 事件            | 負責人     | 狀態     | 備註          |
|------------|-----------------|------------|----------|---------------|
| T0         | 偵測到異常      | SOC        | 已完成   | SIEM 告警     |
| T1         | 確認事件        | IR Lead    | 已完成   | IOC 確認      |
| T2         | 隔離主機        | IT         | 已完成   | 阻止橫向移動  |
| T3         | 記憶體取證      | DFIR       | 進行中   | 保存證據      |
| T4         | 系統復原        | IT         | 待定     | 驗證中        |
| T5         | 發佈報告        | CISO       | 待定     | ...           |

---

## 常用工具與命令範例
- **磁碟鏡像**：`dd if=/dev/sda of=/mnt/evidence/disk.img bs=4M status=progress`
- **記憶體取證**：`winpmem.exe --output memory.raw`
- **封包擷取**：`tcpdump -i eth0 -w capture.pcap`
- **日誌收集**：`wevtutil epl Security security.evtx`
- **雜湊驗證**：`sha256sum disk.img`

---

## docs內容簡介
- **Incident-Response-SOP.md**：完整的資安事件處理流程。
- **Chain-of-Custody-Template.md**：證據鏈模板，用於記錄證據存取過程。
- **Forensic-Report-Template.md**：鑑識報告範例，提供調查結論格式。
- **tools-commands.md**：常用取證與分析工具命令集。

---

## 授權
本專案可依據你的組織政策選擇授權方式（MIT、Apache 2.0 或內部專用）。
