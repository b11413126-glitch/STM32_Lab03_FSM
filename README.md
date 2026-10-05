# Lab 03：多模式即時切換控制系統 (FSM & Non-blocking Timing)

## 📌 實驗核心目標
- **有限狀態機 (FSM)**：管理 NORMAL、WARNING、EMERGENCY 三種狀態與轉移行為。
- **Tick 非阻塞計時 (Non-blocking)**：使用 `HAL_GetTick()` 計算時間差，取代卡死 CPU 的 `HAL_Delay()`。
- **外部中斷 (EXTI)**：緊急停止採用外部中斷機制，具備最高響應優先級。

---

## 📁 專案目錄結構 (Project Structure)

```text
STM32_Lab03/
├── Core/
│   ├── Inc/
│   │   └── main.h               # 系統常數與腳位標頭檔
│   └── Src/
│       ├── main.c               # 核心程式（FSM 狀態機、Tick 計時、按鍵處理）
│       ├── stm32g4xx_it.c       # 中斷服務常式 (ISR)
│       └── system_stm32g4xx.c   # 系統時鐘設定檔
├── tolab_3.ioc                  # STM32CubeMX 腳位與中斷圖形配置檔
└── README.md                    # 專案說明文件
```

---

## 🛠️ 硬體腳位配置 (Hardware Pinout)

| 類型 | 名稱 / 腳位 | 觸發模式 / 電路設定 | 功能說明 |
| :--- | :--- | :--- | :--- |
| **輸入 (Input)** | `PC9` (MODE) | Pull-up, Active Low | 切換 NORMAL / WARNING 模式 |
| **輸入 (Input)** | `PC13` (STOP) | EXTI Interrupt, Rising Edge | 板載藍色按鈕，觸發 EMERGENCY |
| **輸出 (Output)** | `PC8` (綠燈) | GPIO Output | NORMAL 模式指示燈 (1Hz 閃爍) |
| **輸出 (Output)** | `PC5` (黃燈) | GPIO Output | WARNING 模式指示燈 (2Hz 閃爍) |
| **輸出 (Output)** | `PC6` (紅燈) | GPIO Output | EMERGENCY 模式指示燈 (恆亮) |

---

## 🔄 狀態機設計與轉移矩陣 (FSM Transition Table)

| 當前狀態 (Current) | 觸發事件 (Event) | 下一狀態 (Next) | 狀態行為 (Behavior) |
| :--- | :--- | :--- | :--- |
| **正常 (NORMAL)** | 按下 MODE (PC9) | **WARNING** | 綠燈 1Hz (每 500ms 翻轉)，其餘熄滅 |
| **正常 (NORMAL)** | 按下 STOP (PC13) | **EMERGENCY** | - |
| **警告 (WARNING)** | 按下 MODE (PC9) | **NORMAL** | 黃燈 2Hz (每 250ms 翻轉)，其餘熄滅 |
| **警告 (WARNING)** | **逾時 5 秒** (自動復歸) | **NORMAL** | 5 秒無操作自動返回正常狀態 (進階挑戰) |
| **警告 (WARNING)** | 按下 STOP (PC13) | **EMERGENCY** | - |
| **緊急 (EMERGENCY)**| 按下 RESET (板載硬體重置)| **NORMAL** | 紅燈恆亮，其餘熄滅，忽略模式切換 |

---

## 💡 關鍵程式實作重點

1. **非阻塞閃爍計時 (Non-blocking)**：
```c
if (current_tick - previous_tick >= interval) {
    previous_tick = current_tick;
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_8);
}
```

2. **EXTI 中斷僅立旗標 (Flag)**：
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_13) {
        stop_flag = 1;
    }
}
```
