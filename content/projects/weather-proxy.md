+++
title = "☁️ 烘豆環境天氣自動記錄(weather-proxy)"
date = 2026-09-15T09:00:00+08:00
categories = ["網頁與軟體開發"]
tags = ["Python", "aiohttp", "ESP32", "mDNS", "Open-Meteo"]
+++

*   **Github 專案**：[simon345wu/weather-proxy](https://github.com/simon345wu/weather-proxy)
*   **相關韌體**：[Trident（SkywalkerRoasterLab fork）](https://github.com/simon345wu/SkywalkerRoasterLab) — branch `ambient-weather-http`
*   **專案簡介**：讓 **Artisan** 烘豆軟體的烘焙屬性(Roast Properties)自動帶上當下的**環境氣溫、大氣壓力、相對濕度**,不必手動查、手動填。資料依所在城市自 [Open-Meteo](https://open-meteo.com/) 取得,經 **Trident**(驅動 Skywalker V1 烘豆機的 ESP32-S3 控制器)透過 WebSocket 送進 Artisan,同時顯示在 Trident 的螢幕上。
*   **架構重點**：
    *   **weather-proxy(PC,Python / aiohttp)**：向 Open-Meteo 走 HTTPS 抓取並快取,對外提供純 HTTP 端點 `/api/preview` 與城市選取網頁;以 zeroconf 廣播 `_artisanwx._tcp` mDNS 服務。
    *   **為何需要一個 PC 代打**：Trident 這顆 ESP32-S3 同時跑 BLE + WiFi + LVGL 螢幕 + AsyncWebServer,內部 RAM 只剩約 17 KB,做不了需要 16–40 KB 連續記憶體的 TLS 握手(硬做會把整個 WebServer 一起搞垮)。所以把 HTTPS 移到 PC,Trident 只用純 HTTP(幾 KB)拉取。
    *   **mDNS 自動探索**：Trident 用 mDNS 自動找到 PC 目前的 IP,免手動設定;PC 換 IP 時抓取失敗會自動重新探索、自癒。
*   **開發歷程**：
    *   **v1** — Trident 韌體內建 HTTPS 直接抓 Open-Meteo、並在機上設定城市。實測發現 ESP32 記憶體不足以做 TLS:抓取間歇失敗,還把 WebSerial / WebServer 一起餓死。
    *   **v2** — 改為「PC proxy + 純 HTTP」架構:HTTPS 與城市設定移到 PC,Trident 只純 HTTP 拉取,記憶體問題根除。
    *   **v3** — 加上 **mDNS 自動探索**(PC 端 zeroconf ↔ Trident 端 ESPmDNS),連 PC 的 IP 都不用填,換網路也能自動接回。
