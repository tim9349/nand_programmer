# NANDO (NANDopen) 開源 NAND 程式寫入器 — 實作參考

> **本檔角色：** NANDO 專案結構、韌體寫入實作細節、與 STM32H743 的對應參考。
> **建檔日期：** 2026-08-25；2026-10-27 逐段對照 `firmware/programmer/fsmc_nand.c` 更新與更正（見文末〈2026-10-27 更正紀錄〉）
> **來源：** `/home/dm/nand-flash/nand_programmer/` 開源專案分析

---

## 一、專案結構概覽

**NANDO** 是基於 STM32（F1 系列）的開源 NAND Flash 程式寫入器。

| 部分 | 位置 | 角色 |
|---|---|---|
| **韌體（Device-side）** | `firmware/programmer/` | STM32 MCU 邏輯 |
| **PC 應用（Host-side）** | `qt/` | QT GUI 應用（Linux/Windows） |
| **硬體設計** | `kicad/` | PCB 電路圖與 Gerber |
| **Windows 版本** | `windows/` | Windows 原生應用 |

**支援晶片（代表性，完整清單以 CSV 為準）：**
- **Parallel NAND**：K9F2G08U0C, K9F1G08U0E, K9F4G08U0A, HY27US08121B, TC58NVG2S3E, TC58NVG1S3E, F59L2G81A, MX30LF2G18AC, MT29F2G08ABAEA, MT29F4G08ABAD 等（CSV 欄位：名稱、page size、block size、total size）
- **SPI Flash**：AT45DB021D, AT25SF041, MX25L8006E, W25Q16JV
- 詳見 `qt/nando_parallel_chip_db.csv` 與 `qt/nando_spi_chip_db.csv`

**狀態：** ⚠ NOT ACTIVELY SUPPORTED（專案無長期維護）

---

## 二、韌體架構（firmware/programmer/）

### 核心寫入實作檔案

| 檔案 | 內容 |
|---|---|
| **`fsmc_nand.c`** | FSMC NAND 控制器實現（STM32F1 的是 FSMC，不是 H7 的 FMC）+ 讀寫擦操作 |
| **`flash_hal.h`** | Flash HAL 抽象介面（function pointer table） |
| **`nand_bad_block.c`** | Bad block 掃描與標記 |
| **`main.c`** | 整體流程協調 |
| **`usb.c` / `cdc.c`** | USB CDC 通訊層 |
| **`chip_info.h`** | 晶片資訊結構體 |

### 關鍵 API（fsmc_nand.c）

```c
// 讀寫擦操作
int nand_init(void *conf, uint32_t conf_size);          // 初始化 GPIO + FMC
void nand_read_id(chip_id_t *nand_id);                  // 讀晶片 ID（5 byte）
void nand_write_page_async(...);                         // 非同步寫頁
uint32_t nand_read_data(...);                            // 讀資料（支援 offset）
uint32_t nand_read_spare_data(...);                      // 讀 spare area (OOB)
uint32_t nand_erase_block(uint32_t page);               // 擦除塊
uint32_t nand_get_status();                              // 等待操作完成

// 狀態與控制
uint32_t nand_read_status();                             // 讀取 status register（L174）
uint32_t nand_read_page(...);                            // = nand_read_data(offset=0)
uint32_t nand_enable_hw_ecc(bool enable);               // 送 SET_FEATURES(cmd+addr+4 byte) 開關【晶片內建 ECC】，
                                                         // 不是 FSMC 的 ECC 引擎（後者在 nand_fsmc_init 固定 FSMC_ECC_Enable、2048B/頁）
```

### 設定結構 `fsmc_conf_t`（由上位機經 `nand_init(conf, size)` 傳入，packed）

