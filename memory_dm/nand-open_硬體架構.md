# NANDO 硬體架構與 STM32 組態

> **本檔角色：** NANDO 專案 STM32 硬體選型、FMC 配置、GPIO 腳位分配、PCB 設計要點。
> **硬體對象：** STM32F1 系列 NAND 程式寫入器
> **建檔日期：** 2026-08-25

---

## STM32 選型與編號

### 晶片型號

**NANDO 採用：STM32F1 系列**

根據代碼分析（`firmware/` 中 `#include <stm32f10x.h>`）：
- **型號系列**：STM32F103（或 STM32F10x 通用庫）
- **為什麼選 F1**：
  - 廉價（成本優先）
  - FMC（FSMC）控制器成熟穩定
  - GPIO 腳位充足（D/E/F/G 四個 port）
  - 軟體工具鏈完善
  - 已有官方 NAND driver 參考

### 核心時脈

```c
// 標準配置（72MHz HCLK）
RCC_AHBPeriphClockCmd(RCC_AHBPeriph_FSMC, ENABLE);
```

**HCLK = 72MHz** （STM32F1 典型配置）
- FMC 時序單位 = 1 HCLK = ~13.9ns
- 這影響 Setup/Hold/Hi-Z 時間計算

---

## FMC 控制器配置

### FMC Bank 選擇

```c
#define FSMC_Bank_NAND     FSMC_Bank2_NAND
#define Bank_NAND_ADDR     Bank2_NAND_ADDR 
#define Bank2_NAND_ADDR    ((uint32_t)0x70000000)
```

**選擇理由：**
- Bank2 = 0x70000000（預留 Bank1 給其他用途如 NOR Flash）
- FMC 支援最多 4 個 Bank（Bank1-4）

### FMC 時序結構體

```c
typedef struct __attribute__((__packed__))
{
    uint8_t setup_time;          // Setup time (cycles)
    uint8_t wait_setup_time;     // Wait setup time
    uint8_t hold_setup_time;     // Hold setup time
    uint8_t hi_z_setup_time;     // Hi-Z time（讀資料前釋放延遲）
    uint8_t clr_setup_time;      // CLE setup time
    uint8_t ar_setup_time;       // ALE setup time
    uint8_t row_cycles;          // Row 位址週期（1-4）
    uint8_t col_cycles;          // Column 位址週期（1-4）
    uint8_t read1_cmd;           // Read command 1（通常 0x00）
    uint8_t read2_cmd;           // Read command 2（可選）
    uint8_t read_spare_cmd;      // Read spare area command
    uint8_t read_id_cmd;         // Read ID command（0x90）
    uint8_t reset_cmd;           // Reset command（0xFF）
    uint8_t write1_cmd;          // Write command 1（0x80）
    uint8_t write2_cmd;          // Write command 2（0x10）
    uint8_t erase1_cmd;          // Erase command 1（0x60）
    uint8_t erase2_cmd;          // Erase command 2（0xD0）
    uint8_t status_cmd;          // Status command（0x70）
    uint8_t set_features_cmd;    // Set features（可選）
    uint8_t enable_ecc_addr;     // ECC address（可選）
    uint8_t enable_ecc_value;    // ECC enable value
    uint8_t disable_ecc_value;   // ECC disable value
} fsmc_conf_t;
```

**典型參數（2K page 晶片）：**
```
setup_time = 3 cycles       (~42ns)
wait_setup_time = 1 cycle   (~14ns)
hold_setup_time = 2 cycles  (~28ns)
hi_z_setup_time = 1 cycle   (~14ns)
row_cycles = 2
col_cycles = 2
```

### FMC 初始化流程

