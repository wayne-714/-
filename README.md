# 歷史版本/目錄說明

1. feat_quant: 2026 NSTC計畫書的初步結果分析代碼。包含bounding box手工框選、畫人測驗繪圖計分、繪圖計分和MADRS關聯性分析
2. Phase1: 用Python收集Wacom數位墨水數據的測試代碼
3. Phase2: 完成初步畫圖介面並實現墨水數據LSL串流
4. Phase2_rev1: 受試者資訊輸入、繪畫類型選擇、新的資料儲存結構
5. Phase2_color: 實現畫筆顏色選擇功能
6. Phase2_rev_div_screen: 若偵測到Wacom，使用雙螢幕介面配置，對話框在主螢幕，畫布在副螢幕
7. Phase2_divscreen_color: 畫筆顏色選擇功能+雙螢幕介面配置
8. Phase2_workspace: 實現workspace功能，不同專案可採用不同測驗配置
9. Phase2_instr: 在workspace編輯頁面可為測驗附加指導語圖檔，測驗開始時會自動開啟
10. Phase2_wsv2 (**當前最新版本**): 優化儲存目錄的名稱與架構，改善繪圖介面視覺效果

# 硬體設備架設

![hardware](./figures/hardware.png)

# 執行繪圖系統

- 打開終端機，創建conda環境(Python版本3.9)，執行`pip install -r requirements.txt`在該環境安裝相關套件
- 前往Phase2_wsv2目錄(`cd sys_dev\Phase2_wsv2`)
- 執行`python main.py`，開啟繪圖介面

# 打包獨立執行檔

- 打開終端機，前往Phase2_wsv2目錄(`cd sys_dev\Phase2_wsv2`)
- 將`WacomDigitalInk.spec`的這兩行

```
PyInstaller.config.CONF['distpath'] = r'G:\我的雲端硬碟\gsn984309.eed06@nctu.edu.tw 2022-09-14 12 20\Documents\研究助理\數位繪圖開發\Wacom\dist'
PyInstaller.config.CONF['workpath'] = r'G:\我的雲端硬碟\gsn984309.eed06@nctu.edu.tw 2022-09-14 12 20\Documents\研究助理\數位繪圖開發\Wacom\build'
```

換成

```
PyInstaller.config.CONF['distpath'] = r'你的專案根目錄\dist'
PyInstaller.config.CONF['workpath'] = r'你的專案根目錄\build'
```

- 執行`python -m PyInstaller WacomDigitalInk.spec`
  > ⚠️ 若出現`ModuleNotFoundError: No module named 'pkg_resources'`報錯，請降級setuptools版本(`pip install "setuptools<70.0.0"`)
- 執行檔`BMLDigitalDrawing.exe`會存在本專案根目錄下的`dist`目錄(若無此目錄會自動創建)

# 墨水/事件檔說明

## `ink_data.csv` — 墨水點資料

每一列代表 Wacom 筆在畫布上的**一個取樣點**。

| 欄位         | 資料來源                                | 說明                                                                |
| ------------ | --------------------------------------- | ------------------------------------------------------------------- |
| `timestamp`  | LSL `local_clock()`                     | 取樣時間戳（秒，LSL 時間軸）                                        |
| `x`          | `tabletEvent.x()`                       | 筆的 X 座標，已扣除工具列，並 normalize 到數位板長寬，範圍` [0, 1]` |
| `y`          | `tabletEvent.y()`                       | 筆的 Y 座標，已扣除工具列，並 normalize 到數位板長寬，範圍` [0, 1]` |
| `pressure`   | `tabletEvent.pressure()`                | 筆壓，範圍 `[0, 1]`                                                 |
| `tilt_x`     | `tabletEvent.xTilt()`                   | 筆在 X 軸方向的傾斜角（度）                                         |
| `tilt_y`     | `tabletEvent.yTilt()`                   | 筆在 Y 軸方向的傾斜角（度）                                         |
| `velocity`   | `PointProcessor` 計算                   | 筆尖移動速度（像素/秒）                                             |
| `stroke_id`  | `LSLIntegration.current_stroke_id`      | 所屬筆劃的編號（每次筆離開畫面後遞增）                              |
| `event_type` | `LSLIntegration.process_ink_point`      | 點的類型：`0`=中間點、`1`=筆劃開始、`2`=筆劃結束                    |
| `color`      | `WacomDrawingCanvas.current_color_name` | 繪製該點時使用的顏色（HEX 字串，如 `#000000`）                      |