| 欄位群 | 欄位 |
|---|---|
| FSMC 時序 | `setup_time`、`wait_setup_time`、`hold_setup_time`、`hi_z_setup_time`、`clr_setup_time`、`ar_setup_time` |
| 定址週期 | `row_cycles`、`col_cycles`（各 1–4） |
| 命令碼 | `read1_cmd`、`read2_cmd`、`read_spare_cmd`、`read_id_cmd`、`reset_cmd`、`write1_cmd`、`write2_cmd`、`erase1_cmd`、`erase2_cmd`、`status_cmd`、`set_features_cmd` |
| ECC 開關 | `enable_ecc_addr`、`enable_ecc_value`、`disable_ecc_value` |

命令碼為 `0xFF`（`UNDEFINED_CMD`）時該步驟略過。命令值（0x80/0x10/0x60/0xD0/0x70/0x90…）是**設定值**，不是寫死在 C 裡。

### FSMC 接腳（`nand_gpio_init`，STM32F1 LQFP100）

| 信號 | 腳 |
|---|---|
| CLE／ALE | PD11／PD12（= FSMC A16／A17） |
| D0～D3 | PD14、PD15、PD0、PD1 |
| D4～D7 | PE7、PE8、PE9、PE10 |
| NOE／NWE／NCE2 | PD4／PD5／PD7 |
| NWAIT（R/B#） | PD6（上拉輸入，`FSMC_Waitfeature_Enable`） |

全部 AF 推挽 50 MHz（NWAIT 除外）。

### 暴露的 HAL 結構體

```c
flash_hal_t hal_fsmc = {
    .init = nand_init,
    .uninit = nand_uninit,
    .read_id = nand_read_id,
    .erase_block = nand_erase_block,
    .read_page = nand_read_page,
    .read_spare_data = nand_read_spare_data,
    .write_page_async = nand_write_page_async,
    .read_status = nand_read_status,
    .is_bb_supported = nand_is_bb_supported,
    .enable_hw_ecc = nand_enable_hw_ecc,
};
```

---

## 三、NAND 寫入流程（Write Page 詳解）

> 本節與四～六節的 C 碼皆為**節錄／整理**，只保留主要步驟；以原始碼為準。

### 時序結構（ONFI 標準）

```
① CMD CYCLE     ② ADDR CYCLES          ③ DATA CYCLES      ④ CMD2   ⑤ WAIT
┌──────────┐  ┌──────────────────┐  ┌──────────────┐  ┌──────┐
│  0x80    │  │ Col[0] Col[1] Row[0] │  │ DATA[0..page_size-1]  │  │ 0x10 │
│ (PROG)   │  │ Addr  Addr  Addr │  │ byte by byte  │  │(PROG │
└──────────┘  └──────────────────┘  └──────────────┘  └──────┘
    ^           ^                       ^                  ^
 WRITE1_CMD   可配置 1-4 cycle    逐 byte 寫入         WRITE2_CMD
             (col + row)            page_size bytes
```

### 代碼實現（fsmc_nand.c L228-288；以下為節錄，原始碼 col/row 各有 case 1～4 完整分支且無下列說明註解）

```c
static void nand_write_page_async(uint8_t *buf, uint32_t page,
    uint32_t page_size)
{
    uint32_t i;

    // ① 送 WRITE1 命令（通常 0x80）
    *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.write1_cmd;

    // ② 送行內位址（Column Address）：固定全送 0x00（整頁從 column 0 寫起，沒有 column 參數）
    // 週期數取決於 fsmc_conf.col_cycles（1～4，每個 case 就是送 N 個 0x00）
    switch (fsmc_conf.col_cycles) {
    case 2:  // 典型：2K page
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = 0x00;  // col_lo
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = 0x00;  // col_hi
        break;
    // case 1／3／4 同理，送 1／3／4 個 0x00
    }

    // ③ 送列位址（Row Address = page 號，1～4 週期）
    switch (fsmc_conf.row_cycles) {
    case 2:  // 典型
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_1st_CYCLE(page);
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_2nd_CYCLE(page);
        break;
    // case 1／3／4：送 1／3／4 個 ADDR_nth_CYCLE(page)
    }

    // ④ 逐 byte 寫入資料
    for(i = 0; i < page_size; i++)
        *(__IO uint8_t *)(Bank_NAND_ADDR | DATA_AREA) = buf[i];

    // ⑤ 送 WRITE2 命令（0x10 啟動編程）
    if (fsmc_conf.write2_cmd != UNDEFINED_CMD)
        *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.write2_cmd;
    
    // ⚠️ 注意：這裡【不】等待完成（原始碼沒有這段註解；此為本檔說明）
    // 呼叫者須後續 polling status 等 tPROG（200µs~3ms）；上層見 nand_programmer.c
}
```

