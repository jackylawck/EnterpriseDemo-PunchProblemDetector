# 確定性考勤審計決策樹與架構流程圖
## Deterministic Audit Decision Flow & Engine Specification

**專案名稱 (Project):** Enterprise Demo Punch Problem Detector (ED-PPD)  
**引擎類型 (Engine Type):** Client-Side Deterministic Rule Engine (純客戶端確定性規則引擎)  
**技術規範對齊 (Schema Reference):** [`schemas/attendance_audit_schema.json`](../schemas/attendance_audit_schema.json)

---

## 1. 端到端本地審計流水線 (Client-Side Audit Pipeline)

本系統完全摒棄雲端工作流與伺服器端 API，所有 Excel 解析、規則判定與報表重構均於使用者瀏覽器記憶體（RAM Sandbox）內依序執行：

1. **檔案讀取與表頭定位 (Ingestion & Header Discovery):**  
   透過 SheetJS 讀取二進制陣列，於前 15 行內動態檢索 `Department Code` 欄位，並從前 10 行正則提取 `Period` 週期字串，自動鎖定**每月 25 號**為當期考勤截數日（Cutoff Date）。
2. **前置範圍過濾 (Pre-Filtering & Scope Guard):**  
   僅保留 `E` 字頭職員且屬於目標業務部門（`PCD`, `HOF` 等）之記錄；自動剔除截數日之後的未來排班，以及休息日與公眾假期（`REST`, `RES`, `PH`）。
3. **排他性優先級決策樹 (Mutually Exclusive Priority Decision Tree):**  
   每筆有效工作日記錄依序通過四層確定性布林邏輯判定（`Priority 1` 至 `Priority 4`），一旦命中高優先級異常即刻標記並進入下一筆記錄，確保異常分類互斥且不重複計算。
4. **排序與標準化輸出 (Sorting & Workbook Reconstruction):**  
   將所有異常記錄按 `priority` 升序排列（確保「漏打卡」與「全天缺勤」置頂高亮），並重新合併儲存格（`Actual Clocking` 跨 4 欄、`Time IN/OUT` 跨 2 欄），生成對齊原廠規格之 `Problem.xlsx`。

---

## 2. 核心審計決策樹 (Mermaid Decision Flowchart)

```mermaid
flowchart TD
    Start(["上傳原始考勤報表 (.xls / .xlsx)"]) --> ParseHeader["解析表頭與 Period 截數日 (每月 25 號)"]
    ParseHeader --> FilterScope{"是否為目標部門與 E 字頭員工？<br/>且日期 <= 截數日"}
    
    FilterScope -- "否 (No)" --> Skip1(["忽略該筆記錄"])
    FilterScope -- "是 (Yes)" --> FilterRest{"班次或狀態是否為休假？<br/>(REST / RES / PH)"}
    
    FilterRest -- "是 (Yes)" --> Skip2(["豁免：正常休假"])
    FilterRest -- "否 (No)" --> CheckP1{"優先級 1：漏打收工卡？<br/>(Clk1 > 0 或 TimeIN > 0) 且 TimeOUT == 0<br/>且 LastClk < 13:00 且非外勤"}
    
    CheckP1 -- "是 (Yes)" --> TagMissed["標記：MISSED (置頂高亮)"]
    CheckP1 -- "否 (No)" --> CheckP2{"優先級 2：全天缺勤？<br/>PunchCount == 0 且 Reason == ABS"}
    
    CheckP2 -- "是 (Yes)" --> TagAbs["標記：ABS (次高置頂)"]
    CheckP2 -- "否 (No)" --> CheckP3{"優先級 3：遲過九點？<br/>Clk1 > 9.00 且無請假/外勤代碼"}
    
    CheckP3 -- "是 (Yes)" --> TagLate["標記：LATE (遲到)"]
    CheckP3 -- "否 (No)" --> CheckP4{"優先級 4：早過五點半？<br/>非週六 且 Normal >= 7.0<br/>且 0 < max(LastClk, TimeOUT) < 17.30<br/>且無請假/外勤代碼"}
    
    CheckP4 -- "是 (Yes)" --> TagEarly["標記：EARLY (早退)"]
    CheckP4 -- "否 (No)" --> Pass(["合規通過：無考勤異常"])
    
    TagMissed & TagAbs & TagLate & TagEarly --> Export["依優先級排序並生成 Problem.xlsx"]

3. 審計規則與防誤報設計解析 (Audit Rules & False-Positive Prevention)
| 優先級 (Priority) | 異常代碼 (Category) | 數學判定條件 (Deterministic Condition) | 防誤報與企業治理設計 (Governance Rationale) |
|---|---|---|---|
| 1 (置頂) | MISSED | (\text{Clk}_1 > 0 \lor \text{Time IN} > 0) \land \text{Time OUT} = 0 \land \text{Last\_Clk} < 13.00 \land \neg \text{ApprovedDuty} | Time OUT 聯動機制：若人事部已於系統補錄合法收工時間（\text{Time OUT} > 0），即使實體打卡槽位缺失亦自動豁免，杜絕行政補簽後的重複誤報。 |
| 2 (置頂) | ABS | \text{PunchCount} = 0 \land \text{Reason} = \text{'ABS'} | 嚴格區分「完全無打卡之缺勤」與「有打卡但系統誤標 ABS」，確保 HR 優先處理真實曠工或漏交假紙個案。 |
| 3 | LATE | \text{Clk}_1 > 9.00 \land \text{Reason} = \emptyset \land \neg \text{ApprovedDuty} | 凡已標註外勤（SITE）、會議（MEET）、培訓（TRAIN）或請假代碼者無條件放行。 |
| 4 | EARLY | \text{Day} \neq \text{'Saturday'} \land \text{Normal} \ge 7.0 \land 0 < \max(\text{Last\_Clk}, \text{Time OUT}) < 17.30 \land \text{Reason} = \emptyset \land \neg \text{ApprovedDuty} | 自動排除週六短班及標準工時少於 7 小時之特殊排班，並同時比對實體最後打卡與系統核准收工時間，取其較晚者為準。 |
4. 治理與可審計性聲明 (Audit Traceability Note)
本決策樹完全基於確定性布林邏輯（Deterministic Boolean Logic），不涉及任何機率模型、機器學習權重或生成式推論。在任何時間點輸入相同的原始考勤 Excel 檔案，系統均會產出 100% 一致且可完全回溯的審計結果，符合企業內部控制與人力資源合規審查之最高透明度要求。

---

### 建議同步更新 `README.md` 檔案結構樹

建立好這兩個檔案後，您可以順手在 `README.md` 的「專案檔案結構」區塊補上它們，讓整個倉庫的結構呈現更加完整：

```text
ED-PPD/
├── docs/
│   └── audit-decision-flow.md        # 確定性審計決策樹與架構流程圖 (Mermaid)
├── schemas/
│   └── attendance_audit_schema.json  # 標準輸入/輸出數據合約與審計閾值規範
├── index.html                        # 結構骨架、SEO 宣傳標籤與視窗容器
├── app.js                            # 核心業務審計引擎、SheetJS 解析與 Excel 導出
├── i18n.js                           # 國際化雙語模組 (繁體中文 / 英文切換)
├── COMPLIANCE.md                     # 企業級法規適用性判定與科技治理白皮書
├── TERMS.md                          # 法律使用條款、知識產權防護與審計免責聲明
├── LICENSE                           # MIT 開源授權協議條款
├── .gitignore                        # 嚴格禁止任何考勤原檔 (*.xls, *.xlsx) 提交至雲端
└── README.md                         # 專案說明文檔