> 💡 直/橫向模式畫布的x/y座標方向如下。![painting_direction](./figures/painting_direction.png)

## `markers.csv` — 事件標記

每一列代表一個**離散事件**，無固定採樣率。

| 欄位          | 說明                        |
| ------------- | --------------------------- |
| `timestamp`   | 事件發生的 LSL 時間戳（秒） |
| `marker_text` | 事件描述字串（見下表）      |

### `marker_text` 格式一覽

| 格式                              | 觸發時機                         | 範例                                     |
| --------------------------------- | -------------------------------- | ---------------------------------------- |
| `recording_start`                 | LSL 開始記錄 / 清空畫布後重置    | `recording_start`                        |
| `recording_end`                   | LSL 停止記錄                     | `recording_end`                          |
| `stroke_start_X`                  | 筆劃第一個點（壓力 > 0）         | `stroke_start_3`                         |
| `stroke_end_X`                    | 筆劃最後一個點（壓力 = 0）       | `stroke_end_3`                           |
| `tool_switch\|from:A\|to:B`       | 切換工具（筆 ↔ 橡皮擦）          | `tool_switch\|from:pen\|to:eraser`       |
| `color_switch\|from:A\|to:B`      | 切換顏色                         | `color_switch\|from:#000000\|to:#ff0000` |
| `eraser_X\|deleted_strokes:[...]` | 橡皮擦筆劃結束，記錄被刪除的筆劃 | `eraser_2\|deleted_strokes:[1, 3]`       |
| `recording_paused`                | 選擇繪畫類型對話框開啟，暫停記錄 | `recording_paused`                       |
| `recording_resumed`               | 選擇繪畫類型對話框關閉，恢復記錄 | `recording_resumed`                      |

---

# 兩個檔案的對應關係

```
markers.csv                          ink_data.csv
─────────────────────────────────    ──────────────────────────────────
recording_start  (t=100.000)
stroke_start_0   (t=100.100)    →    event_type=1, stroke_id=0 (t=100.100)
                                     event_type=0, stroke_id=0 (t=100.105)
                                     event_type=0, stroke_id=0 (t=100.110)
stroke_end_0     (t=100.200)    →    event_type=2, stroke_id=0 (t=100.200)
tool_switch|from:pen|to:eraser
eraser_0|deleted_strokes:[0]
stroke_start_1   (t=100.500)    →    event_type=1, stroke_id=1 (t=100.500)
                                     ...
stroke_end_1     (t=100.800)    →    event_type=2, stroke_id=1 (t=100.800)
recording_end    (t=101.000)
```

> 💡 `stroke_id` 是兩個檔案的**主要連結鍵**，可用來對齊標記事件與對應的墨水點序列。

# 程式碼說明(Phase2_wsv2):

![code_structure](./figures/code_structure.png)

## `main.py` 主要功能摘要

### 1. Wacom 繪圖畫布（`WacomDrawingCanvas`）

- **流程管理**：依序引導 Workspace 選擇 → 受試者資訊 → 繪畫類型 → 指導語確認；支援多次繪畫切換，切換時自動暫停 LSL 記錄
- **輸入處理**：接收 Wacom 壓力、傾斜、座標輸入；直向模式自動進行座標轉換；橡皮擦使用邊界框快速過濾 + 碰撞檢測刪除筆劃
- **工具列**：根據 Workspace設定動態顯示筆 / 橡皮擦 / 顏色按鈕；支援橫向 / 直向重建
- **顏色選擇**：支援完整色譜與 12 / 24 / 48 色調色盤；直向模式以旋轉視窗呈現
- **視覺呈現**：壓力值動態調整線條粗細，各筆劃保留獨立顏色；直向模式套用 Qt `-90°` 旋轉

### 2. 繪畫成品展示（`ArtworkDisplayWindow`）

- 直接從 `all_strokes` 資料在副螢幕全螢幕展示成品，依據Workspace設定直向或橫向展示繪畫成品
- 雙螢幕（延伸模式）以全螢幕無邊框顯示，可按 ESC 關閉，或按實驗者控制面板的「❌ 關閉成品」按鈕關閉
- 單螢幕以普通視窗顯示，透過標題列關閉按鈕關閉