### 地址編碼巨集

```c
#define ADDR_1st_CYCLE(ADDR)  (uint8_t)((ADDR) & 0xFF)
#define ADDR_2nd_CYCLE(ADDR)  (uint8_t)(((ADDR) & 0xFF00) >> 8)
#define ADDR_3rd_CYCLE(ADDR)  (uint8_t)(((ADDR) & 0xFF0000) >> 16)
#define ADDR_4th_CYCLE(ADDR)  (uint8_t)(((ADDR) & 0xFF000000) >> 24)
```

### FSMC 記憶體分區

```c
#define CMD_AREA    (uint32_t)(1<<16)   // A16 = CLE (Command Latch Enable)
#define ADDR_AREA   (uint32_t)(1<<17)   // A17 = ALE (Address Latch Enable)
#define DATA_AREA   ((uint32_t)0x00000000)

#define Bank2_NAND_ADDR  ((uint32_t)0x70000000)   // 原始碼另有 #define Bank_NAND_ADDR Bank2_NAND_ADDR

// 使用方式：
*(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = 0x80;   // 寫命令
*(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = addr;  // 寫地址
uint8_t data = *(__IO uint8_t *)(Bank_NAND_ADDR);      // 讀資料
```

---

## 四、讀取流程（Read Data）

### nand_read_data() 實作（L290-361）

> 不論 SDR 晶片是否需要 tR 等待，原始碼是**先逐 byte 讀完資料、最後才** `nand_get_status()`；它沒有在送完位址／`0x30` 後先等 busy 再讀。對 tR 很長或需要先等 R/B# 的晶片，這個順序要自己確認（本專案的 H743 實作是先等 ready 再讀）。

```c
static uint32_t nand_read_data(uint8_t *buf, uint32_t page,
    uint32_t page_offset, uint32_t data_size)
{
    uint32_t i;

    // ① 送 READ1 命令
    *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.read1_cmd;

    // ② 送列位址
    switch (fsmc_conf.col_cycles) {
    case 2:
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_1st_CYCLE(page_offset);
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_2nd_CYCLE(page_offset);
        break;
    // case 1／3／4 同理。注意 case 4 在原始碼【沒有 break】，會 fall-through 到 default（結果無害，因 default 什麼也不做）
    }

    // ③ 送行位址
    switch (fsmc_conf.row_cycles) {
    case 2:
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_1st_CYCLE(page);
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_2nd_CYCLE(page);
        break;
    // case 1／3／4 同理
    }

    // ④ 可選：送 READ2 命令（某些晶片需要；原始碼 L354-355）
    if (fsmc_conf.read2_cmd != UNDEFINED_CMD)
        *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.read2_cmd;

    // ⑤ 逐 byte 讀取資料
    for (i = 0; i < data_size; i++)
        buf[i] = *(__IO uint8_t *)(Bank_NAND_ADDR | DATA_AREA);

    // ⑥ 等待讀取完成
    return nand_get_status();
}
```

### Read ID（L210-226）

```c
static void nand_read_id(chip_id_t *nand_id)
{
    uint32_t data = 0;

    *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.read_id_cmd;  // 設定值，ONFI 為 0x90
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = 0x00;  // 地址周期：0x00

    // 讀取 5 byte ID
    data = *(__IO uint32_t *)(Bank_NAND_ADDR | DATA_AREA);
    nand_id->maker_id   = ADDR_1st_CYCLE(data);    // byte 0
    nand_id->device_id  = ADDR_2nd_CYCLE(data);    // byte 1
    nand_id->third_id   = ADDR_3rd_CYCLE(data);    // byte 2
    nand_id->fourth_id  = ADDR_4th_CYCLE(data);    // byte 3

    data = *((__IO uint32_t *)(Bank_NAND_ADDR | DATA_AREA) + 1);
    nand_id->fifth_id   = ADDR_1st_CYCLE(data);    // byte 4
}
```

