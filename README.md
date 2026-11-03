# midterm_exam
# Fast Draw 荒野快槍手 (電子裁判系統)

# Fast Draw 荒野快槍手 (毫秒級電子裁判系統)

## 1. 情境故事：
* **誰**：參加朋友派對、聚會活動的大家（自詡為「荒野快槍手」的挑戰者們）。
* **在哪**：熱鬧的聚會現場、派對桌遊區或歡飲決鬥台。
* **遇到什麼麻煩**：聚會時想玩刺激的毫秒級反應力比拼遊戲（比誰拔槍快、比反應時間），但傳統用人工倒數或手機計時器很容易不公平，肉眼更是無法分辨微小的毫秒差距，容易引起爭議；因此需要一套**自動化、絕對客觀且帶有音效提示的毫秒級電子裁判系統**，來為派對增添緊張刺激的決鬥氛圍！

---

## 2. 替身對照：
* **拔槍觸發感應器 / 擊中目標靶** → 板載 **PC13 按鈕 (User Button B1)**：按下代表拔槍擊中標靶。
* **開火號角 / 蜂鳴器 / 提示燈** → 板載 **PA5 LED (LD2)** / TIM2_CH1 PWM 輸出。
* **裁判終端機 / 主控台** → 電腦端 Serial Terminal（經由 **LPUART1 ST-LINK 虛擬串口** 通訊）。
* **命令替身說明**：輸入 `start N` ＝ 設定 $N$ 秒後開火號角響起（如 `start 3`）。

---

## 3. 功能與驗收條件：

| 測試項目 | 操作動作 | 預期結果 | 實測結果 |
| :--- | :--- | :--- | :--- |
| **串口指令輸入** | 終端機輸入 `start 3` | 系統回應 Ready，並開始 3 秒非阻塞倒數 | **Pass**（顯示倒數提示） |
| **號角觸發** | 倒數時間到達 (3s) | PA5 輸出 1 kHz (50% 佔空比) PWM，串口印出 `>> SIGNAL FIRE! <<` | **Pass**（PWM 精準響應） |
| **玩家成功拔槍** | 號角響起後按下 PC13 | PWM 立刻停止，串口計算並回傳毫秒級反應時間 `HIT! Reaction Time: xx ms` | **Pass**（實測反應時間精確） |
| **玩家超時未反應** | 號角響起超過 2 秒未按 PC13 | PWM 停止，串口輸出 `TOO SLOW! GAME OVER!` 判定失敗 | **Pass**（非阻塞逾時判定正常） |

---

## 4. 用到的技術：
- [x] **GPIO & 中斷控制**：EXTI15_10 (PC13 上升沿觸發，內部下拉電阻，NVIC Priority 2)。
- [x] **中斷去彈跳處理**：基於 `HAL_GetTick()` 1ms 時間戳進行 20ms 非阻塞時間門檻過濾。
- [x] **定時器與 PWM 輸出**：TIM2 Channel 1 產生 1 kHz PWM 訊號（PSC=169, ARR=999），透過 `__HAL_TIM_SET_COMPARE` 寫入 CCR1 暫存器控制 50% 佔空比。
- [x] **UART 通訊與環形緩衝區**：LPUART1 中斷接收 (RXNE)，搭配 64 位元 ($2^6$) 位元遮罩運算 `(tail + 1) & (64 - 1)` 之 Ring Buffer 架構。
- [x] **非阻塞狀態機**：主迴圈基於 `HAL_GetTick()` 時間戳輪詢，實現倒數開火與 2000ms 超時斷定，絕不使用 `HAL_Delay()`。

---

## 5. 腳位表：
* **PA5** ＝ 板載 LED2 (LD2) / TIM2_CH1（PWM 輸出開火號角指示）。
* **PC13** ＝ 板載 User Button B1（EXTI15_10 拔槍擊靶按鈕中斷輸入）。
* **PA2 / PA3** ＝ LPUART1 RX / TX（透過 ST-LINK VCP 與電腦串口終端機連線）。

---

## 6. 建置與執行：
* **開發環境**：STM32CubeIDE (C Language, STM32Cube HAL)。
* **目標硬體**：NUCLEO-G474RE (STM32G474RE @ 170 MHz, HSI16 + PLL, Flash Wait State = 4)。
* **燒錄步驟**：
  1. 將 NUCLEO-G474RE 透過 Micro-USB 接線連接至電腦。
  2. 在 STM32CubeIDE 點擊 **Build** (Hammer 圖示) 確定編譯通過。
  3. 點擊 **Run / Debug** 將 Code 燒錄至開發板。
* **終端機設定**：
  * **Baud Rate**：`115200`
  * **Data Bits**：`8`
  * **Parity**：`None`
  * **Stop Bits**：`1`
  * **Flow Control**：`None` (8N1)
