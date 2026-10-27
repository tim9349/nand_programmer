# NANDO — 開源 NAND 程式寫入器 導航索引

> **本檔角色：** NANDO 專案寫入流程分析的快速導航（電話簿）。
> **詳細分析：** → `/home/dm/nand-flash/nand_programmer/memory/nand-open.md`（完整實作解析）
> **建檔日期：** 2026-08-25；2026-10-27 目錄由 `memory_dm/` 改名 `memory/`（對齊其他專案慣例），路徑與行號已對照原始碼核對。

**本目錄檔案：**

| 檔 | 角色 |
|---|---|
| `/home/dm/nand-flash/nand_programmer/memory/README.md` | NANDO 專案學習要點總覽與技術手冊（firmware／Qt 上位機／硬體等 4 個領域） |
| `/home/dm/nand-flash/nand_programmer/memory/nand-open.md` | 完整實作解析（寫／讀／擦／Status、與 STM32H743 對應） |
| `/home/dm/nand-flash/nand_programmer/memory/nand-open_硬體架構.md` | STM32F1 硬體與 FSMC 配置、GPIO 腳位、PCB |
| `/home/dm/nand-flash/nand_programmer/memory/fmc_write_page.md` | Write Page 五步驟詳解 + FMC 分區 + 時序配置 |

---

## 我要查什麼 → 進 nand-open.md 的哪一節

| 想了解 | 進哪一節 |
|---|---|
| **STM32 硬體與 FMC 配置** | → `/home/dm/nand-flash/nand_programmer/memory/nand-open_硬體架構.md`（STM32F1、GPIO 腳位、PCB 設計） |
| **NANDO 整體專案結構** | §一 專案結構概覽 |
| **韌體有哪些 C 檔案** | §二 韌體架構（firmware/programmer/） |
| **核心讀寫擦 API** | §二 關鍵 API（fsmc_nand.c） |
| **HAL 抽象層設計** | §二 暴露的 HAL 結構體 |
| **Page Program 時序與實作** | §三 NAND 寫入流程（Write Page 詳解） |
| **寫入的命令序列順序** | §三 時序結構（ONFI 標準） |
| **寫入代碼逐行解析** | §三 代碼實現（fsmc_nand.c L228-288） |
| **地址編碼怎麼分解** | §三 地址編碼巨集 |
| **FMC 記憶體怎麼分區** | §三 FMC 記憶體分區（CLE/ALE） |
| **讀資料流程** | §四 讀取流程（Read Data） |
| **Read ID 怎麼讀 5 byte** | §四 Read ID（L210-226） |
| **擦除塊流程** | §五 擦除流程（Erase Block） |
| **Status 怎麼 polling** | §六 狀態輪詢（Status Polling） |
| **NANDO vs STM32H743 對應** | §七 與 STM32H743 的對應參考 |
| **可直接借用的設計** | §七 直接可參考的部分 |
| **可移植的設計模式** | §七 可移植的設計模式 |
| **寫入時的常見陷阱** | §八 使用注意事項 |

---

## 核心實現快速參考

### Page Program（寫頁）— FMC 方式

**時序流程：**
1. CMD `0x80` (WRITE1)
2. Column Address (1-4 cycles)
3. Row Address (1-4 cycles)
4. Data (逐 byte)
5. CMD `0x10` (WRITE2)
6. Wait Status

**對應代碼：** `/home/dm/nand-flash/nand_programmer/firmware/programmer/fsmc_nand.c` L228-288

**實作方式：** FMC 硬體自動時序（指標存取）

**完整說明檔：** → `/home/dm/nand-flash/nand_programmer/memory/fmc_write_page.md`（五步驟詳解 + FMC 分區 + 時序配置）

**關鍵點：**
- `_async` 不等完成，後續需呼叫 `nand_get_status()` polling
- 定址週期數要與晶片規格對齊
- page_size 通常 2048 或 4096（不含 spare area）
- 寫入速度 ~140ns/byte（FMC 自動時序）

