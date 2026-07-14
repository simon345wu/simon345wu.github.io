+++
title = "如何在 Windows 用 C++ 內嵌驅動程式取得 AMD/Intel 實體 CPU 溫度（免安裝/優雅降級）"
date = 2026-07-14T12:10:00+08:00
lastmod = 2026-07-14T12:10:00+08:00
categories = ["技術分享"]
tags = ["C++17", "Windows API", "WinRing0", "硬體開發"]
+++

在開發輕量級桌面監控小工具 [win_hardware_monitor](https://github.com/simon345wu/win_hardware_monitor) 時，最棘手的需求之一就是**顯示 CPU 實體溫度**。

許多現代桌上型主機板（特別是華碩 ASUS 的 AMD B650 等消費級主機板）出於安全或簡化設計，並不會將 CPU 溫度登錄到作業系統的標準 ACPI 表中，這導致透過傳統 WMI 查詢（例如 `MSAcpi_ThermalZoneTemperature`）只會得到 `0x8004100c` (Not Supported) 的錯誤，根本讀不到數據。

為了讓小工具保持**「單一執行檔、免安裝、不依賴背景執行其他監控軟體（如 HWiNFO）」**的核心優勢，我選擇了使用開源的 **WinRing0** 驅動程式進行底層暫存器讀取，並設計了「內嵌加載與優雅降級」的系統架構。

這篇文章將完整記錄這個功能的技術實作細節。

---

## 🛠️ 系統架構設計：內嵌資源與優雅降級

WinRing0 是一個允許使用者模式應用程式直接存取 I/O 連接埠、MSR（Model-Specific Registers）與 PCI 配置空間的內核模式驅動程式（包含 `.sys` 驅動檔與 `.dll` 動態連結庫）。

為了免去使用者手動配置驅動的麻煩，我們採用了以下設計：

1. **資源封裝**：將 `WinRing0x64.dll` 與 `WinRing0x64.sys` 封裝在應用程式的 `.rc` 資源檔案中，編譯時直接打包進 `.exe` 內部。
2. **自動釋放**：程式啟動時，若同目錄下沒有這兩個檔案，則呼叫 Windows Resource APIs 將它們從資源段釋放至執行檔同目錄下。
3. **安全降級（Graceful Degradation）**：
   * **第一階段**：若程式以「系統管理員權限」執行，則載入 WinRing0 驅動，針對 CPU 品牌直接讀取底層暫存器的**真實實體溫度**（UI 上顯示為 **🟢 綠色**）。
   * **第二階段（WMI Fallback）**：若無管理員權限或載入驅動失敗，自動降級查詢 WMI 接口（適合部分支援 ACPI 溫度的筆電或 Intel OEM 主機）。
   * **第三階段（模擬 Fallback）**：若 WMI 也回報不支援，則利用 CPU 即時負載計算動態平滑曲線來模擬溫度（`37°C + cpuPct * 0.45`），並以 **🟡 橘黃色** 顯示，保證程式 100% 穩定且畫面不缺失。

---

## 💻 關鍵實作細節

### 1. 內嵌資源釋放與動態載入

我們在資源定義檔中宣告：
```rc
// resource_ids.h
#define IDR_WINRING0_DLL 101
#define IDR_WINRING0_SYS 102

// resources.rc
#include "resource_ids.h"
IDR_WINRING0_DLL RCDATA "../WinRing0x64.dll"
IDR_WINRING0_SYS RCDATA "../WinRing0x64.sys"
```

啟動時，先取得自身的絕對路徑並進行動態載入：
```cpp
// 資源釋放核心邏輯
bool ExtractResource(int resourceId, const std::wstring& outputPath) {
    HRSRC hRes = FindResourceW(NULL, MAKEINTRESOURCEW(resourceId), RT_RCDATA);
    if (!hRes) return false;
    HGLOBAL hData = LoadResource(NULL, hRes);
    LPVOID pData = LockResource(hData);
    DWORD size = SizeofResource(NULL, hRes);
    
    HANDLE hFile = CreateFileW(outputPath.c_str(), GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
    if (hFile == INVALID_HANDLE_VALUE) return false;
    
    DWORD written = 0;
    WriteFile(hFile, pData, size, &written, NULL);
    CloseHandle(hFile);
    return true;
}
```

> [!WARNING]
> **API 踩坑點：WinRing0 函式命名**
> 在透過 `GetProcAddress` 載入動態庫函數時，MSR 讀取的 API 函數在 64 位元 DLL 裡的實際匯出名稱為 **`Rdmsr`** 而非直覺的 `ReadMsr`。若拼寫錯誤會導致指標為 `NULL` 而加載失敗。

---

### 2. 讀取 AMD Ryzen (Zen 架構) 溫度

對於 AMD 處理器（包括我的 TUF B650 上的 Ryzen 7000 系列），實體溫度是透過 **System Management Network (SMN)** 進行讀取，而不是放在常見的 MSR 暫存器中。

SMN 的讀取採用**間接定址法**，需要透過 PCI 匯流排的 Bus 0, Device 0, Function 0（即 PCI 位置為 `0`）的 `0x60` 與 `0x64` 暫存器作為 Index/Data 對來存取：

1. **指定 SMN 位置**：寫入 `0x00059800`（這是 AMD Zen 架構的 `ZEN_REPORTED_TEMP_CTRL_BASE` 暫存器）至 PCI 偏移量 `0x60`（即 `SmnAddr`）。
2. **讀取資料**：從 PCI 偏移量 `0x64`（即 `SmnData`）讀取返回的 32 位元數值。
3. **數值換算**：溫度位於讀出數值的 `[31:21]` 位元（`val >> 21 & 0x7ff`），單位是 1/8 攝氏度。
4. **偏移量扣除**：Ryzen 處理器在 SMN 中回報的數值是 `Tctl`（控溫溫度），背後帶有特殊的校正偏移量（一般為 `49.0°C`）。我們必須將讀出的溫度減去 `49.0`，才能得到真實的核心 Die 溫度 (`Tdie`)。

```cpp
if (IsAmdCpu() && m_WritePciConfigDwordEx && m_ReadPciConfigDwordEx) {
    // 寫入暫存器位址
    if (m_WritePciConfigDwordEx(0, 0x60, 0x00059800)) {
        unsigned long val = 0;
        if (m_ReadPciConfigDwordEx(0, 0x64, &val)) {
            float temp = ((val >> 21) & 0x7ff) / 8.0f;
            if (temp > 49.0f) {
                temp -= 49.0f; // 扣除 49.0C 的 Tctl 偏移量
            }
            if (temp > 0.0f && temp < 200.0f) {
                cpuTemp = temp;
                isReal = true;
            }
        }
    }
}
```

---

### 3. 讀取 Intel DTS (數位感測器) 溫度

對於 Intel 處理器，溫度資料是標準的透過 MSR 暫存器來讀取：

1. **讀取溫度狀態**：存取 MSR `0x19C` (`IA32_THERM_STATUS`)，並確認第 31 位元（Valid Bit）為 `1`。
2. **獲取偏移量**：擷取第 `22:16` 位元，這代表當前核心溫度距離最高允許溫度（TjMax）的差值（Offset）。
3. **讀取溫度上限 (TjMax)**：存取 MSR `0x1A2` (`IA32_TEMPERATURE_TARGET`)，擷取第 `23:16` 位元即為 TjMax（通常為 100°C）。
4. **計算結果**：`Tdie = TjMax - Offset`。

```cpp
else if (IsIntelCpu() && m_Rdmsr) {
    unsigned long eax = 0, edx = 0;
    if (m_Rdmsr(0x19C, &eax, &edx)) {
        if (eax & 0x80000000) { // Valid bit
            unsigned long offset = (eax >> 16) & 0x7F;
            unsigned long tjMax = 100; // 預設值
            unsigned long eax2 = 0, edx2 = 0;
            if (m_Rdmsr(0x1A2, &eax2, &edx2)) {
                unsigned long val = (eax2 >> 16) & 0xFF;
                if (val > 0) tjMax = val;
            }
            float temp = (float)(tjMax - offset);
            if (temp > 0.0f && temp < 200.0f) {
                cpuTemp = temp;
                isReal = true;
            }
        }
    }
}
```

---

## 🎨 UI 色彩反饋與測試成果

為了讓使用者能直觀了解目前是處於實體讀取還是優雅降級模式，CPU 溫度數據被賦予了不同的顏色：

* **管理員身分執行 (🟢 綠色)**：WinRing0 順利載入。在桌面雙擊右鍵「以系統管理員身分執行」，小工具會在背景自動載入驅動，此時 CPU 溫度顯示為明亮的綠色，數值完美對應 CPU 的真實物理溫度，且程式同目錄下會自動釋放出 `WinRing0x64.dll` 與 `WinRing0x64.sys`。
* **一般雙擊執行 (🟡 橘黃色)**：沒有權限加載核心驅動。程式顯示為橘黃色的模擬估算溫度（依負載動態計算），程式運行依舊極度流暢穩定，適合非敏感工作下的日常監控。

本實作不僅成功解決了 ASUS 等高階主機板 ACPI 溫度丟失的棘手問題，更維持了單一執行檔的簡潔性與極低的運行開銷，為這款輕量化工具補上了最後一塊重要的硬體拼圖！
