---
title: PostgresML 介紹
date: 2023-10-03 00:06:10
tags: 
- postgres
- backend
- machine learning
---

## 前文
最近 CMU Database group 又開始新一輪的資料庫相關的 Seminar，這一次看起來都搭上潮流，幾乎都是跟 AI 有關，
剛好看到一個演講叫做 `PostgresML: Why Moving the Compute to the Data is Better than the Alternative`，
讓我非常好奇，是怎麼把計算搬移到資料上？不過我對於 ML 完全不熟，所以以下的內容都會是我以後端工程師的觀點的想法。


## 傳統微服務在機器學習上遇到什麼問題
在演講中 Montana 提到傳統的微服務雖然能夠解決 Scalability 的問題，但是也會增加服務跟服務之間傳輸的網路延遲，另外切成不同微服務也有可能讓不同團隊根據其需求選擇適合的 Database，這樣序列化跟反序列化就也是個效能上的消耗。


## PostgresML 解決什麼樣的痛點
1. **複雜的微服務架構帶來效能損耗**
    ML 在實際的應用上就是預測，越即時的預測越能夠帶來更好的效益，但是微服務的壞處讓 ML 很難做到即時的預測，所以 PostgresML 想要將服務簡單化，透過 Postgres 的 Connection Pool (PgCat) 與 Sharding，將資料庫的水平擴展藏在系統後面。

![PostgresML](arch.png)

2. **降低記憶體用量**
   PostgresML 是 Postgres 的套件，他們是共享記憶體，訓練跟預測都可以在 Postgres 上完成，所以重疊的內容是不需要重新分配記憶體。


## 結論
雖然 ML 跟 Backend 領域不同，但是看起來 ML 如果要達到想要的效果也是需要 Backend 的優化，而 Backend 在這部分也會遇到很大的挑戰，畢竟要如何及時的處理大量資料不是一件簡單的事情