### 3. 實驗者控制面板（`ExperimenterControlWindow`）

- 顯示於主螢幕，即時呈現受試者編號、繪畫編號與類型
- 內建碼表記錄作畫用時，提供「新繪畫」與「關閉程式」按鈕（含確認防誤觸）

### 4. 資料儲存與匯出

- 三層目錄結構（受試者 / 收案時間 / 繪畫編號）儲存資料
- 每次繪畫結束自動匯出 PNG 畫布與 `system_log.txt`；關閉程式時完成最終存檔並還原螢幕方向

## `InkProcessingSystemMainController.py` 主要功能摘要

### 1. 系統初始化與模組協調

- 統一初始化並管理所有子模組：`BufferManager`、`RawDataCollector`、`PointProcessor`、`StrokeDetector`、`FeatureCalculator`
- 建立原始點、處理後點、筆劃三層緩衝區，並以 `threading.Lock` 保護 `StrokeDetector` 的共享狀態 （如 current_stroke_points、current_stroke_id變數）

### 2. 時間源管理

- 支援外部時間源注入（如 LSL 時間），優先使用外部時間；失敗時自動回退至系統時間

### 3. 處理管線啟動與多執行緒管理

- 支援兩種輸入模式：
  - **外部輸入模式**（PyQt5 / Wacom）：啟動筆劃偵測、特徵計算、狀態監控三條執行緒
  - **內部模擬模式**：額外啟動點處理執行緒，形成完整四段處理鏈
- 各執行緒透過 `stop_event` 協調停止

### 4. 原始點處理（`process_raw_point`）

- 接收來自 PyQt5 的 Wacom 原始點，轉換為 `RawInkPoint` 資料結構
- 壓力 > 0：經 `PointProcessor` 過濾與平滑後送入 `StrokeDetector`，並觸發 `on_point_processed` 回調
- 壓力 = 0：強制完成當前筆劃，觸發 `on_stroke_completed` 回調，並清空處理器歷史緩存

### 5. 各處理執行緒循環

- **點處理循環**：從 `RawDataCollector` 批量取點，經 `PointProcessor` 處理後存入緩衝區（內部模式用）
- **筆劃偵測循環**：從緩衝區取點送入 `StrokeDetector`，偵測筆劃開始 / 結束，觸發對應回調
- **特徵計算循環**：從筆劃緩衝區取出完成筆劃，呼叫 `FeatureCalculator` 計算特徵並直接觸發回調（不使用 feature buffer）
- **狀態監控循環**：每 5 秒記錄一次處理統計（點數、筆劃數、特徵數等）並觸發狀態回調

### 6. 回調機制

- 提供統一的 `register_callback` / `_trigger_callback` 介面，支援五種事件：`on_point_processed`、`on_stroke_completed`、`on_features_calculated`、`on_error`、`on_status_update`

### 7. 停止與關閉

- `stop_processing`：設置停止旗標、丟棄未完成筆劃、等待所有執行緒結束
- `shutdown`：依序關閉所有子模組，確保系統安全退出

## `StrokeDetector.py` 主要功能摘要

### 1. 筆劃狀態管理（`add_point`）

- 依壓力值判斷筆劃狀態：壓力 > 0 時開始或延續筆劃（`IDLE → ACTIVE`），壓力 = 0 時結束筆劃
- 偵測異常時間間隔（> 0.5 秒）：自動強制完成當前筆劃並開啟新筆劃，防止跨筆劃資料污染
- 狀態一致性保護：若狀態為 `ACTIVE` 但無點資料，自動修正回 `IDLE`

### 2. 筆劃完成與儲存（`finalize_current_stroke`）

- 過濾「幽靈筆劃」（僅含單一結束事件點），不儲存亦不消耗 stroke ID
- 無論驗證是否通過，皆儲存筆劃至 `completed_strokes`，並附加 `is_valid` 標記
- 完成後自動遞增 `stroke_id`，並強制重置狀態為 `IDLE`

### 3. 筆劃有效性驗證（`validate_stroke`）

- 驗證條件：點數 ≥ 2，且總長度（像素）超過最小閾值（預設 10px）
- 長度計算依據畫布尺寸將歸一化座標還原為像素距離

### 4. 狀態重置

- **`force_reset_state`**：保留 stroke ID，有點則執行 finalize；無點則僅重置狀態（用於筆離開畫布）
- **`reset_state`**：完全重置，stroke ID 歸零（用於新繪畫開始）