---

## 五、擦除流程（Erase Block）

### nand_erase_block() 實作（L441-473）

> 參數 `page` 是【要抹除的 block 的第一頁的 page 號（row address）】，不是 block 編號（上層 `np_nand_erase(prog, page)` 傳入，L431）。

```c
static uint32_t nand_erase_block(uint32_t page)
{
    // ① 送 ERASE1 命令
    *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.erase1_cmd;  // 設定值，ONFI 為 0x60

    // ② 送列位址（block 第一頁的 page 號，只送 row，沒有 column）
    switch (fsmc_conf.row_cycles) {
    case 2:
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_1st_CYCLE(page);
        *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_2nd_CYCLE(page);
        break;
    // case 1／3／4：送 1／3／4 個 ADDR_nth_CYCLE(page)
    }

    // ③ 送 ERASE2 命令（設定值，ONFI 為 0xD0）
    if (fsmc_conf.erase2_cmd != UNDEFINED_CMD)
        *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.erase2_cmd;

    // ④ 等待擦除完成
    return nand_get_status();
}
```

---

## 六、狀態輪詢（Status Polling）

### nand_get_status() 實作（L191-208；`nand_read_status` 在 L174-189）

```c
static uint32_t nand_get_status()
{
    uint32_t status, timeout = 0x1000000;

    status = nand_read_status();

    /* 等待操作完成或超時 */
    while (status == FLASH_STATUS_BUSY && timeout) {
        status = nand_read_status();
        timeout--;
    }

    if (!timeout)
        status = FLASH_STATUS_TIMEOUT;

    return status;
}

static uint32_t nand_read_status()
{
    uint32_t data, status;

    *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.status_cmd;  // 設定值，ONFI 為 0x70
    data = *(__IO uint8_t *)(Bank_NAND_ADDR);

    // Status 位元定義
    if ((data & NAND_ERROR) == NAND_ERROR)
        status = FLASH_STATUS_ERROR;                // bit 0 = 1 → Error
    else if ((data & NAND_READY) == NAND_READY)
        status = FLASH_STATUS_READY;                // bit 6 = 1 → Ready
    else
        status = FLASH_STATUS_BUSY;                 // Busy

    return status;
}
```

---

## 七、與 STM32H743 的對應參考

### 架構對比

| 項目 | NANDO（STM32F1） | STM32H743 |
|---|---|---|
| **控制器** | FSMC `Bank2_NAND` @ `0x70000000` | FMC `Bank3` @ `0x80000000` |
| **CLE/ALE** | FSMC 位址線 A16/A17（PD11/PD12，AF） | 若走 FMC 同為位址線；本專案主線改用 GPIO bit-bang |
| **定址週期** | 可配置 1-4 cycle | 同 |
| **Byte I/O** | `*(__IO uint8_t *)` 直接指標 | 同 |
| **Status 等待** | polling loop `timeout=0x1000000` | 應類似 |
| **ECC 支援** | FSMC 硬體 ECC 引擎 + 另可用 SET_FEATURES 開晶片內建 ECC | 可參考 |

> ⚠ **本專案現況（2026-08-14 定案）**：stm32h743 主線是「GPIO bit-bang + sn74lvc8t245 DIR 控制」，FMC 只是已驗證備選（見 `/home/dm/stm32h743/CLAUDE.md`）。NANDO 的 FSMC 位址線＝CLE/ALE 做法只對應 FMC 備選路徑；GPIO 主線沒有「記憶體映射」可借用，只能借流程與命令序列。
> ⚠ NANDO 只覆蓋 SDR 非同步 NAND（ONFI 1.0 類）；沒有 Toggle／NV-DDR、沒有 vendor 命令，**不能**拿來對照 8T24、YMTC 等的 vendor 流程。