```c
static void nand_fsmc_init()
{
    FSMC_NANDInitTypeDef fsmc_init;
    FSMC_NAND_PCCARDTimingInitTypeDef timing_init;

    RCC_AHBPeriphClockCmd(RCC_AHBPeriph_FSMC, ENABLE);

    // 配置時序
    timing_init.FSMC_SetupTime = fsmc_conf.setup_time;
    timing_init.FSMC_WaitSetupTime = fsmc_conf.wait_setup_time;
    timing_init.FSMC_HoldSetupTime = fsmc_conf.hold_setup_time;
    timing_init.FSMC_HiZSetupTime = fsmc_conf.hi_z_setup_time;

    // 配置 FMC Bank
    fsmc_init.FSMC_Bank = FSMC_Bank2_NAND;
    fsmc_init.FSMC_Waitfeature = FSMC_Waitfeature_Enable;    // 等待 R/B# 信號
    fsmc_init.FSMC_MemoryDataWidth = FSMC_MemoryDataWidth_8b;  // 8-bit 平行
    fsmc_init.FSMC_ECC = FSMC_ECC_Enable;                     // ECC 硬體支援
    fsmc_init.FSMC_ECCPageSize = FSMC_ECCPageSize_2048Bytes;  // 2K 頁 ECC
    fsmc_init.FSMC_TCLRSetupTime = fsmc_conf.clr_setup_time;
    fsmc_init.FSMC_TARSetupTime = fsmc_conf.ar_setup_time;
    fsmc_init.FSMC_CommonSpaceTimingStruct = &timing_init;
    fsmc_init.FSMC_AttributeSpaceTimingStruct = &timing_init;
    FSMC_NANDInit(&fsmc_init);

    FSMC_NANDCmd(FSMC_Bank2_NAND, ENABLE);
}
```

---

## GPIO 腳位分配

### 連接規劃（STM32F103）

```c
static void nand_gpio_init(void)
{
    GPIO_InitTypeDef gpio_init;
  
    // 啟用 GPIO 時脈
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD | RCC_APB2Periph_GPIOE | 
        RCC_APB2Periph_GPIOF | RCC_APB2Periph_GPIOG, ENABLE);
  
    // GPIOD：CLE, ALE, D0-D3, NOE, NWE, NCE2
    gpio_init.GPIO_Pin = GPIO_Pin_11 | GPIO_Pin_12 | GPIO_Pin_14 | GPIO_Pin_15 |
        GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_4 | GPIO_Pin_5 | GPIO_Pin_7;
    gpio_init.GPIO_Speed = GPIO_Speed_50MHz;
    gpio_init.GPIO_Mode = GPIO_Mode_AF_PP;  // Alternate Function Push-Pull
    GPIO_Init(GPIOD, &gpio_init);

    // GPIOE：D4-D7
    gpio_init.GPIO_Pin = GPIO_Pin_7 | GPIO_Pin_8 | GPIO_Pin_9 | GPIO_Pin_10;
    GPIO_Init(GPIOE, &gpio_init);

    // GPIOD PD6：NWAIT（R/B# 信號，Input）
    gpio_init.GPIO_Pin = GPIO_Pin_6;
    gpio_init.GPIO_Speed = GPIO_Speed_50MHz;
    gpio_init.GPIO_Mode = GPIO_Mode_IPU;  // Input Pull-Up
    GPIO_Init(GPIOD, &gpio_init); 
}
```

### 腳位對應表（STM32F103 LQFP100）

| GPIO | FMC 信號 | 功能 | 方向 |
|---|---|---|---|
| **PD0** | D2 | Data bit 2 | O |
| **PD1** | D3 | Data bit 3 | O |
| **PD4** | NOE | Output Enable（讀取選訊號） | O |
| **PD5** | NWE | Write Enable（寫入選訊號） | O |
| **PD6** | NWAIT | Wait（R/B# 信號） | I |
| **PD7** | NCE2 | Chip Enable（片選） | O |
| **PD11** | A16 / CLE | Command Latch Enable | O |
| **PD12** | A17 / ALE | Address Latch Enable | O |
| **PD14** | D0 | Data bit 0 | O |
| **PD15** | D1 | Data bit 1 | O |
| **PE7** | D4 | Data bit 4 | O |
| **PE8** | D5 | Data bit 5 | O |
| **PE9** | D6 | Data bit 6 | O |
| **PE10** | D7 | Data bit 7 | O |

