+++
title = "🎨 Windows Resource Monitor Widget"
date = 2026-06-28T07:00:00+08:00
lastmod = 2026-07-14T00:00:00+08:00
categories = ["設計與其他作品"]
tags = ["C++17 ", "DirectX 11 ",]
+++

*   **專案連結**：[Github Simon345wu](https://github.com/simon345wu/win_hardware_monitor)
*   **專案簡介**：一個常駐桌面右上角的輕量級 Windows 即時資源監控小工具，目的是讓人一眼看出電腦現在有沒有在工作。使用 C++17 開發，結合 Dear ImGui、ImPlot 與 DirectX 11，以四條折線圖即時呈現 CPU、記憶體、磁碟讀寫與網路流量。無邊框置頂視窗、支援高 DPI，並透過閒置節流渲染與低成本系統 API 採樣，將自身開銷壓到整機 CPU 不到 0.1%、單一執行檔不到 500 KB。
*   **開發歷程**：v1 為多分頁的完整監控介面（逐核 CPU 網格、重疊曲線、記憶體詳情）；v2 重新定位為「掛在桌面角落的小工具」，只保留最能反映系統活動的四項指標，換取更小的體積與更低的資源佔用。
