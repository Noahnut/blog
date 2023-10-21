---
title: 什麼是 Cache
tags:
  - backend
date: 2023-10-21 16:15:38
---


## Cache 概述

Cache 就是將之後要再使用的資料存在記憶體中，這樣就不用去很慢的非揮發性存儲空間拿取資料，這不止是在後端架構中很常見，OS 或 Database 中的 Buffer 其實都是 Cache 的一種，在這邊會專注在後端系統架構中的 Cache。後端的 Cache 常見的工具有，Application 的內存、Redis、Memcache。

## 什麼時候要用 Cache

1. 靜態網站資料，例如：不會變動的使用者資訊或者圖片等等
2. 需求從 DB 或者非揮發性存儲空間拿取的資料，且這些資料不太容易有變動

## Cache 的使用策略

Cache 最常遇到的就是要處理在 Cache 中找不到資料跟資料更新時 Cache 要怎麼處理一致性的這些問題，常見的讀寫策略有以下幾種，這些策略各有優缺點端看系統在設計時的需求考量。

### Read 策略

Read 相對單純，只需要處理 Cache Miss，但通常都不會有太大問題

#### Cache Aside

Cache 更新的處理主要是由 Application 來做，當出現 Cache Miss 的時候，Application 會跟 Database 拿新的資料並且更新 Cache。

##### Pros

Cache 中的值彈性很大，可以是自定義含商業邏輯的值

##### Cons

必須自行維護

#### Read Through

Cache 的更新由 Cache 自行處理，當出現 Cache Miss 的時候，Cache 會去跟 Database 拿資料在回傳給 Application。

##### Pros

開發者不需要管如何去更新 Cache

##### Cons

Cache 中的值彈性不大

### Write 策略

Write 就比較複雜，需要處理 Cache 跟 Database 之間可能會有短暫不一致的問題

#### Write Around

先刪除 Cache 中的值，再去更新 DB，避免其他 Request 從 Cache 拿到的值是舊的，更新完 DB 後，後續的 Request 會出現 Cache Miss，然後再去更新 Cache 中的值

#### Write Back

所有的更新都會先去更新 Cache，Cache 在背景定時的去將更新後的值 Update 到 Database，但如果 Cache 掛掉要更新到 DB 的值就會遺失。

#### Write Through

所有更新會先去更新 Cache，然後在馬上更新到 DB，這樣代表後續的 Request 必須要等到 DB 更新完後才能拿到新的值。

這邊附上一張 ByteByteGo 的圖，很清楚的解釋跟描述這些策略

![Alt text](cache_strategies.png)

## Cache 會遇到的問題

### Cache Avalanche

Cache 上大量的資料一起過期或者 Cache 壞掉重啟資料不見，導致大量的資料突然出現 Cache Miss，並將 Loading 壓在 Database 上，這樣就可能造成 Database 直接被打趴。

#### 解決方法

Cache Avalanche 發生時可以分成兩個問題，一個是資料同時過期，另外一個是 Cache 無法提供服務

##### 1. 資料同時過期

如果是資料同時過期，這部分就跟設計有關係，避免大量的資料都有相同的過期時間就可以避免這個問題

##### 2. Cache 無法提供服務

Cache 無法提供服務，如果要避免在 Cache 還沒起來或者 Cache 起來了但有大量 Cache Miss 造成 Cache Avalanche 問題，可以分成三個時間來處理。

* **事前預防** - 從根本解決，讓 Cache 有 HA 的機制，例如 Redis 的 Cluster，這樣就算掛掉一台還是有其他台能夠備援處理。
* **Cache 掛掉** - 直接限流避免 Database Loading 過重，等待 Cache 重新復活跟 Cache Miss 的 Hit 次數降低。
* **Cache Recovery** - 會出現 Cache Avalanche 的原因是有大量 Cache Miss，如果 Cache 再重新起來後能夠有將掛掉之前的資料自己補回來的機制，這樣就可以減少 Cache Miss 從而減少 Database 的 Loading，例如 Redis 的 AOF。

### Hotspot Invalid

如果有一個大量 Request 會使用的資料在 Cache 中過期，這也會造成在更新到 Cache 前，Database 會有大量的 Request，這也是有可能造成 Database 無法負擔流量。

#### 解決方法

* **只讓一個 Request 去 Database 拿新的資料** - 使用 Global Lock 或者類似的機制，只讓一個 Request 去從 Database 拿取新的資料並更新到 Cache 上，其他的 Request 則在 Cache 等待，等待到 Lock 釋放且有新資料。

### Cache Penetration

如果資料不存在 Cache 也不存在 DB 中，這樣可以讓有心人士透過這樣的機制，DDos Database，讓大量的流量打到 Database 上。

#### 解決方法

* **BloomFilter** - BloomFilter 是一個快速的 BitMap，可以低成本空間與高效率的知道這個值存不存在 Database 中，但 Bloom Filter 有 False Positive 的問題存在，就是 Bloom Filter 說存在但實際上不存在，這個部分可以靠調整 BloomFilter 的大小來降低 False Positive 的機率。