### 5. 其他功能

- `get_completed_strokes`：取出並清空已完成筆劃緩衝區
- `get_detection_statistics`：回傳偵測統計（偵測數、通過數、拒絕數、總點數）

## `PointProcessor.py` 主要功能摘要

### 1. 初始化與配置驗證

- 初始化平滑緩衝區（`smoothing_buffer`）與歷史點緩存（`history_buffer`，保留最近 10 點）
- 驗證必要配置參數（`smoothing_window_size`、`max_point_distance` 等），並設置品質評估閾值

### 2. 原始點處理主流程（`process_point` / `process_raw_point`）

- 壓力低於閾值的點直接過濾丟棄
- 自動判斷座標是否已歸一化（0~1 範圍），未歸一化則呼叫 `normalize_coordinates` 轉換
- 依序執行：座標正規化 → 衍生特徵計算 → 品質評估 → 平滑濾波
- 處理成功後將結果存入 `history_buffer`；失敗時回傳備用點（信心度 0.5）

### 3. 衍生特徵計算（`_calculate_derived_features`）

- **速度**：依時間差與像素距離計算（像素/秒）
- **方向**：以 `atan2` 計算移動角度（0~2π 弧度）
- **加速度**：依速度變化量與時間差計算
- **曲率**：以三點法計算角度變化除以弧長
- **累積距離**：從筆劃起點累加歐氏距離

### 4. 點品質評估（`validate_point_quality`）

- 檢查座標與壓力值是否在合法範圍內
- 與前一點比較：距離跳躍、時間連續性、速度變化、壓力變化是否超出閾值
- 各異常情況依嚴重程度乘以對應懲罰係數，回傳 0~1 的信心度分數

### 5. 平滑與插值

- **單點平滑**（`_apply_point_smoothing`）：對最近 3 點以指數衰減權重加權平均
- **序列平滑**（`apply_smoothing`）：對整段點序列套用高斯加權移動平均
- **點插值**（`interpolate_points`）：依目標時間間隔在兩點間線性插值，角度插值處理跨 0 點的周期性問題

### 6. 狀態管理

- `clear_history`：筆劃結束時清空歷史緩存，防止跨筆劃速度計算污染
- `reset_statistics`：重置處理統計（總處理數、插值數、平滑數、低品質點數）

## `FeatureCalculator.py` 主要功能摘要

`⚠️ 每完成一個筆劃，系統會在背景執行緒中計算 30+ 種特徵（包含 FFT、ConvexHull、自相關等），但這些結果只是被 print() 後就丟棄了`

### 1. 基本統計特徵（`calculate_stroke_statistics`）

- 計算筆劃總長度（歸一化座標轉像素後計算）、持續時間、點數、邊界框（寬高）
- 壓力與速度統計：平均值、標準差、最大 / 最小值、四分位數、變異係數
- 異常值過濾：以 3 倍標準差為閾值排除離群點後再計算統計量

### 2. 平滑度與複雜度

- **平滑度**（`calculate_smoothness`）：結合 jerk（加速度變化率）與方向變化量，以 6:4 加權平均，值域 0~1
- **複雜度**（`calculate_complexity`）：綜合曲率標準差、方向變化密度、迂曲度（路徑長 / 直線距離）、轉向點密度四項指標

### 3. 顫抖指數（`calculate_tremor_index`）

- 對速度序列進行 Welch 功率譜密度估計，計算 4~12 Hz 顫抖頻段的功率佔比
- 同時計算 X / Y 座標序列的高頻成分（> 2 Hz）佔比
- 三項指標加權合併（0.5 / 0.25 / 0.25），值域 0~1

### 4. 壓力動態特徵（`calculate_pressure_dynamics`）

- 計算壓力建立時間、釋放時間、峰值位置（歸一化）
- 計算壓力穩定性（峰值附近變異係數）、壓力上升 / 下降速率
- 計算壓力不對稱性（建立時間與釋放時間的差異比）

### 5. 節奏特徵（`calculate_rhythm_features`）

- 計算節拍（速度變化峰值的平均頻率）
- 計算時間間隔的節奏規律性（變異係數反轉）
- 計算暫停頻率（低於平均速度 10% 的點比例）
- 計算速度週期性（自相關函數的第一個顯著峰值）

### 6. 幾何特徵（`extract_geometric_features`）

