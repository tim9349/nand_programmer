# NANDO (NANDopen) 專案學習要點與技術手冊

本文件為 **nand_programmer (NANDO)** 專案的系統化學習資料與技術架構解析手冊。

---

## 📚 專案學習要點總覽

在 `nand_programmer` 專案中，你可以學到以下 4 個核心領域的完整實作：

### 1. 嵌入式韌體開發與 MCU 控制 (firmware)
* **STM32 晶片控制**：學習 STM32 微控制器時脈 (RCC)、GPIO 與暫存器底層驅動設定。
* **USB CDC 通訊協定**：學習如何透過 USB 虛擬串口實現 PC 上位機與 MCU 之間的雙向指令與資料傳輸。
* **韌體更新 (DFU)**：學習線上更新 MCU 韌體 (Bootloader + App 分區跳轉) 的架構設計。

### 2. NAND Flash 與 SPI Flash 底層儲存原理
* **Parallel NAND 匯流排操作**：學習 8-bit 平行介面的 ALE/CLE 時序控制，以及 Read ID、Page Read/Program、Block Erase 命令序列。
* **Bad Block 壞塊管理**：學習如何讀取與處理 OOB (Spare Area)、壞塊檢測與 Skip Bad Block 跳過壞塊策略。
* **晶片自動辨識機制**：學習比對 Flash ID 並動態載入晶片參數檔（如 [qt/nando_parallel_chip_db.csv](file:///home/dm/nand-flash/nand_programmer/qt/nando_parallel_chip_db.csv)）。

### 3. 跨平台 GUI 上位機開發 (qt)
* **Qt / C++ UI 介面設計**：學習製作燒錄器主控圖形介面與燒錄進度/狀態顯示。
* **燒錄器通訊層**：學習在 PC 端封裝自訂 protocol 命令（讀/寫/抹除/備份）與串口資料封包解析。
* **打包與發佈**：學習 Linux Debian 套件 (debian) 及 Windows 安裝檔 (windows) 的軟體打包流程。

### 4. 硬體電路與 PCB 設計 (kicad)
* **KiCad 原理圖與 Layout**：學習編程器主板原理圖與 PCB 佈線設計。
* **TSOP-48 轉接板設計**：學習 NAND/SPI Flash 的 Pinout 設計與轉接板適配。

---

## 📖 核心技術專題詳細說明

### 專題一：USB CDC 通訊協定 vs SDCC 的本質差異與技術拆解

#### 1. USB CDC (Communication Device Class) 通訊協定
* **概念與定位**：USB CDC 是 USB 官方組織訂立的 **裝置類別規範 (Device Class Specification)**。其最常見的子類別為 PSTN (Public Switched Telephone Network) 中的 **ACM (Abstract Control Model)**，簡稱 **Virtual COM Port (VCP, 虛擬串口)**。
* **工作原理**：
  - 當 STM32 晶片作為 USB Device 插上 PC 時，向 Host 主機進行列舉 (Enumeration)，回傳 CDC 類別描述符 (Class Descriptor)。
  - PC 端作業系統（Linux `/dev/ttyACM*` 或 Windows `COM*`）會載入標準串口驅動，將 USB 雙向端點 (Endpoint 1 IN / Endpoint 2 OUT) 包裝成傳統串口。
  - 上位機 (PC) 與 MCU 便可以無視底層複雜的 USB 封包 (Token/DATA/ACK)，直接透過標準 `read()` / `write()` 或 Qt `QSerialPort` 發送與接收流式資料。
* **本專案應用**：NANDO 上位機 Qt 與 STM32 燒錄器韌體間所有的燒錄指令（讀取 ID、擦除 Block、寫入 Page、備份 Flash）皆建立在 USB CDC 雙向極速管道之上。

#### 2. SDCC (Small Device C Compiler) 編譯器
* **概念與定位**：SDCC 是一個 **開源可交叉編譯的 C 語言編譯器工具 (Cross Compiler)**。
* **工作原理**：它將 C 語言原始碼編譯生成目標微控制器（如 8051、Z80、STM8、PICC、DS390 等 8-bit 微控制器）的組譯碼與 `.hex` / `.bin` 機器碼。
* **本質差異對比表**：

| 項目 | USB CDC | SDCC |
|---|---|---|
| **類別屬性** | **硬體傳輸協定 / 裝置類別規範** | **軟體編譯開發工具 (Compiler)** |
| **主要功能** | 定義 PC 與 MCU 之間如何透過 USB 模擬串口傳輸資料 | 將 C 語言原始碼轉換成 MCU 執行之機器碼 |
| **運作位置** | 運行於 USB 物理介面與 OS/MCU 韌體協定棧中 | 運行於開發者 PC 上（編譯期工具）|
| **與本專案關係** | STM32 韌體與 Qt 上位機通訊的核心傳輸協定 | 本專案 STM32 使用 GCC (`arm-none-eabi-gcc`)，而非 SDCC |

---

### 專題二：STM32 微控制器時脈系統 (RCC)、GPIO 與裸機暫存器驅動開發

#### 1. 時脈樹 (RCC / Clock Tree) 初始化與分配
STM32 晶片開機後，預設運行於內部低速 RC 振盪器 (HSI)。要達到最高主頻（例如 STM32F103 72MHz 或 STM32F4/F7/H7 高速運算），必須手動配置 **RCC (Reset and Clock Control)** 暫存器：
1. **開啟 HSE (High-Speed External)**：啟用外部石英晶體振盪器（如 8MHz），並等待 HSE 就位 (`HSERDY`)。
2. **預分頻與 PLL 設定**：將 HSE 信號送入 PLL (相鎖迴路) 設定 M/N/P 分頻與倍頻係數。
3. **切換系統時脈 (SYSCLK)**：待 PLL 穩定後，將 SYSCLK 切換為 PLL 輸出。
4. **匯流排分頻 (AHB / APB1 / APB2)**：透過預分頻器設定 AHB 總線時脈、APB1 (低速週邊) 與 APB2 (高速週邊，如 GPIO/SPI/USART) 時脈。

#### 2. GPIO 暫存器配置
STM32 的每個 GPIO 引腳都受一組 32-bit 暫存器直接控制：
* **MODER (模式暫存器)**：設定引腳為 輸入 (00)、通用輸出 (01)、複用功能 AF (10) 或 類比模式 (11)。
* **OTYPER (輸出類型暫存器)**：設定推挽 Output Push-Pull (0) 或 開汲極 Open-Drain (1)。
* **OSPEEDR (輸出速度暫存器)**：設定邊沿觸發速率（Low / Medium / High / Very High）。
* **PUPDR (上拉/下拉暫存器)**：無電阻 (00)、內部上拉 (01)、內部下拉 (10)。
* **BSRR (Bit Set Reset Register) / ODR (Output Data Register)**：
  - `ODR`：直接讀寫整體暫存器值。
  - `BSRR`：**原子性操作 (Atomic Operation)**。寫入低 16 位拉高引腳，寫入高 16 位拉低引腳，無需 read-modify-write，避免多執行緒或中斷爭奪 (race condition)。

#### 3. 裸機暫存器存取範例
```c
// 1. 開啟 GPIOA 與 GPIOC 的外設時脈 (AHB1/APB2 Bus)
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN | RCC_AHB1ENR_GPIOCEN;

// 2. 將 PA5 設定為通用輸出模式 (Push-Pull, High Speed)
GPIOA->MODER   &= ~(3U << (5 * 2));   // 清除原本 bits
GPIOA->MODER   |=  (1U << (5 * 2));   // 設為 01 (General Purpose Output)
GPIOA->OSPEEDR |=  (3U << (5 * 2));   // 設為 11 (Very High Speed)

// 3. 使用 BSRR 原子拉高 PA5
GPIOA->BSRR = (1U << 5);              // PA5 Output High

// 4. 使用 BSRR 原子拉低 PA5
GPIOA->BSRR = (1U << (5 + 16));       // PA5 Output Low
```

---

### 專題三：韌體線上更新 (DFU / Bootloader) 架構設計與跳轉原理

#### 1. 內部 Flash 記憶體分區架構
線上韌體更新 (DFU) 的核心思想是將 MCU 內部的 Flash 劃分為至少兩個獨立區域：

```
+------------------------+  0x08000000 (Flash 起始位址)
|     Bootloader 區      |  (例如 16KB / 32KB)
|  (固化不變的升級程式)   |
+------------------------+  0x08004000 (App 區起始位址)
|     Application 區     |  
|  (可更換的主功能韌體)   |
+------------------------+  Flash 結束位址
```

#### 2. DFU 完整執行流程
1. **開機自檢**：MCU 上電 Reset 後，晶片硬體固定由 `0x08000000` (Bootloader 起始) 載入向量表並執行 Bootloader。
2. **條件判定**：
   * **進入 DFU 升級模式**：若開機時偵測到特定按鍵按下、或 App 區無效（向量表為 `0xFFFFFFFF`）、或收到特定上位機 USB 命令。
   * **進入正常工作模式**：若判定無須升級，Bootloader 直接跳轉至 App 區。
3. **韌體接收與 Flash 燒錄**：
   * 在 DFU 模式下，MCU 初始化 USB CDC，接收 PC 送來的 `.bin` 檔案封包。
   * 呼叫 STM32 解鎖 Flash (`FLASH_Unlock()`)，依 Sector 單位擦除 App 區。
   * 逐 Byte / Word 將新韌體數據寫入 Flash，並進行 CRC32 校驗。
4. **跳轉至 App 區 (Jump to Application)**：
   * 設定主堆疊指標 MSP (Main Stack Pointer)：將 App 起始位址的第一個 Word 內容寫入 MSP。
   * 重新定位向量表位址 (VTOR)：設定 `SCB->VTOR = 0x08004000`。
   * 設定 Jump 函数並執行：讀取 App 起始位址第二個 Word (Reset Handler)，將 PC (Program Counter) 指針指向該位址執行。

#### 3. 跳轉核心程式碼範例
```c
typedef void (*pFunction)(void);

void Jump_To_Application(uint32_t appAddr) {
    // 檢查 App 起始位址之 MSP 是否合法（應位於 RAM 位址區間 0x20000000）
    if (((*(__IO uint32_t*)appAddr) & 0x2F000000) == 0x20000000) {
        uint32_t JumpAddress = *(__IO uint32_t*)(appAddr + 4);
        pFunction JumpToApp = (pFunction)JumpAddress;

        // 關閉所有中斷，防止跳轉中途觸發舊中斷導致崩潰
        __disable_irq();
        for (int i = 0; i < 8; i++) {
            NVIC->ICER[i] = 0xFFFFFFFF; // 清除 NVIC 中斷致能
            NVIC->ICPR[i] = 0xFFFFFFFF; // 清除 NVIC 中斷掛起
        }

        // 重新設定向量表位址 (VTOR)
        SCB->VTOR = appAddr;

        // 初始化主堆疊指標 MSP
        __set_MSP(*(__IO uint32_t*)appAddr);

        // 執行跳轉
        JumpToApp();
    }
}
```

---

### 專題四：NAND Flash 平行匯流排時序與壞塊管理策略

#### 1. Parallel NAND 時序 (ALE / CLE / WE# / RE#)
* **CLE (Command Latch Enable)**：命令鎖存致能。拉高時，表示當前 DQ 匯流排上傳輸的是命令 (Opcode，如 `0x90` Read ID, `0x80` Page Program)。
* **ALE (Address Latch Enable)**：位址鎖存致能。拉高時，表示當前 DQ 匯流排上傳輸的是位址 (Column / Row Address)。
* **WE# (Write Enable)**：寫入致能。MCU 在 WE# 上升緣將 CLE/ALE/DQ 數據鎖存至 NAND 晶片。
* **RE# (Read Enable)**：讀取致能。MCU 在 RE# 下降緣驅動 NAND 晶片將資料輸出至 DQ 匯流排。

#### 2. OOB (Out Of Band / Spare Area) 與 Skip Bad Block
* **Page 結構**：每個 NAND Page 由 **Main Area**（如 2048 Bytes，放用戶資料）與 **Spare Area / OOB**（如 64 Bytes，放 ECC 與壞塊標記）組成。
* **Bad Block 檢測**：出廠壞塊或使用損毀的 Block，其第一頁或最後一頁的 Spare Area 第一個 Byte 讀出值不為 `0xFF`。
* **Skip Bad Block 策略**：燒錄器寫入影像檔時，逐 Block 掃描；若遇到 Bad Block，自動跳過該 Block，順延寫入下一個正常的 Good Block，確保數據燒錄無誤。

---

### 專題五：Qt / C++ 燒錄器上位機與跨平台發佈

* **Qt GUI**：採用 `QMainWindow` 與 `QThread` 實現多執行緒操作。燒錄耗時任務運行於背景執行緒，避免 UI 介面凍結。
* **通訊協定封裝**：封裝自訂二進位 Protocol，包含包頭 (Header `0xAA55`)、指令碼 (CMD)、長度 (Length)、Payload 與 CRC16/32 校驗碼。
* **Cross-platform Packaging**：Linux 使用 `dpkg-deb` 打包 Debian `.deb` 安裝包；Windows 使用 Inno Setup 或 CPack 產生 `.exe` 安裝檔。

---

### 專題六：KiCad 硬體電路與 PCB Layout

* **原理圖設計**：包含 STM32 主控電路、USB Type-C 介面、3.3V LDO 供電、及 TSOP-48 / SPI 插座邏輯腳位連接。
* **PCB 佈線技巧**：Parallel NAND 8-bit DQ0~DQ7 資料線需做**等長平行佈線 (Length Matching)**，降低 high-speed 時序過衝與 skew；高頻石英晶振下方鋪銅隔離雜訊。