**NAND 信號定義：**
- **D0-D7**：8-bit 平行資料線
- **CLE** (A16)：命令鎖存（寫命令時拉高）
- **ALE** (A17)：位址鎖存（寫位址時拉高）
- **WE#**：寫選訊號（低電位有效）
- **RE#**：讀選訊號（低電位有效）
- **CE#**：晶片選訊號（低電位有效）
- **R/B#**：就緒/忙碌訊號（低電位表示忙）

---

## 記憶體分區（FMC Bank2）

### 邏輯地址空間

```c
#define CMD_AREA   (uint32_t)(1<<16)   // A16 = CLE，基址 + 0x10000
#define ADDR_AREA  (uint32_t)(1<<17)   // A17 = ALE，基址 + 0x20000
#define DATA_AREA  ((uint32_t)0x00000000)  // 基址 + 0（無偏移）

#define Bank2_NAND_ADDR  ((uint32_t)0x70000000)
```

**三個邏輯區域：**

| 用途 | 物理位址 | A16 | A17 | 實例 |
|---|---|---|---|---|
| **命令** | 0x70010000 ~ | 1 | 0 | `*CMD = 0x80` |
| **位址** | 0x70020000 ~ | 0 | 1 | `*ADDR = col_lo` |
| **資料** | 0x70000000 ~ | 0 | 0 | `*DATA = buf[i]` |

**軟體使用方式：**
```c
// 寫命令
*(__IO uint8_t *)(Bank2_NAND_ADDR | CMD_AREA) = 0x80;

// 寫位址
*(__IO uint8_t *)(Bank2_NAND_ADDR | ADDR_AREA) = addr_byte;

// 讀/寫資料
uint8_t byte = *(__IO uint8_t *)(Bank2_NAND_ADDR | DATA_AREA);
```

---

## NAND Socket 與連接器

### TSOP-48 Socket

**規格：** TSOP-48（薄小外形封裝，48 pin）

**相容晶片範例：**
- TC58NVG5T2F（平行 NAND）
- TC58TFG8T24（BiCS3 3D NAND）
- HY27US08121B
- MX30LF2G18AC

**Socket 連接：**
- **Pin 1-2**：NC (No Connect)
- **Pin 3-18**：DQ0-DQ7、CE#、DQS、OE#、WE#
- **Pin 19-28**：A0-A5（5 bit 位址，低位用於內部定位）
- **Pin 29-40**：CLE、ALE、RE#、R/B#、VccQ、GND（多組）
- **Pin 41-48**：Vcc、GND（多組）

---

## 硬體設計要點

### 電源設計

**NAND Flash 電源要求：**
- **Vcc**：3.3V（邏輯與 I/O）
- **VccQ**：1.8V 或 3.3V（取決於晶片型號）
- **電流**：平時 ~1mA，編程期間 ~50-100mA（charge pump）

**NANDO 設計：**
```c
// 3.3V NAND power supply（典型）
VccQ = 3.3V
```

**去耦電容建議：**
- 每個 Vcc pin 旁 0.1µF 陶瓷電容
- 靠近 NAND socket，寄生電感 < 5mm

### 訊號完整性

**關鍵訊號（高速）：**
- **DQ0-DQ7**（資料線）：50-100pF 布線寄生容量，需控制
- **WE#/RE#**（控制線）：需邊沿快速切換
- **CLE/ALE**（鎖存線）：時序不同步會造成定址錯誤

**建議：**
- WE#/RE#/CLE/ALE 應盡量靠近 STM32 与 NAND
- 避免長跨距平行走線（交叉串擾）
- 資料線 DQ0-7 可適度加終端電阻（視頻率）

### R/B# 訊號處理

**R/B#（就緒/忙碌）：**
- 晶片產出：開漏輸出（open-drain）
- STM32 端：需上拉電阻（通常板載 10k 上拉）
- 功能：Program/Erase 期間 R/B# 拉低，完成時釋放為高

