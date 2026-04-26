# 彈性力學綜合模擬 — 互動 Log 規格說明

> 文件對象：`sping_ruberband.html`
> 最後更新：2026-04-25
> 用途：紀錄學生在「理想彈簧」與「橡皮筋」兩個模擬器中的所有可分析互動行為

---

## 1. 儲存位置與格式

### 1.1 Storage Key
所有互動事件以**單一陣列**儲存於瀏覽器 `localStorage`：

```
Key:  TaskInteractionData
Type: JSON-stringified Array<Event>
```

### 1.2 跨頁與跨 session 行為

| 情境 | 是否保留 |
|---|---|
| 切換 tab（彈簧 ↔ 橡皮筋） | ✅ 保留 |
| 同分頁內 F5 / 重新整理 | ✅ 保留（此乃 localStorage 預設行為） |
| 關閉分頁後重開 | ✅ 保留 |
| 換瀏覽器 / 換裝置 / 隱私模式 | ❌ 不保留 |
| 手動清除瀏覽器 site data | ❌ 不保留 |

> 注意：模擬器**自身的 React state**（如已記錄資料表、控制台鎖定狀態）**不會**寫入 localStorage，重新整理頁面就會重置；只有「互動事件 log」會持續累積。

### 1.3 容量限制

採 **rolling buffer**：超過 500 筆時自動移除最舊的一筆，避免 localStorage 配額被吃滿。

```js
if (interactionHistory.length > 500) interactionHistory.shift();
```

---

## 2. 單筆事件結構

每一筆事件都是一個物件：

```json
{
  "timestamp": "2026-04-25T14:32:18.421Z",   // ISO 8601 UTC，毫秒精度
  "action":    "AddWeight_Spring",            // 事件名稱（見下節）
  "details":   { "amount": 50, "newTotalMass": 250 }   // 隨事件而異的附加資料
}
```

| 欄位 | 型別 | 說明 |
|---|---|---|
| `timestamp` | string | `new Date().toISOString()` 產生，用於前後事件排序與停留時間計算 |
| `action` | string | 事件代號，採 `動作_實驗` 命名（如 `AddWeight_Spring`） |
| `details` | object | 該事件特定的脈絡資料；可能為空物件 `{}` |

---

## 3. 事件名稱完整清單

共 **23 種事件**，分為四類：

### 3.1 全域事件 (App-level)

| Action | 觸發時機 | details 欄位 |
|---|---|---|
| `Initialize` | 應用程式首次載入 | `timestamp`、`sim` (`"Combined_Elasticity_Simulation"`) |
| `SwitchTab` | 切換到不同的實驗 tab | `targetTab` (`"spring"` / `"rubber"`) |

### 3.2 彈簧實驗事件 (Spring)

| Action | 觸發時機 | details 欄位 |
|---|---|---|
| `ChangeMode_Spring` | 切換「單一」/「並聯」模式 | `mode` (`"single"` / `"parallel"`) |
| `TogglePan_Spring` | 掛上 / 取下秤盤 | `isAttached` (boolean) |
| `AddWeight_Spring` | 成功新增砝碼 | `amount` (1/5/10/50)、`newTotalMass` (g) |
| `WeightLimitExceeded_Spring` | 嘗試新增超過 1000g 的砝碼 | `attemptedMass` (g) |
| `ClearWeight_Spring` | 點擊「清空砝碼」 | （無 details） |
| `RecordDataPoint_Spring` | 成功記錄一筆數據 | `mode`、`mass` (g)、`totalLength` (cm) |
| `ClearDataHistory_Spring` | 確認清除實驗數據表 | （無 details） |
| `CalculateTrendline_Spring` | 計算當前模式的線性趨勢 | `mode`、`slope` (g/cm) |
| `ExportDataCSV_Spring` | 下載 CSV | （無 details） |
| `TrendlineInsufficient_Spring` ⭐新增 | 嘗試分析但資料 <2 筆 | `mode`、`currentCount` |
| `AttemptedWhileLocked_Spring` ⭐新增 | 控制台已鎖定（達 10 次）時還呼叫操作函式 | `action` (`"addWeight"` / `"recordData"`)、`amount` 或 `mode` |
| `ExperimentLocked_Spring` ⭐新增 | 一次性 — 累計記錄達 10 次的瞬間 | `totalRecordCount` (=10) |

### 3.3 橡皮筋實驗事件 (Rubber)

