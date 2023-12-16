---
title: 論文閱讀：Spanner：Google’s Globally-Distributed Database
tags: paper
date: 2023-12-16 20:36:14
---

> 原文: [Spanner: Google’s Globally-Distributed Database](https://www.usenix.org/system/files/conference/osdi12/osdi12-final-16.pdf)


## 前言

Spanner 是 Google 的一套支援 Relational Database 的 Schema 限制、 SQL Query 語法操作、Transaction 的分散式資料庫。
Spanner 實現傳統 Relational Database 中難以處理的資料 Externally-Consistent (外部一致性) transaction 還有自動 Sharding 的功能。

## Spanner 的架構

### 最上層架構
Spanner 的一個完整架構會是由好幾個 `Zone` 所組成，而這些 `Zone` 底下可能有若干的 `Data Center`，而這些 `Zone` 會由 `univermaster` 與 `placement driver` 這兩個服務所管理，`univermaster` 文章中提到是類似監控的角色，可以讓維護人員透過 `univermaster` 知道目前所有 `Zone` 的狀態，而 `placement driver` 則是作為 `Zone` 之間資料的 `Rebalance` 的角色。 

### Zone 的細節

分散式資料庫最忌諱就是雞蛋放在同一個籃子中，所以一個資料的群集至少會有三個副本 (Replica)，這些副本會被放在不同的 `Data Center`，這些副本之間的資料會用 `Paxos` 一致性演算法來**同步 `Leader` 與 `Slave` 的資料**還有**災難復原**，所以這些資料的所有副本會被當作為一個 `Paxos Group`，而其中一個副本會被選為 `Leader` 負責資料之間的同步與事務開始時的 `Transaction Manager` 與 `Lock Table` 的管理者。


下圖為 Spanner 的架構。

![Alt text](spanner_arch.png)

## Spanner Transaction

Spanner 提供 **Read Only (RO)** 與 **Read Write (RW)** 的事務類型，Spanner 事務的實作使用常見的 `Two Phase Commit` 和 `Two Phase Locking`，而事務之間的衝突解決則是使用 `Timestamp` 來處理但是在跨多個 Server 之間的時間可能有 `Drift` 的問題，Google 則是發明 `TrueTime` 來解決這個問題。


### Read Write Transaction 流程

寫入的事務可以分成在同一個 `Paxos Group` 跟跨多個 `Paxos Group` 的類型來講。

#### 單一個 `Paxos Group`

1. Client 發起事務寫入到某一個資料中，事務請求會發送到目標資料副本的 `Leader`。
2. `Leader` 的 `Lock Manager` 會嘗試獲得目標資料的寫鎖，如果目前有其他事務持有這個事務的讀鎖或寫鎖則會等待，並用 Wound Wait (年輕等待老，如果持有鎖的比較年輕直接去死) 方式來避免 `Dead Lock`。
3. `Leader` 的 `Transaction Manager` 會獲取一個基於 `TrueTime` 的時間戳，利用這個時間戳來確保事務的順序性與一致性，`Leader` 將這個事務的 `時間戳` 與資料基於 `Paxos` 演算法將資料傳給其他的 `Slave`，這邊就進入 `2PC` 的 `Prepare`。
4. `Slave` 資料都收到後回傳成功給 `Leader`，`Leader` 確認後代表資料落地已經可以 `Commit`，就將 `Commit` 的消息傳給 Client 跟另外兩個 `Slave`，並釋放資料的鎖。

![Alt text](write_one_group.png)

#### 跨多個 `Paxos Group`

跨多個 `Paxos Group` 情況就會比較複雜，因為會牽扯到多個 `Paxos Group Leader`。

1. Client 發起寫入跨多個 `Paxos Group` 的事務，Spanner 這個時候會隨機選一個這些 `Paxos Group` 中的一個 `Leader` 作為協調者。
2. 所有相關的 `Paxos Group` 取得需要的鎖。
3. 各自的 `Paxos Group` 做跟單一個寫入時一樣的事情。
4. 執行完成的 `Paxos Group` 會告知協調者已經完成寫入。
5. 協調者收到所有的完成後就會告知所有的 `Paxos Group` 可以將資料 `Commit`。
6. `Paxos Group` 的 `Leader` 會告知其管理的 `Slave` 能夠寫入資料。
7. 協調者告知 Client 寫入成功。

![Alt text](write_many_group.png)

### Read Only Transaction 流程

RO 的事務在 Spanner 中是沒有上鎖，主要是基於 `Snapshot Read`，所以流程會跟 RW 的事務不一樣。

1. Client 在要求 RO 事務 (只有 SELECT) 時會先獲得基於 `TrueTime` 的時間戳，並且將 Request 發到最近的副本 (不一定會是 Leader)。
2. 收到 Request 的副本會先去 `Leader` 檢查自己目前的資料是否為最新的，如果不是就更新副本的資料。
3. 副本將符合 Client 時間戳的資料回傳回去 (因為有可能是多次讀，所以必須利用時間戳來避免 `Non-Repeatable` 的問題發生)。

![Alt text](read_only.png)

### Spanner 時間偏移問題

分散式系統中如果牽扯到 `時間戳` 都會是一個相當困難的議題，因為不同 Server 可能會有出現時間偏移而導致資料的順序出現錯誤，而 Spanner 在設計時就先考慮到這樣的時間偏移問題，在誤差範圍內去比較時間戳的先後順序，並且為降低這個誤差範圍也利用 GPS 跟原子鐘去盡量讓時間同步。


## 結論

Spanner 是一個非常開創且成功的商業運轉分散式資料庫，解決時間問題也處理掉分散式事務的外部一致性的問題。

## 參考
1. [Internals of Google Cloud Spanner](https://thedataguy.in/internals-of-google-cloud-spanner/)
2. [Life of Spanner Reads & Writes](https://cloud.google.com/spanner/docs/whitepapers/life-of-reads-and-writes?hl=en)

