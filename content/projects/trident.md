+++
title = "🔱 Trident — Skywalker 烘豆機三合一控制器"
date = 2026-09-15T10:00:00+08:00
categories = ["設計與其他作品"]
tags = ["ESP32-S3", "C++", "LVGL", "PlatformIO", "Artisan", "BLE"]
+++

*   **Github 專案**：[simon345wu/SkywalkerRoasterLab](https://github.com/simon345wu/SkywalkerRoasterLab)（fork 自 [jmoore52/SkywalkerRoaster](https://github.com/jmoore52/SkywalkerRoaster)）
*   **專案簡介**：把 **Skywalker V1 咖啡烘豆機**接上 **Artisan** 烘豆軟體的 ESP32-S3 控制器韌體。名稱「Trident(三叉戟)」取自它同時提供 **USB / BLE / WiFi** 三種連線:USB 走 TC4 序列協定、BLE 相容 HiBean、WiFi 走 WebSocket,三者可分別(甚至同時)連上 Artisan 或 HiBean 監看與控制烘焙。以 C++ / PlatformIO 開發,搭配 LVGL 觸控螢幕。
*   **主要功能**：
    *   **三種控制介面**：USB(TC4 文字協定)、BLE(NimBLE,相容 HiBean「Comm」模式)、WiFi(Artisan WebSocket 裝置)。
    *   **硬體級烘豆機通訊**：把與烘豆機之間的 RX/TX 從 bit-banging 移到 ESP32 **RMT 硬體**,脈波由硬體計時,不受 WiFi/BLE 中斷抖動影響。
    *   **LVGL 觸控介面**：splash / 主控 / 設定三個畫面;即時 BT / ET / 升溫速率(RoR)數字磚、火力/風力滑桿、滾筒/冷卻/STOP、WiFi/WS/BLE/USB 連線狀態燈、亮/暗主題。
    *   **外接 ET(排氣)溫度探針**：MAX31865 + 四線 PT100;自寫暫存器級驅動(避開 Adafruit 函式庫每次取樣阻塞 ~75ms 會卡住畫面的問題),中位數去突波 + EMA 兩段平滑,平滑度可由 Artisan 的 `FILT` 指令即時調整。
    *   **PID 火力控制**;RoR 演算法對齊 Artisan 本身的 `compute_ror_simple()`,BT/ET 用同一套計算。
    *   **WiFi 設定與診斷**：首次開機開 AP、mDNS(`trident.local`)、`/api/wifi` 儲存認證;WebSerial 分類日誌(`LOG;<類別>;ON|OFF`)。
    *   **環境天氣**：結合 [weather-proxy](/projects/weather-proxy/) 專案,把當下氣溫/氣壓/濕度自動帶進 Artisan 烘焙屬性並顯示在螢幕上。
*   **硬體**：ESP32-S3 DevKitC-1 N16R8(16MB flash + 8MB PSRAM)、ILI9341 / ST7789 SPI 觸控螢幕(XPT2046 觸控)、MAX31865 + PT100 探針。
