+++
title = "如何在 Windows 用 C++ 直接讀取 DDR5 記憶體溫度（SMBus + SPD5118 實戰）"
date = 2026-07-19T18:00:00+08:00
lastmod = 2026-07-19T18:00:00+08:00
categories = ["技術分享"]
tags = ["C++17", "Windows API", "WinRing0", "SMBus", "DDR5", "硬體開發"]
+++

在[上一篇文章](/posts/win-hardware-monitor-physical-cpu-temp/)中，我為輕量級桌面監控小工具 [win_hardware_monitor](https://github.com/simon345wu/win_hardware_monitor) 實作了 CPU 實體溫度讀取。這次的目標更進一步：**顯示記憶體（DIMM）的實體溫度**。

這件事在 DDR4 時代幾乎是「看緣分」——溫度感測器（TSOD）在 DDR4 模組上是選配，消費級記憶體大多根本沒裝。但 **DDR5 改變了遊戲規則**：JEDEC 規範（JESD300）要求每一條 DDR5 模組都必須搭載 **SPD5118 hub 晶片**，而這顆晶片**內建溫度感測器**（精度 ±0.5°C、解析度 0.25°C）。換句話說，只要你用的是 DDR5，溫度數據就一定在那裡，問題只剩下——**怎麼把它讀出來**。

麻煩的是，Windows 完全沒有提供讀取 DIMM 溫度的 API，WMI 也查不到。感測器掛在主機板的 **SMBus（System Management Bus）** 上，想拿到數據就必須直接操作晶片組的 SMBus 控制器——這正好是 WinRing0 驅動的 I/O port 存取能力可以做到的事。

這篇文章記錄在 AMD AM5 平台（Ryzen 5 7500F + ADATA XPG DDR5）上的完整實作。

---

## 🗺️ 讀取路徑總覽

整條讀取鏈是這樣的：

```
應用程式 → WinRing0 (I/O port 讀寫) → AMD FCH SMBus 控制器 → SPD5118 hub → 溫度暫存器 MR49/MR50
```

拆解成四個步驟：

1. **找到 SMBus 控制器的 I/O base**：AMD FCH（晶片組）的 SMBus 主控制器位址記錄在 PM 暫存器中，透過 index/data port `0xCD6`/`0xCD7` 讀出（通常是 `0xB00`）。
2. **實作 SMBus 讀取協定**：AMD FCH 的 SMBus 控制器沿用經典的 PIIX4 相容暫存器佈局，用 port I/O 就能發起一筆 "read byte data" 交易。
3. **掃描 DIMM**：SPD 裝置固定佔用 SMBus 位址 `0x50`–`0x57`（對應 8 個插槽），逐一探測並用裝置識別碼確認是 SPD5118。
4. **讀取並解碼溫度**：讀 MR49/MR50 兩個暫存器，按 JESD300 規格解碼出攝氏溫度。

---

## 💻 關鍵實作細節

### 1. 從 FCH PM 暫存器找到 SMBus base

AMD FCH 的 SMBus 控制器是 PCI 裝置 Bus 0 / Device `0x14` / Function 0（Vendor ID `0x1022`）。先用 PCI 配置空間確認裝置存在，再從 PM 暫存器讀出 I/O base：

```cpp
// FCH SMBus controller is at PCI bus 0, device 0x14, function 0 (AMD 0x1022).
unsigned long id = 0;
if (!m_ReadPciConfigDwordEx(0x14 << 3, 0x00, &id)) return;
if ((id & 0xFFFF) != 0x1022) return;

// SMBus I/O base from the FCH PM registers (index/data ports 0xCD6/0xCD7):
// PM reg 0x00 bit 4 = SMBus decode enabled, PM reg 0x01 = base >> 8.
m_WriteIoPortByte(0xCD6, 0x00);
unsigned char lo = m_ReadIoPortByte(0xCD7);
m_WriteIoPortByte(0xCD6, 0x01);
unsigned char hi = m_ReadIoPortByte(0xCD7);
unsigned short base = 0;
if (lo & 0x10) base = (unsigned short)hi << 8;
if (base == 0 || base == 0xFF00) base = 0xB00; // common default on AM4/AM5
if (m_ReadIoPortByte(base) == 0xFF) return;    // nothing decodes here
m_smbusBase = base;
```

這個做法參考了 Linux 核心 `i2c-piix4` 驅動對 SB800 之後 AMD 晶片組的處理方式；讀出來若無效則退回常見預設值 `0xB00`，最後用 status 暫存器做一次 sanity check（讀到 `0xFF` 代表這個位址根本沒有裝置在解碼）。

### 2. PIIX4 相容的 SMBus 讀取協定

SMBus 主控制器的暫存器佈局（相對於 base）：offset 0 是狀態、2 是控制、3 是命令（目標暫存器編號）、4 是從裝置位址、5 是資料。一筆 "read byte data" 交易的流程是：等待匯流排空閒 → 清除舊狀態 → 填入位址與命令 → 下達開始 → 輪詢完成：

```cpp
bool SystemMonitor::SmbusReadByte(unsigned char devAddr, unsigned char reg, unsigned char& value) {
    const unsigned short base = m_smbusBase;
    if (base == 0) return false;

    // Wait out any in-flight transaction, then clear stale status bits.
    int spin = 0;
    while ((m_ReadIoPortByte(base) & 0x01) && ++spin < 400) {}
    if (spin >= 400) return false;
    m_WriteIoPortByte(base, 0x1F);

    m_WriteIoPortByte(base + 4, (unsigned char)((devAddr << 1) | 1)); // slave addr, read
    m_WriteIoPortByte(base + 3, reg);
    m_WriteIoPortByte(base + 2, 0x48); // start | byte-data protocol

    for (int i = 0; i < 2000; ++i) {
        unsigned char st = m_ReadIoPortByte(base);
        if (st & 0x1C) {               // device error / bus collision / failed
            m_WriteIoPortByte(base, 0x1F);
            return false;
        }
        if (st & 0x02) {               // transaction complete
            value = m_ReadIoPortByte(base + 5);
            m_WriteIoPortByte(base, 0x02);
            return true;
        }
    }
    return false;
}
```

> [!WARNING]
> **SMBus 是共享匯流排，搶奪會出亂子**
> RGB 燈效軟體（控制記憶體燈條就是走 SMBus！）、其他監控工具（HWiNFO、LibreHardwareMonitor）都可能同時操作這條匯流排。兩個程式的交易互相交錯，輕則讀到亂數、重則讓 RGB 軟體當機。業界慣例是操作前先取得具名互斥鎖 **`Global\Access_SMBUS.HTP.Method`**——所有主流監控軟體都遵守這個協議。我的實作在每輪取樣前以 50ms timeout 搶鎖，搶不到就跳過這次取樣、沿用上次數值，畫面不會缺格。

> [!DANGER]
> **只讀，絕對不寫**
> SPD 存的是記憶體的時序參數（頻率、電壓、CL 值），主機板開機時就靠它初始化記憶體。**寫壞 SPD 會讓那條記憶體直接無法開機**。因此整個實作只使用 SMBus read 命令，從頭到尾沒有任何一筆對 SPD 裝置的寫入，最壞情況就只是「讀取失敗」。

### 3. 掃描並確認 SPD5118 裝置

SPD 裝置固定佔用 SMBus 位址 `0x50`–`0x57`。但「該位址有東西回應」不代表它是 SPD5118（DDR4 的 SPD EEPROM 也用同一段位址），所以要讀 MR0/MR1 裝置識別碼確認——SPD5118 固定回應 `0x51 0x18`（很好記，就是晶片名字）：

```cpp
// Probe all 8 SPD slots for SPD5118 hubs (MR0/MR1 = device type 0x51 0x18).
for (unsigned char addr = 0x50; addr <= 0x57; ++addr) {
    unsigned char mr0 = 0, mr1 = 0;
    if (SmbusReadByte(addr, 0x00, mr0) && SmbusReadByte(addr, 0x01, mr1) &&
        mr0 == 0x51 && mr1 == 0x18) {
        m_dimmAddrs[m_dimmCount++] = addr;
    }
}
```

掃描只在程式啟動時做一次；之後每秒的取樣只對已確認的位址讀溫度。

### 4. 解碼 MR49/MR50 溫度值

溫度放在 MR49（低位元組）與 MR50（高位元組），格式是 JESD300 定義的 **11-bit 二補數，落在 16-bit 字的 [12:2] 位元，每 LSB 代表 0.25°C**：

```cpp
bool SystemMonitor::ReadDimmTemp(unsigned char devAddr, float& tempC) {
    unsigned char lo = 0, hi = 0;
    if (!SmbusReadByte(devAddr, 49, lo) || !SmbusReadByte(devAddr, 50, hi))
        return false;
    int raw = (((int)hi << 8) | lo) >> 2;
    if (raw & 0x400) raw -= 0x800;      // 11-bit two's complement sign extend
    tempC = raw * 0.25f;
    return tempC > -25.0f && tempC < 125.0f;
}
```

> [!WARNING]
> **別忘了右移 2 位**
> 溫度值不是直接放在字的最低位——bit 0/1 是保留位。組完 16-bit 字之後要先 `>> 2` 再做符號延伸，否則讀出來的溫度會是實際值的 4 倍。這個格式與 Linux 核心 `spd5118` hwmon 驅動的解碼方式一致，可以互相對照驗證。

---

## 🎨 UI 設計決策：讀不到就不顯示

CPU 溫度有「負載模擬」的降級路徑，但記憶體溫度我刻意**不做模擬值**：

* **讀得到（🟢 綠色）**：MEM 面板 header 顯示兩條 DIMM 中**最熱一條**的實溫。取最大值而非平均，因為監控的目的是抓異常，最熱的那條才是需要注意的。
* **讀不到（直接隱藏）**：Intel 平台、DDR4、或沒有管理員權限時，溫度欄位整個不出現。記憶體溫度和負載沒有穩定的對應關係，模擬值只會誤導使用者，不如誠實地不顯示。

實測在 Ryzen 5 7500F + ADATA XPG Lancer DDR5-6400（雙通道兩條）上，啟動即正確偵測到兩條 DIMM，待機溫度約 35–40°C，與 HWiNFO 的讀值一致，每秒取樣的額外開銷小到量不出來（每輪只有 4 筆 SMBus byte 交易）。

## 📌 目前的限制

* **僅支援 AMD 平台**：Intel PCH 的 SMBus 控制器掛在不同的 PCI 位置（Bus 0, Device 31, Function 4）、base 的取得方式也不同，需要另外實作。
* **僅支援 DDR5**：DDR4 的 TSOD 感測器（若有）用的是不同的裝置位址段（`0x18`–`0x1F`）與暫存器格式，且消費級 DDR4 大多沒裝感測器，投報率太低就先不做了。

從 CPU 溫度到記憶體溫度，這個小工具的「硬體拼圖」又補上了一塊——而且依然維持單一執行檔、免安裝、極低開銷的初衷。完整程式碼在 [GitHub](https://github.com/simon345wu/win_hardware_monitor) 上，歡迎參考。
