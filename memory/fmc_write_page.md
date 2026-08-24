# NANDO FMC Write Page — 頁面寫入流程完整說明

> **本檔角色：** NANDO 專案 FMC 方式頁面寫入（Page Program）詳細說明。
> **方式：** FMC 自動時序（Flexible Static Memory Controller）
> **硬體：** STM32F1 FMC Bank2
> **建檔日期：** 2026-08-25

---

## 寫入方式：FMC vs GPIO

### ✅ 本檔對象：FMC 寫入（自動時序）

```c
// FMC 方式 ← NANDO 採用此法
*(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = 0x80;
*(__IO uint8_t *)(Bank_NAND_ADDR | DATA_AREA) = buf[i];  // FMC 自動產生 RE#/WE# 邊沿
```

**特點：**
- 硬體控制器（FMC）自動管理 WE#/RE# 時序
- 軟體只需寫指標，無需手動控制腳位
- 每 byte 寫入耗時 ~140ns（HCLK 週期級）
- STM32F1 / STM32H7 FMC 都支援

---

## Write Page 流程五步驟

### 第①步：發送寫入開始命令

```c
*(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.write1_cmd;
// fsmc_conf.write1_cmd = 0x80 (WRITE1 命令)
```

**功能：** 告訴晶片進入「接收資料模式」，準備抓住後續的位址與資料

**FMC 動作：**
- 軟體寫 `0x80` 到 `CMD_AREA` （A16 拉高）
- FMC 自動產生 WE# 脈衝（寫選訊號）
- 晶片鎖存命令 byte

---

### 第②步：發送列位址（Column Address）

```c
switch (fsmc_conf.col_cycles)
{
case 2:  // 2KB page 典型
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_1st_CYCLE(page_offset);  // col_lo
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_2nd_CYCLE(page_offset);  // col_hi
    break;
case 3:  // 4KB page
    // 三個週期
    break;
case 4:  // 大容量晶片
    // 四個週期
    break;
}
```

**地址編碼巨集：**
```c
#define ADDR_1st_CYCLE(ADDR)  (uint8_t)((ADDR) & 0xFF)
#define ADDR_2nd_CYCLE(ADDR)  (uint8_t)(((ADDR) & 0xFF00) >> 8)
#define ADDR_3rd_CYCLE(ADDR)  (uint8_t)(((ADDR) & 0xFF0000) >> 16)
#define ADDR_4th_CYCLE(ADDR)  (uint8_t)(((ADDR) & 0xFF000000) >> 24)
```

**功能：** 定位「在頁內的哪個位置開始寫」

**例子（2KB 頁）：**
- Column address = 0x0000 ~ 0x07FF (2048 bytes)
- 分成 2 個 byte 傳送：第一個 byte (bit 0-7)、第二個 byte (bit 8-11)

**FMC 動作：**
- 軟體寫 col_lo、col_hi 到 `ADDR_AREA` （A17 拉高）
- FMC 自動產生 ALE 脈衝與 WE# 邊沿
- 晶片鎖存每個位址 byte

---

### 第③步：發送行位址（Row Address）

```c
switch (fsmc_conf.row_cycles)
{
case 2:  // 小容量晶片
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_1st_CYCLE(page);
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_2nd_CYCLE(page);
    break;
case 3:  // 中容量
    // 三個週期
    break;
case 4:  // 大容量（如 8T24）
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_1st_CYCLE(page);
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_2nd_CYCLE(page);
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_3rd_CYCLE(page);
    *(__IO uint8_t *)(Bank_NAND_ADDR | ADDR_AREA) = ADDR_4th_CYCLE(page);
    break;
}
```

**功能：** 定位「是哪一頁、在哪一塊」

**例子（8T24 = 64GB）：**
- 需要 4 個位元組才能表達完整位址
- Page + Block 合併成一個 32-bit row 變數，逐 byte 拆開傳送

---

### 第④步：逐 byte 寫入資料

```c
for(i = 0; i < page_size; i++)
    *(__IO uint8_t *)(Bank_NAND_ADDR | DATA_AREA) = buf[i];
```