### 直接可參考的部分

1. **`nand_write_page_async()`** — Page Program 流程（寫入資料→完成 CMD）
2. **`nand_read_data()`** — 讀資料流程（地址編碼→逐 byte 讀→Status polling）
3. **`nand_get_status()`** — 標準 Status polling 框架（避免 timeout）
4. **地址編碼巨集** — ADDR_*_CYCLE 提取分多位元組
5. **FMC 記憶體分區** — CLE/ALE 用位址線區分的實務方法

### 可移植的設計模式

✅ **函式指標 HAL 抽象**（`flash_hal_t`）— 便於多晶片支援
✅ **配置結構體**（`fsmc_conf_t`）— 時序與命令集可配置
✅ **_async 模式分離**（write 與 status 等待分開）— 便於多工或 DMA
✅ **Bad block 掃描層**（獨立 `nand_bad_block.c`）— 可複用

---

## 八、使用注意事項

⚠️ **`_async` 不代表真的非同步**
- 這只是不等待編程完成就返回
- 必須在外層 call `nand_get_status()` polling tPROG（200µs~3ms）

⚠️ **地址編碼必須正確**
- Column cycles + Row cycles 數量要與晶片規格對齊
- 少送或多送位址週期會定位到完全不同的位置

⚠️ **page_size 通常是 2048 或 4096**
- 不包含 spare area（OOB）
- Spare area 獨立用 `nand_read_spare_data()` 讀

⚠️ **Status 定義要看晶片規格**
- `NAND_ERROR` (bit 0) / `NAND_READY` (bit 6) 是 ONFI 標準
- 部分廠商晶片可能不同

---

## 參考資源

- **NANDO GitHub**：https://github.com/bbogush/nand_programmer
- **ONFI 標準**：`/home/dm/nand-flash/onfi/`
- **STM32H743 FMC 配置**：`/home/dm/stm32h743/memory/stm32_fmc_暫存器配置與通訊操作.md`
- **NAND 通則（ONFI 公開命令／Read ID／Status）**：`/home/dm/nand-flash/memory/nand_vendor_command.md` §0；其餘入口 `/home/dm/nand-flash/memory/index_nand.md`
- **STM32H743 NAND 實作**：`/home/dm/stm32h743/stm32_fw/ok/index_fw.md`

---

## 2026-10-27 更正紀錄（對照原始碼）

| 原敘述 | 更正 |
|---|---|
| 稱 FMC／「GPIO 位址線 A16/A17」 | STM32F1 是 **FSMC**；A16/A17 是 FSMC 位址線（PD11/PD12 AF），不是一般 GPIO |
| 寫頁 col 位址的 `case 3/4` 寫成空殼、並註「如 8T24」 | 原始碼 case 1～4 皆有實作（送 N 個 `0x00`）；「8T24」為誤註，NANDO 不涉及該晶片 |
| 寫頁 column 暗示可指定 | 原始碼 column **固定 0**，整頁從頭寫 |
| 抹除「送行位址（Block 位址）」「超大容量如 8T24」 | 參數是 block 第一頁的 **page 號**；「8T24」誤註刪除 |
| 命令碼註解 0x60/0xD0/0x70/0x90 像是寫死 | 皆為 `fsmc_conf` 設定值 |
| `nand_enable_hw_ecc` 寫成「ECC 控制」 | 實為 SET_FEATURES 開關晶片內建 ECC，與 FSMC ECC 引擎不同 |
| 支援晶片列 5＋3 顆 | CSV 實有 parallel 19 顆、SPI 4 顆（SPI 原漏列 `AT25SF041`）；改為列代表性並註明以 CSV 為準 |
| 讀頁碼省略 `case 4` 缺 `break` | 原始碼 `nand_read_data`／`nand_read_spare_data` 的 `case 4` 沒有 `break`（fall-through 至 `default`，無害） |
