# Enterprise Demo Punch Problem Detector (ED-PPD)
### 企業打卡異常偵測工具・企業級純前端考勤審計工作站

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Platform](https://img.shields.io/badge/Platform-Client--Side%20Web%20(SheetJS)-green)
![Privacy](https://img.shields.io/badge/Data%20Privacy-Zero--Data%20Retention-emerald)
![Architecture](https://img.shields.io/badge/Governance--as--Code-JSON%20Schema%20%26%20Mermaid-blue)
![Jurisdiction](https://img.shields.io/badge/Jurisdiction-Hong%20Kong%20SAR-orange)

---

## 🌐 線上工作站與架構導覽 (Live Workstation & Architecture)
* **GitHub Pages 線上系統:** [https://jackylawck.github.io/EnterpriseDemo-PunchProblemDetector/](https://jackylawck.github.io/EnterpriseDemo-PunchProblemDetector/)
* **確定性審計決策樹:** [docs/audit-decision-flow.md](docs/audit-decision-flow.md) *(Mermaid 視覺化邏輯流程圖)*
* **系統數據合約規範:** [schemas/attendance_audit_schema.json](schemas/attendance_audit_schema.json) *(JSON Schema 欄位與閾值定義)*

---

## 📖 繁體中文項目簡介

### 1. 系統定位與研發背景
**Enterprise Demo Punch Problem Detector (ED-PPD / 企業打卡異常偵測工具)** 是一個展示企業級無伺服器考勤完整性快篩的開源示範專案。

本系統專為解決每月份人力資源考勤審計中的大量重複性工序而設計。針對考勤系統導出的原始報表，引擎透過純確定性（Deterministic）業務規則，在瀏覽器本地秒級提取打卡異常、漏簽與缺勤記錄，並一鍵導出對齊原廠格式規範之 `Problem.xlsx`（`Problem.Log`）報表。

### 2. 核心架構：零數據留存 (Zero-Data Retention)
* **純客戶端記憶體沙盒運算 (Client-Side Sandboxing):** 所有 Excel 文件的解析、陣列過濾、規則判定與二進制重組均透過 SheetJS 在使用者本機瀏覽器 RAM 中完成。
* **零伺服器傳輸 (Zero Network Transmission):** 系統不設任何後端 API 或雲端資料庫，員工姓名、工號及考勤打卡數據絕不離開本機終端。
* **從設計著手保護隱私 (Privacy by Design):** 不使用 `localStorage`、`IndexedDB` 或追蹤 Cookie；關閉或刷新瀏覽器分頁即刻釋放記憶體，徹底杜絕資料外洩與跨境傳輸風險。

### 3. 確定性業務審計基準 (Deterministic Audit Rules)
系統在執行異常診斷前，會自動鎖定 **每月 25 號為截數日（Cutoff Date）**，僅針對目標業務部門的 `E` 字頭職員進行審計，並無條件排除休息日與公眾假期（`REST`、`RES`、`PH`）：

1. **⚠️ 漏打收工卡 (Missed Checkout - 置頂高亮 / Priority 1):**
   * 上午有有效打卡或系統記錄（$\text{Clk}_1 > 0$ 或 $\text{Time IN} > 0$），但系統收工時間完全未記錄（$\text{Time OUT} = 0.00$），且下午時段無任何打卡（$\text{Last\_Clk} < 13.00$）。
   * **行政補簽聯動豁免:** 若人事部已於系統核准並補錄 `Time OUT`（$> 0$），即使實體打卡槽位缺失亦自動豁免，杜絕重複誤報。
2. **🚨 全天缺勤 (Absence - 置頂高亮 / Priority 2):**
   * 系統狀態標記為 `ABS` 且整天實體打卡次數為 $0$（$\text{PunchCount} = 0$）。
3. **正當外勤與休假全面放行 (Approved Duty Exemption):**
   * 已審批之外勤（`SITE`）、會議（`MEET`）、培訓（`TRAIN`）、活動（`EVENT`）、出差（`TRIP`）及休假（`LEAVE`、`OFF`）代碼全面豁免。
4. **遲過九點 (Late Check-in / Priority 3):**
   * 首度上班打卡遲於上午 9:00（$\text{Clk}_1 > 9.00$），且無請假或正當外勤備註。
5. **早過五點半 (Early Leave / Priority 4):**
   * 星期一至五之標準長工時班次（$\text{Normal} \ge 7.0$ 小時），實體最後打卡與系統核准收工時間皆早於下午 5:30（$0 < \max(\text{Last\_Clk}, \text{Time OUT}) < 17.30$），且無請假或外勤備註。

---

## 🌐 English Project Overview

### 1. System Mission & Scope
The **Enterprise Demo Punch Problem Detector (ED-PPD)** is an enterprise-grade, client-side attendance integrity inspection workstation provided as an open-source demonstration project.

It eliminates manual screening overhead across thousands of monthly timecard entries. Operating on strict deterministic boolean logic, it instantly filters missed checkout punches, unverified absences, late check-ins, and early departures, exporting a standardized audit workbook (`Problem.xlsx`) compatible with legacy HR systems.

### 2. Architecture: Zero-Data Retention
* **Client-Side Sandbox Execution:** All Excel parsing, rule evaluation, sorting, and binary workbook reconstruction execute purely in-memory via SheetJS within the user's browser endpoint.
* **Zero Network Traffic:** Zero backend servers, zero API requests, and zero telemetry logging. Employee Personally Identifiable Information (PII) never traverses external networks.
* **Privacy by Design:** No persistent browser storage (`localStorage`, `IndexedDB`, or cookies) is utilized. All in-memory structures are purged immediately upon tab closure.

### 3. Deterministic Audit Logic Benchmarks
Prior to anomaly evaluation, the engine dynamically parses the reporting period to enforce a **25th-of-the-month cutoff date**, scopes checks to target operational departments (`E`-prefixed staff codes), and excludes scheduled rest days and public holidays (`REST`, `RES`, `PH`):

1. **⚠️ Missed Checkout (Pinned Top / Priority 1):**
   * Valid morning clock-in recorded ($\text{Clk}_1 > 0$ or $\text{Time IN} > 0$), but checkout is completely unrecorded ($\text{Time OUT} = 0.00$) with no afternoon punch ($\text{Last\_Clk} < 13.00$).
   * **Administrative Adjustment Immunity:** Automatically suppresses false positives if `Time OUT` has already been administratively populated ($> 0$).
2. **🚨 Unexcused Absence (Pinned Top / Priority 2):**
   * Tagged as `ABS` with zero physical punch entries ($\text{PunchCount} = 0$).
3. **Approved Official Duty Exemption:**
   * Legitimate off-site and leave codes (`SITE`, `MEET`, `TRAIN`, `EVENT`, `OTHER`, `LEAVE`, `TRIP`, `OFF`) are unconditionally exempted.
4. **Late Check-in (Priority 3):** First punch after 09:00 AM ($\text{Clk}_1 > 9.00$) without approved leave or duty remarks.
5. **Early Leave (Priority 4):** Final checkout before 17:30 PM ($0 < \max(\text{Last\_Clk}, \text{Time OUT}) < 17.30$) on standard weekdays ($\text{Normal} \ge 7.0$ hrs) without approved leave.

---

## 🧪 測試報表格式說明 (Input Excel Structure)

為嚴格遵守零真實個資原則，本倉庫不內含任何 `.xls` 或 `.xlsx` 範例檔案（受 `.gitignore` 保護）。如需建立虛擬 Excel 檔案進行功能測試，請確保工作表包含以下關鍵結構（詳見 [`schemas/attendance_audit_schema.json`](schemas/attendance_audit_schema.json)）：
* **前 10 行內包含週期字串：** 例如 `Period : 26/09/2026 To 25/10/2026`（用於自動鎖定截數日）。
* **前 15 行內包含表頭關鍵字：** `Department Code`（資料列從表頭下方第 2 行開始讀取）。
* **核心欄位索引對應：** `Col[0]` 部門代碼（如 `PCD`, `HOF`）、`Col[2]` 員工編號（如 `E001`）、`Col[3]` 姓名、`Col[5]` 日期、`Col[6]` 星期、`Col[7..10]` 四次原始打卡（小數時間制，如 `9.15`）、`Col[11..12]` 系統 `Time IN` / `Time OUT`、`Col[13]` 班次、`Col[14]` 標準工時、`Col[23]` 狀態備註（如 `ABS`, `SITE`）。

---

## 📂 專案檔案結構 (Repository Structure)

```text
EnterpriseDemo-PunchProblemDetector/
├── docs/
│   └── audit-decision-flow.md        # 確定性審計決策樹與 Mermaid 視覺化流程圖
├── schemas/
│   └── attendance_audit_schema.json  # 標準輸入/輸出數據合約與審計閾值 JSON Schema
├── index.html                        # 結構骨架、CSP 資安標頭與響應式預覽介面
├── app.js                            # 核心確定性業務審計引擎、SheetJS 解析與 Excel 導出
├── i18n.js                           # 國際化雙語模組 (繁體中文 / 英文即時切換)
├── COMPLIANCE.md                     # 企業級法規適用性判定、數據主權與科技治理白皮書
├── TERMS.md                          # 法律使用條款、知識產權防護與審計免責聲明
├── LICENSE                           # MIT 開源授權協議條款
├── .gitignore                        # 嚴格禁止任何考勤原檔 (*.xls, *.xlsx, *.csv) 提交至雲端
└── README.md                         # 專案說明文檔

🏛️ 企業治理與法規宣告 (Governance & Compliance)
本項目技術特徵為「純確定性布林邏輯（Deterministic Boolean Logic）」，具備 100% 數學可解釋性與審計可追溯性，不包含黑箱概率推論或生成式模型。詳細法律與標準對照請查閱專題文件：
 * 合規架構白皮書 (COMPLIANCE.md):
   闡明對香港 Cap. 486《個人資料（私隱）條例》、ISO/IEC 27001、ISO/IEC 27701 的完全合規實踐，並依法出具對 EU AI Act、ISO/IEC 42001、GDPR 及國家網信辦算法備案的非適用性宣告（Statement of Non-Applicability）。
 * 法律條款與審計免責 (TERMS.md):
   明定本系統屬於「管理輔助診斷工具」，確立人事主管人工複核（Human-in-the-Loop）原則，免除演算法直接作為法律紀律處分憑證之連帶責任。
 * 審計決策邏輯規範 (docs/audit-decision-flow.md) & 數據合約 (schemas/attendance_audit_schema.json):
   以機器可讀的 JSON Schema 與標準流程圖固化業務規則，確保跨團隊與內部稽核之透明度。
版權所有 © 2026 Enterprise Demo。本專案依據 MIT License 條款開放授權。