**流程：**
- buf[0] → buf[page_size-1]
- page_size 通常 2048 或 4096（不含 spare area）

**FMC 時序：**
- 軟體寫第 i 個 byte 到 `DATA_AREA`
- FMC 自動產生 WE# 脈衝
- 晶片 I/O buffer 接收並存入內部 SRAM

**總耗時：** page_size × 140ns（FMC 自動時序週期）
- 2KB = ~280µs
- 4KB = ~560µs

---

### 第⑤步：發送編程啟動命令

```c
if (fsmc_conf.write2_cmd != UNDEFINED_CMD)
    *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.write2_cmd;
// fsmc_conf.write2_cmd = 0x10 (WRITE2 / PROGRAM 命令)
```

**功能：** 「我的資料都傳完了，開始編程吧」

**晶片內部動作：**
1. 啟動 charge pump（高壓電路）
2. 產生編程電壓 ~19-21V（針對浮動柵極晶片編程）
3. 燒寫 buffer 中的資料到 NOR 陣列
4. 耗時 200µs ~ 3ms（取決於晶片和編程技術）

**⚠️ 關鍵：此命令返回後，晶片仍在編程狀態**
- 程式不會阻塞等待（"_async" 設計）
- 需後續呼叫 `nand_get_status()` polling

---

## FMC 記憶體分區詳解

### 位址線控制（A16 / A17）

```c
#define CMD_AREA   (uint32_t)(1<<16)   // A16 = CLE (Command Latch Enable)
#define ADDR_AREA  (uint32_t)(1<<17)   // A17 = ALE (Address Latch Enable)
#define DATA_AREA  ((uint32_t)0x00000000)

#define Bank2_NAND_ADDR  ((uint32_t)0x70000000)
```

**三種區域使用方式：**

| 區域 | 位址線 | 用途 | 例子 |
|---|---|---|---|
| **命令** | A16 拉高 | 寫命令 | `0x80` (WRITE1) / `0x10` (WRITE2) |
| **位址** | A17 拉高 | 寫位址 | Column / Row 位址 byte |
| **資料** | A16=A17=0 | 讀/寫資料 | `buf[i]` 資料 byte |

**FMC 自動對應：**
- 軟體寫 `CMD_AREA` → FMC 拉 A16、CLE 信號
- 軟體寫 `ADDR_AREA` → FMC 拉 A17、ALE 信號
- 軟體寫 `DATA_AREA` → FMC 不拉 CLE/ALE，純資料讀寫

---

## Status Polling（等待編程完成）

### 為什麼要 Polling？

```c
// ❌ 錯誤：寫完就認為成功
nand_write_page_async(buf, page, 2048);
printf("寫入成功");  // 錯！晶片還在編程

// ✅ 正確：等待編程完成
nand_write_page_async(buf, page, 2048);
status = nand_get_status();  // polling tPROG（200µs ~ 3ms）
if (status == FLASH_STATUS_READY)
    printf("寫入成功");
```

### Polling 實作

```c
static uint32_t nand_get_status()
{
    uint32_t status, timeout = 0x1000000;

    status = nand_read_status();

    while (status == FLASH_STATUS_BUSY && timeout)
    {
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

    *(__IO uint8_t *)(Bank_NAND_ADDR | CMD_AREA) = fsmc_conf.status_cmd;  // 0x70
    data = *(__IO uint8_t *)(Bank_NAND_ADDR);

    if ((data & NAND_ERROR) == NAND_ERROR)
        status = FLASH_STATUS_ERROR;      // bit 0 = 1 → FAIL
    else if ((data & NAND_READY) == NAND_READY)
        status = FLASH_STATUS_READY;      // bit 6 = 1 → Ready
    else
        status = FLASH_STATUS_BUSY;       // Still busy

    return status;
}
```

**Status 位元定義（ONFI 標準）：**
- **bit 0**：ERROR (1 = 編程失敗)
- **bit 6**：READY (1 = 就緒，編程完成)

---

## FMC 時序配置

### 必要的 FMC 暫存器設定