- 計算長寬比、圓度（4π × 面積 / 周長²）、矩形度、凸包比率
- 計算水平 / 垂直對稱性（重心兩側點距離的平均差異）
- 計算形狀描述符：緊密度、伸長度（主軸特徵值比）、實心度（面積 / 凸包面積）

## `RawDataCollector.py` 主要功能摘要

`⚠️ RawDataCollector僅用於產生模擬數據，真實 Wacom 系統中完全用不到`

### 1.設備初始化與管理

- 支援四種設備類型：`wacom`、`touch`、`mouse`、`simulator`
- 以 `DeviceStatus` 枚舉追蹤設備狀態（`DISCONNECTED` → `CONNECTED` → `COLLECTING`）
- 各設備類型以 handler dict 分派，初始化後儲存設備規格至 `device_info`

### 2.數據收集生命週期控制

- `start_collection()`：啟動背景收集執行緒，清空舊佇列並重置統計
- `stop_collection()`：停止執行緒（最多等待 2 秒），計算總時長與平均採樣率
- 以 `threading.Lock` 保護啟動/停止操作的執行緒安全

### 3.數據佇列管理

- 以 `queue.Queue(maxsize=10000)` 作為生產者/消費者緩衝區
- 提供三種取點方式：
  - `get_raw_point()`：單點阻塞式
  - `get_raw_points()`：混合策略（先非阻塞批次取，無資料才等待）
  - `get_raw_points_batch()`：純非阻塞批次取
- 佇列滿時自動丟棄新點並計入 `dropped_points`

### 4.模擬器模式（測試用）

- `_simulate_data_point()`：產生隨機 x/y/pressure/tilt/twist 的假資料點
- 以 `time.sleep(1/sampling_rate)` 模擬真實採樣頻率

### 5.座標校準

- `calibrate_device()`：接受至少 4 個校準點，計算含 scale/offset/rotation 的校準矩陣
- `_create_coordinate_transform()`：產生可套用校準矩陣的座標轉換函式，在收集執行緒中自動套用

### 6.統計監控

- 即時追蹤：總點數、丟失點數、錯誤次數、最後一點時間戳
- `get_collection_statistics()`：若正在收集中，動態計算當前時長與採樣率

## `LSLIntegration.py` 主要功能摘要

### 1. 串流啟動與停止

- `start()`：初始化 LSL 串流、開始數據記錄、推送 `recording_start` 標記，並重置筆劃 ID 與狀態
- `stop()`：推送 `recording_end` 標記、關閉串流、儲存所有數據並回傳已儲存的檔案路徑
- 支援 `with` 語法（context manager），自動管理啟動與停止

### 2. 墨水點處理（`process_ink_point`）

- 接收每個墨水點的座標、壓力、傾斜角、速度、顏色等資訊
- 依 `is_stroke_start` / `is_stroke_end` 旗標推送對應的筆劃開始 / 結束標記
- 過濾無效結束事件（無對應開始的孤立結束點不予記錄）
- 資料同步推送至 LSL 串流（`LSLStreamManager`）與本地記錄器（`LSLDataRecorder`）
- 筆劃結束後才遞增 `stroke_id`，確保同一筆劃的所有點使用相同 ID

### 3. 事件標記

- **工具切換**（`mark_tool_switch`）：記錄筆 / 橡皮擦切換事件
- **顏色切換**（`mark_color_switch`）：記錄顏色變更事件
- **橡皮擦筆劃**（`mark_eraser_stroke`）：記錄橡皮擦 ID 及被刪除的筆劃 ID 列表
- **實驗階段**（`mark_experiment_phase`）：標記基線、任務、休息等實驗階段
- **自訂事件**（`mark_custom_event`）：支援附帶 JSON 格式資料的任意事件標記

### 4. 記錄暫停與恢復

- `pause_recording` / `resume_recording`：在對話框等待期間暫停記錄，並推送對應標記，恢復後繼續記錄

### 5. 狀態查詢

- `get_recording_stats()`：回傳當前筆劃 ID、會話 ID 及數據記錄統計
- `is_recording()`、`get_current_stroke_id()`、`get_session_id()`：提供各項狀態查詢介面

## `LSLStreamManager.py` 主要功能摘要

### 1. 串流配置（`LSLStreamConfig`）

