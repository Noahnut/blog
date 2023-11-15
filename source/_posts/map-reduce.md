---
title: 論文閱讀： MapReduce Simplified Data Processing on Larger Clusters
tags: paper
date: 2023-11-16 00:09:23
---



## 介紹

MapReduce 是 Google 所提出的一個 Programming Model，目標是要實現簡單的介面讓工程師就能夠將複雜的將大量資料分散到不同機器運算。
MapReduce 其實就是利用 Divide and Conquer 的概念將大資料切分成小資料後再分別處理後在合再一起，透過這樣的方式可以讓這些被切分的小資料能夠被分散到不同機器上執行。

## MapReduce 的 Programming Model

MapReduce 的 Programming Model 由 `Map` 和 `Reduce` 這兩個行為所組成，將傳入的 Key Value 資料經過這兩個行為後得出結果

### Map (Divide)

由使用者自定義的行為，會將 Key Value 的資料根據相同的 Key 或其他邏輯將這些資料組成有相同類型資料的 `intermediate` 資料。
Map 這邊在定義的時候必須要考慮**資料計算的幂等性**，相同的資料不管計算幾次出來的 `intermediate` 資料應該都要是一樣，原因為 MapReduce 是一個分散式的系統架構，如果機器失效導致當下的這個 `Map` 失敗的話就必須讓其他機器來執行，當沒有這個幂等性時就會導致結果錯誤。由 `Map` 計算出來的 `intermediate` 資料就會送出 `Reduce` 做下一階段的處理。

### Reduce (Conquer)

`Reduce` 會收集來自不同機器的 `Map` 所產生的 `intermediate` 資料，就透過其 Key，將這些資料透過自訂的邏輯合併再一起，這個在定義的也是要考慮**資料計算的幂等性**。

其實 MapReduce 的 Programming Model 就是 Divide and Conquer 的概念，透過資料的切分讓資料能夠分散到不同機器上。


## MapReduce 的執行流程

MapReduce 的分散式設計是基於 `Master-Worker` 的模型去實作的，也就是會有一個主要角色去做工作上的分派與協調。

### 名詞解釋

**Master**:

MapReduce 中做 `Map` 與 `Reduce` 的工作分派、通知與 Fault Tolerance 的角色，負責控制全場的 Worker。


**Worker**:

接收來自 Master 的工作分派的角色


### 執行流程

![Alt text](flow.png)

MapReduce 的主要執行流程會是上圖的方式，下面會詳細介紹在論文中提到這些流程步驟與需要注意的部分。

**前置動作**

前面有提到 MapReduce 是將切片好的資料分給不同 `Worker` 所以就需要先將資料做個切塊的前置動作，論文中提到切片的大小通常介於 16MB 到 64MB 之間
這個部分是由使用者自行去決定的。

**Step One**

`Master` 會分派 M 個 `Map` 跟 R 個 `Reduce` 到工作給目前閒置的 `Worker`，而拿到 `Map` 的 `Worker` 會從被切塊中的資料取若干個進行計算。
而 M 跟 R 的大小會分別是多大？還有 `Map` 的 `Worker` 的切塊資料取多少會比較合理？

1. **M 跟 R 的大小會分別是多大**
   
    `Map` 的工作是取得切塊的資料做計算，如果 M 太小，會導致取的資料大小變大這樣如果出現問題重做，損耗就會很大，而 `Reduce` 會希望根據客戶需求產生出較少數量的文件，所以 Google 本身常用的數字會是 `M = 200,000`，`R = 5,000` 在 `2000` 個 worker 上。

2. **`Map` 的 `Worker` 的切塊資料取多少會比較合理**
    通常是 16MB 到 64MB 之間，原因應該是 `Map` worker 如果出現問題，在 MapReduce 的機制是直接重做，如果過大的資料會導致重做的成本太高。

**Step Two**

`Map` 計算完後會產生出 `intermediate Key Value` 的資料，這些資料都會先暫存在 `Worker` 的 Memory 與 Local Disk，`Worker` 完成後會通知 `Master` 已經完成 `Map` 任務。

**Step Three**

`Master` 會通知目前沒任務執行的 `Reduce`，有哪些 `Map` 的 `Worker` 是完成的，並且去從這些 `Worker` 中拿取 `intermediate Key Value` 的資料並做運算，而運算出來的資料會放在 Global 的檔案系統 (HDFS 或者是 GFS)。

**Step Four**

如果 `Reduce` 出來的資料有多份，就會持續進行上述步驟直到滿足顧客需求。

## MapReduce 的 Fault Tolerance

MapReduce 是將工作給分攤給多台不同的機器，這樣就會需要處理機器壞掉時的情況。

### Worker Failure

`Master` 會持續的 Ping `Worker` 監控各個 `Worker` 目前的生存狀況，如果 `Worker` 在一段時間內沒有回覆 `Master` 就會根據這個 `Worker` 的工作內容執行錯誤處理。
Worker 如果因為機器掛掉或者出現網路問題，MapReduce 這邊的機制就是直接重新將任務交由其他 Worker 重做，但是針對已完成但是出現錯誤的 Worker 根據任務會有不一樣的處理方式，如果是 `Map` 因為資料儲存在本地端所以是需要重做，但如果是 `Reduce` 任務因為是存在 Global File System，所以不需要重做。

而 `Master` 針對不同任務的 Worker，在處理重複計算的資料有不同的做法，`Map` 因為有資料的幂等性所以不管誰來做都可以，這樣 Master 如果收到不同 Worker 完成相同的 `Map` 任務就會無視下一個 Worker 的完成回傳，而如果是 `Reduce` 相同的任務會產生相同的檔案名稱，這樣下一個就只是覆蓋掉相同檔案也維持結果的幂等性。


### Master Failure

在論文中，針對 `Master` Failure 的做法就是單純告知用戶執行失敗並請用戶重新執行。

### Backup Task

在論文中提到 MapReduce 常常在最後階段時會遇到某些 Worker 因為硬體因素執行過久導致實際完成時間拉長，為了解決這個問題這邊當執行到最後的時候會讓其他空閒的 Worker 也去執行相同任務避免等待過久。

## 結論

MapReduce 提供一個非常簡單的介面讓使用者可以平行的計算資料，雖然這邊的做法是用很單純的 `Master Worker` 的 Model，但也提供許多有趣的思路，利用簡單的方法去簡化問題。
