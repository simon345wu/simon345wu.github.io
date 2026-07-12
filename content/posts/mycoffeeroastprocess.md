---
title: 我的咖啡烘焙過程
date: 2026-07-13T07:05:41+08:00
draft: false
tags: []
categories: []
---

- [ ] 購買咖啡生豆  
          -我通常購買的來源。從網路的生豆商購買。針對這些生豆商我已經建立可定期更新查詢的資料庫。  
          1. 一個爬蟲系統，定期更新生豆資料的資料庫。  
          2. 資料庫建於 Neon Postgresql.   
          3. 查詢系統建於google cloud Firebase App host。Next.js 網站。  
  
- [ ] 咖啡烘培  
          1.  PC windows 使用 artisan 烘培軟體， 控制 skywalker roaster進行烘培，並存下烘培紀錄(.log)和曲線影像(.jpg)  
- [ ] 烘培紀錄的服務  
1.  使用 roastwise 網站服務 :( firebase app host Next.js 網站)  
          - 建生豆庫存  
           - 建立烘培紀錄 : 上傳 artisan log 和 jpg  
           - 產生可查詢網頁    
           - 可列印的豆袋標籤(pdf)供下載，標籤上網頁的QR code。  
- [ ] 咖啡標籤的列印  
1.  在pc 使用 pdfmerge python 程式，進行標籤合併，每張標籤為一頁然後acrobat PDF列印的時候設定為合併頁面。 每張A4紙可以列印9張標籤(3X3)。