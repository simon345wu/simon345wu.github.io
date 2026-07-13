# 咖啡烘焙流程整合計畫（討論紀錄 2026-07-13）

## 背景：四個系統

| 系統 | 技術 | 角色 |
|------|------|------|
| taiwan_greenbeans 比價爬蟲 | Python 爬蟲 + Neon Postgres + Next.js (Firebase App Hosting) | 台灣生豆商「市場目錄」(catalog) |
| RoastWise | Next.js + Node.js + **Firestore** (Firebase App Hosting) | 個人庫存 + 烘焙紀錄 + 豆袋標籤 |
| Artisan + Skywalker | Windows PC，輸出 .alog(.log) + .jpg | 烘焙（維持人工操作，不交給 AI Agent） |
| pdfmerge | Python 腳本 + Acrobat 列印 | 標籤 3x3 合併列印 |

## 核心決策

1. **關聯方式：引用 + 快照**。庫存記錄存 `productId` 指向爬蟲 DB，同時複製描述性欄位快照。目錄會變動（下架/改名/改價），庫存是歷史事實不能跟著壞。實付價格存庫存端（目錄價是市價非成本）。
2. **自然鍵備援**：`productRef` 同時存 `supplier + productUrl`。爬蟲 ID 穩定性（upsert vs 重建）尚未處理，靠 URL 可日後重新對回。
3. **該埋進 Artisan log 的是 roastwise 的 inventory_id**（哪包豆被烘掉、扣庫存），不是爬蟲的 product_id。
4. `productRef` 可為 null（目錄外的豆子：國外訂購、朋友分豆）。
5. 舊的手動輸入庫存：等 API 好後寫一次性腳本模糊比對（品名+供應商），配不上就留空。

## Firestore 庫存文件結構（新增欄位，不動既有資料）

```
inventory/{inventoryId}
  productRef:                # 可為 null
    productId: "..."         # Neon 主鍵
    supplier: "豆超"
    productUrl: "https://..."  # 自然鍵備援
  snapshot:                  # 建檔當下複製，之後不同步
    name, origin, farm, variety, process, altitude
  purchase:
    date, pricePaid, weightKg, remainingKg
```

## 爬蟲網站要加的 API（唯讀 + API key）

```
GET /api/products/:id        # 單筆商品
GET /api/products?q=關鍵字   # 搜尋
```

## Artisan 銜接方式（已研究）

- Artisan 的 Roast → Properties 的 **Beans 欄位**（自由文字）會存進 .alog 的 `beans` 欄位。
- .alog 是 Python dict 純文字檔（可 `ast.literal_eval` 解析）；roastwise 已有 JS 版 alog 解析器（node server 端），需改寫加 `[bean_id: xxx]` regex 抽取。
- 烘焙前：roastwise 庫存頁「複製到 Artisan」按鈕 → 產生含 `[bean_id: inventory_id]` 的格式化文字 → 貼進 Beans 欄位。
- 上傳 .alog 後 roastwise 解析 ID → 自動關聯庫存 + 扣庫存。
- 參考：artisan.plus 的庫存下拉互動設計；Config → Autosave 可把 beans 帶進檔名。

## 施工順序（每步獨立可用）

1. 爬蟲網站加兩支唯讀 API
2. roastwise 建檔表單支援 `?product_id=xxx` 預填 + 存 productRef/snapshot
3. 比價網商品列表加「加入 roastwise 庫存」連結
4. roastwise「複製到 Artisan」按鈕 + alog 解析器加 regex ← 做完此步閉環即通
5. 爬蟲 upsert / 穩定 ID（有快照+URL 備援，不擋前面各步）
6. （一次性）舊庫存模糊比對補 productRef

## 完整資料流

```
比價網商品(product_id) → 進貨建 roastwise 庫存(inventory_id, 含 productRef+snapshot)
→ 烘焙前複製豆資訊(含 [bean_id]) → 貼 Artisan Beans 欄位
→ .alog 存檔 → 上傳 roastwise → 解析 bean_id → 自動關聯 + 扣庫存 → 標籤 PDF → pdfmerge 列印
```

## 其他已解決事項（同日）

- Hugo 需要 Dart Sass（FixIt 主題強制 dartsass），Windows 用 scoop install sass；勿用 npm 版。
- hugo.toml 已加 `timeZone = 'Asia/Taipei'`；文章日期建議帶完整時區。
- Obsidian 模板資料夾設定已改為 `template`（原誤設 content/template）；已建 template/Post.md。