**代碼中的等待機制：**
```c
// 直接讀 Status register（推薦）
*(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = 0x70;  // Status CMD
uint8_t status = *(__IO uint8_t *)(Bank_NAND_ADDR);

// 或 polling R/B# GPIO（可選）
while (GPIO_ReadInputDataBit(GPIOD, GPIO_Pin_6) == 0) {
    // R/B# 仍低 = 晶片忙
}
```

---

## PCB 佈局建議

### 層數與走線

**推薦 4 層 PCB：**
- **L1（信號層）**：STM32 上方，訊號走線
- **L2（GND）**：完整接地層（重要）
- **L3（Vcc）**：3.3V 電源層
- **L4（訊號層）**：NAND 側，訊號走線

**走線規則：**
1. **訊號完整性**
   - DQ0-7：等長（允差 ±5mm）
   - WE#/RE#：最短路徑
   - CLE/ALE：分開走線，避免平行

2. **回流路徑**
   - GND via 密度：每 5mm 打一個（減少阻抗）
   - 訊號 via 應靠近 pin，減少 stub 長度

3. **電源完整性**
   - Vcc 母線寬度 > 10mm
   - 去耦電容靠近 IC pin（< 5mm）
   - 分層電源（3.3V、1.8V 分開）

### 溫度與 EMI 考量

**工作溫度：**
- STM32F1：-40 ~ 85°C
- NAND Flash：通常 0 ~ 70°C（檢查 datasheet）
- 建議散熱設計：自然對流（小功率）

**EMI 抑制：**
- WE#/RE# 高速邊沿可加 RC 濾波（可選，但會延遲時序）
- CLE/ALE 直接連，不濾波（時序關鍵）
- 長導線加屏蔽（如跨 PCB 的排線）

---

## NANDO vs STM32H743 硬體對比

| 項目 | NANDO (STM32F1) | STM32H743 |
|---|---|---|
| **MCU 型號** | STM32F103 | STM32H743IIT6 |
| **HCLK** | 72MHz | 64MHz (HSI) / 更高 (PLL) |
| **FMC Bank** | Bank2 @ 0x70000000 | Bank3 @ 0x80000000 |
| **時序單位** | ~13.9ns (HCLK) | ~15.6ns (HCLK @64MHz) |
| **GPIO 數量** | 充足（4 port） | 更多（5+ port） |
| **成本** | 低（經典型號） | 高（高端型號） |
| **功能** | 純讀寫擦 | 適合多工應用 |

**選型建議：**
- **NANDO**：學習 / 成本敏感 / 簡單應用
- **STM32H743**：工業級 / 多功能 / 長期支援

---

## 與你的 STM32H743 的設計對應

**可借用的硬體設計思想：**

1. **FMC 時序配置方法** → `fsmc_conf_t` 結構體設計可參考
2. **GPIO 初始化順序** → 先時脈、再 pin 模式、再 FMC 啟能
3. **NAND socket 連接** → 8-bit 平行、CLE/ALE 位址線分區
4. **去耦電容位置** → VccQ pin 旁靠近 socket
5. **R/B# 信號處理** → Status polling 優先（比 GPIO polling 穩定）

**你的板子升級點：**
- H743 更新工藝，時序裕量更大（更容易調試）
- 可用更多 GPIO 做診斷（LED、debug port）
- 支援 DMA，可加速大檔案傳送

---

## 參考資源

- **FMC 初始化代碼**：`firmware/programmer/fsmc_nand.c` L97-121
- **GPIO 配置代碼**：`firmware/programmer/fsmc_nand.c` L65-95
- **STM32F1 HAL 庫**：STM32Cube_FW_F1
- **KiCAD 硬體設計**：`kicad/` 目錄（PCB + 原理圖）
- **STM32H743 對應**：`/home/dm/stm32h743/memory/stm32_fmc_暫存器配置與通訊操作.md`
