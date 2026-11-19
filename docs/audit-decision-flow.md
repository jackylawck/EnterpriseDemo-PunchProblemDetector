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