- 定義墨水數據串流（9 個通道：x, y, pressure, tilt_x, tilt_y, velocity, stroke_id, event_type, color_id）與事件標記串流的參數
- 支援設備資訊設定（製造商、型號、序號）及座標標準化範圍

### 2. 串流初始化（`initialize_streams`）

- 建立**墨水數據串流**（`_create_ink_stream`）：設定通道名稱、單位、設備 metadata，並將顏色映射表寫入串流描述
- 建立**事件標記串流**（`_create_marker_stream`）：不規則採樣（`nominal_srate=0`），用於推送字串標記
- 初始化成功後記錄串流開始時間

### 3. 數據推送

- **`push_ink_sample`**：將9個通道的樣本推送至 LSL；支援指定時間戳
- **`push_marker`**：推送字串事件標記（如 `stroke_start_0`、`recording_end`）至標記串流；支援指定時間戳

### 4. 顏色管理（`_get_color_id`）

- 維護顏色名稱 / HEX 值到數字 ID 的映射表（預設 6 種常用色）
- 遇到新顏色時自動分配遞增 ID，並記錄至 log

### 5. 串流關閉與統計

- `close_streams`：推送 `stream_end` 標記、記錄本次使用的顏色映射表，並釋放串流物件
- `get_stats`：回傳總樣本數、標記數、串流時長、使用顏色數等統計資訊
- `get_stream_time`：回傳當前 LSL 時鐘時間（`local_clock()`），供全系統統一時間戳使用

## `LSLDataRecorder.py` 主要功能摘要

### 1. 記錄管理

- `start_recording`：清空緩衝區、設定會話 ID（可自動生成）、記錄開始時間與 metadata
- `record_ink_sample`：將每個墨水點（座標、壓力、傾斜、速度、stroke ID、事件類型、顏色）存入 `ink_samples` 緩衝區
- `record_marker`：將事件標記（時間戳 + 文字）存入 `markers` 緩衝區
- `stop_recording`：停止記錄、觸發資料清理與儲存流程，回傳所有已儲存的檔案路徑

### 2. 無效筆劃清理（`_clean_invalid_strokes_extended`）

清理分五個步驟執行：

1. **去除重複 `stroke_start`**：相同 stroke ID 的重複開始標記只保留第一個
2. **識別無效筆劃**：偵測 `stroke_start → tool_switch(pen→eraser)` 或 `stroke_start → tool_switch(pen→pen)` 模式，標記為無效時間範圍
3. **清理標記**：移除無效時間範圍內的所有標記
4. **過濾 eraser 事件**：若 eraser 事件中引用的 stroke ID 已被清理，整筆移除或部分修正
5. **清理墨水點**：移除無效時間範圍內及 `recording_start` 之前的所有墨水點

### 3. 資料儲存（`_save_data`）

每次停止記錄後自動產生以下六個檔案：

| 檔案               | 內容                                               |
| ------------------ | -------------------------------------------------- |
| `ink_data.csv`     | 清理後的墨水點資料（含顏色欄位）                   |
| `ink_data.json`    | 清理後的墨水點資料（含清理統計）                   |
| `markers.csv`      | 清理後的事件標記                                   |
| `markers_raw.csv`  | 原始事件標記（供對比）                             |
| `ink_data_raw.csv` | 原始墨水點資料（供對比）                           |
| `metadata.json`    | 會話 metadata + 清理統計                           |
| `summary.txt`      | 人類可讀的統計摘要（採樣率、壓力範圍、標記列表等） |

## `SubjectInfoDialog.py` 主要功能摘要

### 1. Workspace 選擇（`WorkspaceSelectionDialog`）

- 掃描 `./workspaces/` 目錄，列出所有 `.workspace.json` 檔案
- 支援以下操作（透過選單列）：
  - **雙擊編輯**：直接進入 `WorkspaceEditorDialog`
  - **新增 / 刪除**：建立空白 Workspace 或移除檔案
  - **複製**：深拷貝後以新 ID 儲存，並自動選中新項目
  - **恢復預設**：覆寫 `default_clinical.workspace.json`
- 操作完成後自動以 `project_id` 重新選中對應列表項目

### 2. Workspace 複製命名（`WorkspaceDuplicateDialog`）

- 輸入新的專案名稱與 ID（預填 `原名稱_copy` / `原ID_copy`）
- 驗證 ID 不含非法檔名字元（`\ / : * ? " < > |`）
- 若目標檔案已存在，提示是否覆寫