| Action | 觸發時機 | details 欄位 |
|---|---|---|
| `TogglePan_Rubber` | 掛上 / 取下秤盤 | `isAttached` (boolean) |
| `AddWeight_Rubber` | 成功新增砝碼 | `amount`、`newTotalMass` |
| `WeightLimitExceeded_Rubber` | 嘗試新增超過 1000g | `attemptedMass` |
| `ClearWeight_Rubber` | 點擊「清空砝碼」 | （無 details） |
| `RecordDataPoint_Rubber` | 成功記錄一筆數據 | `mass` (g)、`totalLength` (cm) |
| `ClearDataHistory_Rubber` | 確認清除實驗數據表 | （無 details） |
| `CalculateTrendline_Rubber` | 計算整體線性趨勢 | `slope` (g/cm) |
| `ExportDataCSV_Rubber` | 下載 CSV | （無 details） |
| `TrendlineInsufficient_Rubber` ⭐新增 | 嘗試分析但資料 <2 筆 | `currentCount` |
| `AttemptedWhileLocked_Rubber` ⭐新增 | 控制台已鎖定（達 30 次）時還呼叫操作函式 | `action`、`amount` |
| `ExperimentLocked_Rubber` ⭐新增 | 一次性 — 累計記錄達 30 次的瞬間 | `totalRecordCount` (=30) |

---

## 4. 教學分析建議

### 4.1 行為訊號對照表

| 訊號 / 衍生指標 | 來源事件 | 教學詮釋 |
|---|---|---|
| **探究時長** | 第一筆 `RecordDataPoint_*` 與最後一筆的 timestamp 差 | 學生實際投入時間 |
| **試誤比率** | `WeightLimitExceeded_*` ÷ `AddWeight_*` | 是否會主動評估器材安全極限 |
| **過早分析** | `TrendlineInsufficient_*` 次數 | 是否搞混「收集資料」與「分析」階段 |
| **模式切換意願** | `ChangeMode_Spring` 次數 | 是否主動進行對照組實驗 |
| **資料完整度** | `RecordDataPoint_*` 在 mass 軸上的均勻程度 | 是否系統性取點而非亂試 |
| **重整後堅持度** | 兩次 `Initialize` 之間的事件量 | 是否在重置後仍續做 |
| **完成率** | 是否出現 `ExperimentLocked_*` | 是否打滿 10 / 30 次上限 |
| **鎖定後行為** | `AttemptedWhileLocked_*` 次數 | 是否在實驗結束後仍想嘗試（探究熱度指標） |

### 4.2 常見學生剖面 (persona) 範例

**A. 系統性探究型**
```
Initialize → TogglePan → (AddWeight × 多次, mass 均勻分布) →
RecordDataPoint × 10 → CalculateTrendline → ExportDataCSV
```

**B. 試誤型**
```
Initialize → AddWeight → WeightLimitExceeded ×3 → ClearWeight →
RecordDataPoint × 2 → TrendlineInsufficient → ...
```

**C. 缺乏耐心型**
```
Initialize → SwitchTab → SwitchTab → SwitchTab → 少於 3 筆 RecordDataPoint
```

---

## 5. 取得 Log 的方法

### 5.1 在瀏覽器 DevTools 直接讀取

```js
// 全部
JSON.parse(localStorage.getItem('TaskInteractionData'))

// 計算事件數量分佈
const data = JSON.parse(localStorage.getItem('TaskInteractionData'));
const counts = data.reduce((m, e) => (m[e.action] = (m[e.action]||0)+1, m), {});
console.table(counts);
```

### 5.2 匯出為 JSON 檔（DevTools Console 一行）

```js
const blob = new Blob(
  [localStorage.getItem('TaskInteractionData')],
  { type: 'application/json' }
);
const a = document.createElement('a');
a.href = URL.createObjectURL(blob);
a.download = `student_log_${new Date().toISOString().slice(0,10)}.json`;
a.click();
```

### 5.3 程式內既有 API

```js
InteractionLogger.clear();   // 重設 log（已存在，但 UI 上未綁定按鈕）
InteractionLogger.log('CustomEvent', { foo: 1 });   // 從外部加 log
```

---

## 6. 後續擴充建議

如需更細顆粒度的行為紀錄，可考慮新增：

| 候選事件 | 教學分析價值 | 實作成本 |
|---|---|---|
| `DuplicateRecord_*` | 學生是否清楚已記錄狀態 | 低（單行） |
| `ModeLimitReached_Spring` | 學生是否會在某模式滿額時主動切換 | 低 |
| `DialogDismissed` | 警告訊息的閱讀停留時間 | 中（需 timestamp 配對） |
| `ChartHover` | 學生是否實際檢視圖表細節 | 高（需節流） |

> 若想加入以上事件，請在對應的 `showAlert()` 或互動 handler 內補上 `InteractionLogger.log(...)` 即可，無需更改 storage 結構。

---

## 7. 隱私與資料倫理提醒

- 此 log **僅儲存於學生本機瀏覽器**，伺服器並未收集
- 若教學場景需匯總到後端，建議：
  1. 在學生明確知情並同意的前提下（IRB / 校內倫理）
  2. 學生 ID **匿名化**（不要把姓名直接寫進 details）
  3. 統一以**班級／學號雜湊**為單位
- 教育研究發表時請註明匿名化方法與 IRB 編號

---

*Schema 版本：v1.1（2026-04-25 補入 TrendlineInsufficient / AttemptedWhileLocked / ExperimentLocked）*