* **命令說明**：
  * `start N`：設定倒數 $N$ 秒後開火（例如 `start 3` 代表 3 秒後響起號角）。

---

## 7. 程式結構：

*```text
Core/
├── Inc/
│   └── main.h             # 腳位與系統定標頭檔
└── Src/
    ├── main.c             # 專案主程式 (含 Ring Buffer、狀態機、指令解析)
    ├── stm32g4xx_it.c     # 中斷向量處理入口 (呼叫 HAL 庫 Callback)
    └── system_stm32g4xx.c # 系統時脈初始化 (170 MHz 配置)

* **ISR 中斷服務程序分工（貫徹零阻塞紀律）**：
  * `HAL_UART_RxCpltCallback()`：將接收到的單一位元組推入 `rx_fifo` Ring Buffer，推進 `head`，並重啟 `HAL_UART_Receive_IT`。
  * `HAL_GPIO_EXTI_Callback()`：進行 20ms 去彈跳檢查，當 `signal_start_ms > 0` 時紀錄 `hit_reaction_ms` 並設定 `hit_flag = 1`。
* **主迴圈 (Main Loop Task) 分工**：
  1. `Process_UART_Commands()`：輪詢並出列 Ring Buffer 資料，解析 `start N` 指令。
  2. **開火觸發檢查**：當達到 `target_fire_time` 時，寫入 CCR1 暫存器（500）並開啟 PWM。
  3. **擊中結果處理**：若 `hit_flag == 1`，停止 PWM 並格式化串口印出反應毫秒數。
  4. **超時判定**：號角響起超過 2000ms 未按按鈕，關閉 PWM 並顯示 `TOO SLOW! GAME OVER!`。

---

## 8. 實作照片：每張附一行圖說

![NUCLEO-G474RE 板載硬體與 PA5/PC13 標示](images/hardware_setup1.png)
*圖 1：NUCLEO-G474RE 開發板實體圖，未啟動時的 PC13 拔槍按鈕與 PA5 開火指示燈。*

![串口輸入 start 3 命令與開火提示](images/hardware_setup2.png)
*圖 2：NUCLEO-G474RE 開發板實體圖，燈光與號角響起時 PC13 拔槍按鈕與 PA5 開火指示燈。*

![成功按下按鈕顯示毫秒級反應時間](images/serial_hit.png)
*圖 3：Serial Terminal 輸入 `start 5` 後系統進入 Ready 倒數狀態並觸發開火訊號。 號角響起後按下按鈕，系統關閉 PWM 並精準回傳反應時間（如 268 ms）。*

---

## 9. AI 使用說明：哪段請 AI 、你怎麼查核

* **AI 協助部分**：
  1. 產生符合 STM32CubeIDE 規範的 `/* USER CODE BEGIN/END */` 模板與結構拆分。
  2. 輔助設計使用 $2^n$ 位元運算 `(tail + 1) & (64 - 1)` 的 Ring Buffer 邏輯與狀態機架構。
* **查核與驗證方式**：
  1. **暫存器計算查核**：親自核對 170 MHz 下 TIM2 Prescaler (`169`) 與 ARR (`999`) 的計算公式，確認 PWM 頻率為精準的 1 kHz。
  2. **ISR 規範審查**：檢查 `HAL_UART_RxCpltCallback` 內無任何 `printf` 或延遲函式，確保中斷反應即時性。
  3. **示波器 / 邏輯分析儀驗證**：量測按鈕觸發到 PWM 停止的時間間隔，確認與 `HAL_GetTick()` 計算之反應時間一致。

---

## 10. 已知限制：哪些用替身、換真貨要改哪裡

* **已知限制與替身**：
  * 目前使用 **PC13 按鈕** 模擬拔槍與擊中，存在機械按鍵彈跳問題。
  * 目前使用 **PA5 板載 LED / 蜂鳴器** 模擬決鬥開火號角。
  * 倒數時間目前為固定 $N$ 秒，缺乏真正的隨機性。
* **換成真貨時的修改指南**：
  1. **替換為光電 / 雷射標靶感應器**：將 PC13 外部中斷改接至外部光敏電路或光電開關腳位，將 `GPIO_PULLDOWN` 依感應器模組改為 `GPIO_NOPULL` 或 Pull-up，並可適當縮短 EXTI 的去彈跳時間（如縮短至 2~5 ms）。
  2. **替換為大功率音效號角 / 大燈**：將 PA5 輸出的 1 kHz PWM 訊號連接至 MOSFET 驅動板或繼電器模組，用以驅動高分貝工業蜂鳴器或強光 Flash。
  3. **加入硬體隨機延遲**：使能 STM32G4 內建的 **RNG (Hardware Random Number Generator)**，將 `random_delay_ms` 改由硬體隨機數生成（如 1000 ms ~ 5000 ms），達成無規則的隨機開火決鬥機制。