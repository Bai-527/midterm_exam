# midterm_exam
# Fast Draw 荒野快槍手 (電子裁判系統)

## 1.
* **誰**：夜晚聚會需要一點小遊戲暖場。
* **在哪**：宿舍或任何聚會地點。
* **遇到什麼麻煩**：。

---

## 2.
* **拔槍觸發感應器 / 擊中目標靶** → 板載 **PC13 按鈕 (User Button B1)**：按下代表拔槍擊中標靶。
* **開火號角 / 蜂鳴器 / 提示燈** → 板載 **PA5 LED (LD2)** / TIM2_CH1 PWM 輸出。
* **裁判終端機 / 主控台**  電腦端 Serial Terminal（經由 **LPUART1 ST-LINK 虛擬串口** 通訊）。
* **命令替身說明**：輸入 `start N` ＝ 設定 $N$ 秒後開火號角響起（如 `start 3`）。

---

## 3. 

| 測試項目 | 操作動作 | 預期結果 | 實測結果 |
| :--- | :--- | :--- | :--- |
| **串口指令輸入** | 終端機輸入 `start 3` | 系統回應 Ready，並開始 3 秒非阻塞倒數 | **Pass**（顯示倒數提示） |
| **號角觸發** | 倒數時間到達 (3s) | PA5 輸出 1 kHz (50% 佔空比) PWM，串口印出 `>> SIGNAL FIRE! <<` | **Pass**（PWM 精準響應） |
| **玩家成功拔槍** | 號角響起後按下 PC13 | PWM 立刻停止，串口計算並回傳毫秒級反應時間 `HIT! Reaction Time: xx ms` | **Pass**（實測反應時間精確） |
| **玩家超時未反應** | 號角響起超過 2 秒未按 PC13 | PWM 停止，串口輸出 `TOO SLOW! GAME OVER!` 判定失敗 | **Pass**（非阻塞逾時判定正常） |

---

## 4. 
- [x] **GPIO & 中斷控制**：EXTI15_10 (PC13 上升沿觸發，內部下拉電阻，NVIC Priority 2)。
- [x] **中斷去彈跳處理**：基於 `HAL_GetTick()` 1ms 時間戳進行 20ms 非阻塞時間門檻過濾。
- [x] **定時器與 PWM 輸出**：TIM2 Channel 1 產生 1 kHz PWM 訊號（PSC=169, ARR=999），透過 `__HAL_TIM_SET_COMPARE` 寫入 CCR1 暫存器控制 50% 佔空比。
- [x] **UART 通訊與環形緩衝區**：LPUART1 中斷接收 (RXNE)，搭配 64 位元 ($2^6$) 位元遮罩運算 `(tail + 1) & (64 - 1)` 之 Ring Buffer 架構。
- [x] **非阻塞狀態機**：主迴圈基於 `HAL_GetTick()` 時間戳輪詢，實現倒數開火與 2000ms 超時斷定，絕不使用 `HAL_Delay()`。

---

## 5. 
* **PA5** ＝ 板載 LED2 (LD2) / TIM2_CH1（PWM 輸出開火號角指示）。
* **PC13** ＝ 板載 User Button B1（EXTI15_10 拔槍擊靶按鈕中斷輸入）。
* **PA2 / PA3** ＝ LPUART1 RX / TX（透過 ST-LINK VCP 與電腦串口終端機連線）。

---

## 6. 
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

## 7. 

```text
Core/
├── Inc/
│   └── main.h             # 腳位與系統定標頭檔
└── Src/
    ├── main.c             # 專案主程式 (含 Ring Buffer、狀態機、指令解析)
    ├── stm32g4xx_it.c     # 中斷向量處理入口 (呼叫 HAL 庫 Callback)
    └── system_stm32g4xx.c # 系統時脈初始化 (170 MHz 配置)

#ISR 中斷服務程序分工（貫徹零阻塞紀律）：

    HAL_UART_RxCpltCallback()：將接收到的單一位元組推入 rx_fifo Ring Buffer，推進 head，並重啟 HAL_UART_Receive_IT。

    HAL_GPIO_EXTI_Callback()：進行 20ms 去彈跳檢查，當 signal_start_ms > 0 時紀錄 hit_reaction_ms = current_tick - signal_start_ms 並設定 hit_flag = 1。

主迴圈 (Main Loop Task) 分工：

    Process_UART_Commands()：輪詢並出列 Ring Buffer 資料，解析 start N 指令。

    開火觸發檢查：當達到 target_fire_time 時，寫入 CCR1 暫存器（500）並開啟 PWM。

    擊中結果處理：若 hit_flag == 1，停止 PWM 並格式化串口印出反應毫秒數。

    超時判定：號角響起超過 2000ms 未按按鈕，關閉 PWM 並顯示 TOO SLOW! GAME OVER!