```c
typedef struct __attribute__((__packed__))
{
    uint8_t setup_time;        // Setup time (HCLK cycles)
    uint8_t wait_setup_time;   // Wait setup time
    uint8_t hold_setup_time;   // Hold setup time
    uint8_t hi_z_setup_time;   // Hi-Z time（讀資料前释放 WE#/OE# 延遲）
    uint8_t clr_setup_time;    // CLE setup time
    uint8_t ar_setup_time;     // ALE setup time
    // ...更多欄位
} fsmc_conf_t;
```

**典型配置（STM32F1 @ 72MHz HCLK）：**
- Setup Time: 3 cycles (~42ns) — WE# 建立時間
- Hold Time: 2 cycles (~28ns) — 地址保持時間
- Hi-Z Time: 1 cycle (~14ns) — 讀取前提前釋放驅動

---

## 寫入與 GPIO Bit-Bang 的對比

| 項目 | FMC 自動時序 | GPIO Bit-Bang |
|---|---|---|
| **控制方式** | FMC 硬體自動 | 軟體逐一設定腳位 |
| **寫入速度** | ~140ns/byte (HCLK 級) | ~11µs/byte (軟體迴圈) |
| **2KB 頁耗時** | ~280µs | ~22ms |
| **軟體複雜度** | 簡單（指標存取） | 複雜（GPIO BSRR 迴圈） |
| **時序精確度** | 由 FMC 控制器保證 | 受軟體干擾影響 |
| **可用場景** | 一般高效寫入 | Debug / 診斷 / 時序靈活調整 |

---

## 完整寫入流程示意圖

```
程式呼叫 nand_write_page_async()
│
├─ ① 寫 0x80 到 CMD_AREA
│    └─ FMC: WE# pulse, CLE high → 晶片鎖存命令
│
├─ ② 逐 byte 寫 Column Address 到 ADDR_AREA
│    └─ FMC: WE# pulse, ALE high → 晶片鎖存位址 byte
│
├─ ③ 逐 byte 寫 Row Address 到 ADDR_AREA
│    └─ FMC: WE# pulse, ALE high → 晶片鎖存位址 byte
│
├─ ④ 逐 byte 寫資料到 DATA_AREA
│    └─ FMC: WE# pulse, CLE=ALE=low → 晶片接收資料存 buffer
│         (2KB: 2048 次迴圈 × 140ns = ~280µs)
│
├─ ⑤ 寫 0x10 到 CMD_AREA
│    └─ FMC: WE# pulse, CLE high → 晶片啟動編程
│         (晶片內部：charge pump → 高壓 → NOR 編程)
│
└─ 函式返回（"_async" = 不等完成）
   │
   └─ 程式需要手動 polling nand_get_status()
      (等待 200µs ~ 3ms，直到 bit6=1)
```

---

## 注意事項

⚠️ **Column/Row cycles 數量要與晶片對齊**
- 少送位址 → 定位到錯誤位置
- 多送位址 → 晶片解釋錯誤（或被忽略）

⚠️ **page_size 不包含 spare area**
- 只寫 2048 bytes（data 區）
- Spare area (OOB) 用另一函式 `nand_write_spare_data()`

⚠️ **WRITE2 之後必須 polling status**
- 晶片編程時間不可預測（200µs ~ 3ms）
- 不能假設立即完成

⚠️ **Buffer 溢出 = 覆寫其他頁面**
- page_size 寫超過 → 資料散落到相鄰地址
- 驗證 page_size 與晶片規格一致

⚠️ **FMC 時序配置影響可靠性**
- Setup/Hold 太短 → 邊沿搶不到
- Hi-Z 不足 → 讀資料時舊 byte 仍在線上汙染新資料

---

## 參考資源

- **原始代碼**：`/home/dm/nand-flash/nand_programmer/firmware/programmer/fsmc_nand.c`
- **完整分析**：[`nand-open.md`](nand-open.md) §三
- **STM32H743 對照**：`/home/dm/stm32h743/memory/stm32_fmc_初始化與暫存器時序.md`
- **NAND 命令集**：`/home/dm/nand-flash/memory/nand_vendor_command.md`