### 3. Workspace 編輯器（`WorkspaceEditorDialog`）

- 編輯專案 ID、名稱、版本、描述
- 測驗清單以表格顯示（啟用、順序、繪畫類型、細節說明）
- **即時重排**：修改順序欄位後自動觸發 `_sync_ui_to_workspace()` + `load_workspace_data()` 重新排序
- 使用 `_is_loading_data` 旗標防止 `itemChanged` 回呼造成無限循環
- 儲存時處理 project_id 改名（刪除舊檔、寫入新檔），並同步回原始物件

### 4. 測驗配置編輯器（`TestConfigEditorDialog`）

- 編輯繪畫類型代碼、細節說明、順序、副螢幕方向（橫 / 直向）
- 設定工具列：筆、橡皮擦、顏色選擇器（支援 12 / 24 / 48 色盤及完整色譜）
- 瀏覽並設定施測者 / 受試者指導語檔案（支援 PDF / TXT / 圖片 / DOCX）
- 驗證繪畫類型代碼不重複（排除自身）

### 5. 受試者資訊輸入（`SubjectInfoDialog`）

- 輸入受試者編號、姓名、生日、性別
- 自動產生三層目錄命名規則：
  - 第一層：`受試者ID_姓名_生日`
  - 第二層：`受試者ID_姓名_收案年月日_時間`
  - 第三層（繪畫）：`編號_繪畫類型_收案年月日_時間`

### 6. 繪畫類型選擇（`DrawingTypeDialog`）

- 依 `order` 排序顯示已啟用的測驗
- **展示繪畫成品**按鈕：呼叫 `canvas_ref.show_artwork_display()`，展示期間鎖定開始 / 取消按鈕
- 關閉對話框時自動關閉成品展示視窗

### 7. 受試者指導語（`ParticipantInstructionDialog`）

- 支援圖片（PNG / JPG）、純文字（TXT）、其他格式（顯示提示文字）
- **直向模式**（`set_rotation(-90)`）：以 `QGraphicsView + QGraphicsProxyWidget` 旋轉整個 UI 內容，視覺上呈現直向版面

### 8. Workspace 配置雜湊與摘要

- `generate_workspace_hash`：以 MD5 計算 Workspace 配置的唯一雜湊值
- `save_workspace_config_summary`：
  - 配置未變更時跳過寫入
  - 配置變更時以 **append 模式**追加新配置至 `workspace_config_summary.txt`
- `save_drawing_config_to_metadata`：將當前測驗配置（工具、限制條件等）寫入 `metadata.json`

## `Config.py` 主要功能摘要

### 1. UI 相關配置（Workspace 系統）

#### `ColorPickerMode`（Enum）

- 定義顏色選擇器的五種模式：停用、12 色、24 色、48 色調色盤、完整色譜

#### `InstructionConfig`

- 控制指導語對話框顯示、文字內容、持續時間
- 儲存施測者 / 受試者指導語檔案路徑

#### `ToolbarConfig`

- 設定筆、橡皮擦、顏色選擇器的啟用狀態與模式
- 內建 48 色調色盤（分三層：12 / 24 / 48 色）
- 控制自訂顏色、復原、清除畫布功能的開關

#### `DrawingTestConfig`

- 單一測驗的完整配置：繪畫類型代碼、細節說明、順序、啟用狀態
- 包含副螢幕方向（`landscape` / `portrait`）
- 組合 `ToolbarConfig`、`DrawingConstraints`、`InstructionConfig`

#### `WorkspaceConfig`

- 管理整個專案的配置，包含測驗序列與全域設定
- `to_dict` / `from_dict`：支援 JSON 序列化與反序列化
- `save_to_file` / `load_from_file`：直接讀寫 `.workspace.json` 檔案

#### `get_default_workspace()`

- 產生包含 pretest、DAP、HTP、FD 四個測驗的預設臨床配置

### 2. 系統處理配置（`ProcessingConfig`）

#### 主要參數群組