### Read Data（讀資料）

**時序流程：**
1. CMD `0x00` (READ1)
2. Column Address (1-4 cycles)
3. Row Address (1-4 cycles)
4. Optional CMD `0x30` (READ2)
5. Read (逐 byte)
6. Wait Status

**對應代碼：** `/home/dm/nand-flash/nand_programmer/firmware/programmer/fsmc_nand.c` L290-361

### Read ID（讀晶片 ID）

**簽名：** `nand_read_id(chip_id_t *nand_id)` L210-226

**流程：**
- CMD `0x90`
- ADDR `0x00`
- 讀 5 bytes（maker_id, device_id, third_id, fourth_id, fifth_id）

### Erase Block（擦除塊）

**簽名：** `nand_erase_block(uint32_t page)` L441-473

**流程：**
1. CMD `0x60` (ERASE1)
2. Row Address (block 位址)
3. CMD `0xD0` (ERASE2)
4. Wait Status

### Status Polling

**核心函式：**
- `nand_get_status()`（`fsmc_nand.c` L191）— polling loop with timeout
- `nand_read_status()`（L174）— 讀 status register，判 ERROR/READY/BUSY

**Status 位元定義：**
- bit 0 = ERROR (1 = fail)
- bit 6 = READY (1 = ready)

---

## 架構設計亮點

✅ **FMC Bank2 @ 0x70000000**
- A16 = CLE (Command Latch Enable)
- A17 = ALE (Address Latch Enable)

✅ **HAL 函式指標抽象**（`flash_hal_t`）
- 便於多晶片支援
- PC 應用層 (`qt/`) 與韌體層分離

✅ **配置結構體**（`fsmc_conf_t`）
- 時序參數可配置
- 命令序列可配置（支援不同晶片指令集）

✅ **_async 設計**
- write 與 status 等待分開
- 便於多工或 DMA

✅ **Bad block 獨立層**（`/home/dm/nand-flash/nand_programmer/firmware/programmer/nand_bad_block.c`）
- 可複用於其他 NAND driver

---

## 與 STM32H743 的對應

| 項目 | NANDO | STM32H743 |
|---|---|---|
| **FMC Bank** | Bank2 @ 0x70000000 | Bank3 @ 0x80000000 |
| **寫入流程** | 同 ONFI 標準 | 可參考 |
| **Status polling** | 標準框架 | 可複用 |
| **ECC 支援** | `FSMC_ECC_Enable` | 可參考 |

**直接可借用的代碼段：**
1. `nand_write_page_async()` — Page Program 實作邏輯
2. `nand_read_data()` — Random Data Output 讀取
3. `nand_get_status()` — polling 框架與 timeout 機制
4. ADDR_*_CYCLE 巨集 — 地址分解

---

## 使用注意事項

⚠️ **`_async` 不代表真的非同步** — 必須後續 polling status

⚠️ **定址週期要對齊晶片規格** — 少/多送位址會定位到完全不同位置

⚠️ **page_size 不包含 spare area** — 用 `nand_read_spare_data()` 分別讀

⚠️ **Status 定義要看晶片規格** — bit 0/bit 6 是 ONFI 標準，廠商可能不同

⚠️ **專案無長期維護** — 僅用於學習/對照，不整合進 production firmware

---

## 參考資源

- **完整實作分析**：`/home/dm/nand-flash/nand_programmer/memory/nand-open.md`
- **NANDO GitHub**：https://github.com/bbogush/nand_programmer
- **STM32H743 NAND 實作**：`/home/dm/stm32h743/stm32_fw/ok/index_fw.md`
- **NAND 通則**：`/home/dm/nand-flash/memory/nand_vendor_command.md` §0（ONFI 公開命令、Read ID、Status；晶片專屬 vendor 內容已分檔，入口 `/home/dm/nand-flash/memory/index_nand.md`）