| 群組     | 代表參數                                                                                             |
| -------- | ---------------------------------------------------------------------------------------------------- |
| 設備基本 | `device_type`、`target_sampling_rate`、`canvas_width/height`                                         |
| 筆劃偵測 | `pressure_threshold`、`pause_duration_threshold`、`stroke_timeout`、`min/max_stroke_duration/length` |
| 點處理   | `max_point_distance`、`max_velocity_jump`、`max_pressure_jump`、`min/max_time_delta`                 |
| 插值     | `interpolation_method`、`enable_interpolation`、`max_interpolation_points`                           |
| 品質控制 | `enable_quality_check`、`quality_score_threshold`、`outlier_threshold`                               |
| 緩衝區   | `point_buffer_size`、`stroke_buffer_size`、`raw_data_buffer_size`                                    |
| 模擬器   | `simulator_noise_level`、`simulator_latency`、`simulator_jitter`                                     |

#### 設備預設值（`_adjust_device_specific_settings`）

| 設備      | 採樣率 | 暫停閾值 | 最小筆劃時間 |
| --------- | ------ | -------- | ------------ |
| Wacom     | 200 Hz | 0.3 s    | 0.02 s       |
| Touch     | 100 Hz | 0.6 s    | 0.08 s       |
| Mouse     | 100 Hz | 0.8 s    | 0.1 s        |
| Simulator | 100 Hz | 0.5 s    | 0.05 s       |

### 3. 工具函數

- **`validate_config`**：驗證設備類型、特徵類型、插值方法、座標系統、筆劃偵測方法、資料格式的合法性，回傳 `(bool, 錯誤訊息)`
- **`create_config_from_device_type`**：以設備類型為基礎建立配置，支援 `**kwargs` 覆寫任意參數
- **`get_config_summary`**：回傳人類可讀的配置摘要字串
- **`create_workspaces_directory`**：模組載入時自動建立 `./workspaces/` 目錄並儲存預設配置

## `reconstruct.py` 主要功能摘要

### 1. 初始化與資料讀取

- **`__init__`**：接受畫布尺寸參數；若未指定，延遲至讀取 `metadata.json` 時決定
- **`load_metadata`**：讀取同目錄的 `metadata.json`，提取畫布尺寸、受試者 ID、繪畫類型等資訊
- **`load_ink_data`**：讀取 `ink_data.csv`，驗證必要欄位，自動偵測座標類型（歸一化 vs 像素），補齊缺少的 `color` 欄位（預設黑色）
- **`load_markers`**：讀取 `markers.csv`，統計各類標記數量（stroke_start / end、eraser、color_switch、canvas_cleared）

### 2. 筆劃解析（`parse_strokes`）

- 依 `event_type`（1=開始、0=中間、2=結束）與 `stroke_id` 分割筆劃
- 自動判斷座標類型，歸一化座標乘以畫布尺寸轉換為像素
- 每個筆劃記錄為 `{'points': [(x, y, pressure), ...], 'color': '顏色字串'}`
- 處理未完成筆劃（無結束事件的最後一筆）

### 3. 刪除事件解析與應用

- **`parse_eraser_events`**：以正則表達式解析 `eraser_X|deleted_strokes:[...]` 格式，支援同一橡皮擦 ID 的累積刪除
- **`parse_canvas_clear_events`**：找出 `canvas_cleared` 標記，將其之前所有已結束的筆劃加入清除集合
- **`apply_deletion_events`**：合併橡皮擦與清空畫布的刪除集合，從筆劃字典中移除對應筆劃

### 4. 繪圖重建（`reconstruct_drawing`）

- 建立白色背景的 `QPixmap`，啟用抗鋸齒
- 依 `stroke_id` 排序繪製，每筆劃以 `_parse_color` 解析顏色（支援 HEX 與顏色名稱）
- **極短筆劃**（移動距離 < 3 px）：視為單點，以平均壓力計算筆寬後繪製圓點
- **正常筆劃**：逐段繪製線段，筆寬依各點壓力動態調整（`1 + pressure × 5`），使用圓頭筆刷
- 最終輸出為 PNG 檔案

### 5. 完整處理流程（`process`）

```
讀取 metadata → 讀取 ink_data.csv → 讀取 markers.csv
    → 解析筆劃 → 解析橡皮擦事件 → 解析清空畫布事件
        → 應用刪除事件 → 重建繪圖 → 輸出 PNG
```

### 6. 主程式（`main`）

- 以 `QFileDialog` 選擇 `ink_data.csv` 檔案
- 畫布尺寸自動從同目錄的 `metadata.json` 讀取（預設 1800×700）
- 處理完成後詢問是否開啟圖片（跨平台支援 Windows / macOS / Linux）
