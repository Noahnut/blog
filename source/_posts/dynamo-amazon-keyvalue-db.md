---
title: Dynamo Amazon 的高可用性 Key-Value Database
tags:
  - paper
  - database
date: 2024-03-08 11:21:01
---


## Dynamo 的背景

Amazon 面對大流量的電商相關的操作，如果有一些用戶無法使用就會造成嚴重的用戶體驗不佳的問題，所以 Amazon 根據他們的使用場景設計出具有**高可用性**、**高效能**和**水平擴展**但犧牲**強一致性**的分散式 Key-Value 資料庫。

## Dynamo 的設計

Amazon 根據他們的需求提出以下的設計原則，並接下來會討論他們在這些設計原則下使用哪些技術來完成他們的目的。

### Dynamo 設計原則

1. **Incremental Scalability**
    Dynamo 的 Node 應該可以在不影響系統的情況下水平擴展或縮減。
2. **Symmetry**
    每一個 Node 應該負責的功能都要相同，不應該有些 Node 存有特定資料。
3. **Decentralization**
   去中心化的系統，讓系統可以更容易水平擴展與高可用性。
4. **Heterogeneity**
    不同的服務去使用 Dynamo 不需要因為 Dynamo 的 Node 增減也去做服務本身的調整。

#### Partitioning

Dynamo 要達到 `Scalability` 與 `高可用性`，資料的分區 (Partitioning) 是非常重要的，Dynamo 為了要達到增刪 Node 的時候對於資料的影響降到最小，採用 `Consistent Hash` 的方法來進行資料分區。

`Consistent Hash` 演算法是想像將 Node 放在一個圓上，然後根據資料 Hash 後的值來看其在圓上順時針後遇到的第一個 Node 就是資料要存放的地方，如果增減節點會影響的只有其順時針向的另外一個節點。

![Alt text](consistent_hash.png)

最基礎的 `Consistent Hash` 演算法會有一個問題就是當 Node 數量少的時候，因為 Node 在圓上的位置可能沒這麼均衡而導致某些 Node 出現資料傾斜，所以 Dynamo 這邊加入 `Virtual Node`，這個 `Virtual Node` 的意思就是在圓上多幾個 Node 而這些 Node 實際上都代表著同一台實體的 Node，當數量增加就可以讓資料的分區均衡。

![Alt text](virtual_node_consistent.png)


#### 寫入的高可用性

Dynamo 設計時的目的就是要避免 Amazon 的客戶在使用電商網站時出現新增產品但出現錯誤，產品新增失敗而導致用戶體驗不佳，所以 Dynamo 非常重視寫入資料時的高可用性。

Dynamo 的資料在 Amazon 架構中會有多個副本在不同 Node 中，目的是提供資料的高可用性與減少網路距離的消耗，但是這樣的架構下就必須面對 **一致性** 的問題而 Dynamo 為了要達成寫入的高可用性選擇 `最終一致性` 作為其一致性的標準，`最終一致性` 所代表的意思是可以接受 Node 之間的資料可以不一樣但最後會變成一樣，這樣就能提高寫入的效能與高可用性但這可能會出現資料在不同 Node 上有不同的值，而 Dynamo 這邊是使用 `Vector Clock` 來處理資料衝突的問題。

##### Vector Clock

`Vector Clock` 是一種用來處理資料的先後順序與因果關係衝突的演算法，每一個 Node 在跟其他 Node 同步資料的時候都會帶上 `(Node, Counter)`，`Counter` 所代表的意思是每一個資料更新
時的遞增值，所以一個值被更新時會有不同時間的 `Vector`，例如 `Node A` 寫入 `Object A` 就會有 `(Node_A, 1)`， 然後 `Node A` 再將其同步給其他節點，其他節點就會拿到包含 `(Node_A, 1)` 的物件，假設今天有兩個節點，分別是 `Node B` 跟 `Node C` 都接受 `Object A` 的更新就會出現資料版本的分歧，`Vector Clock` 的做法是將所有的版本先記錄下來，如果出現衝突在由用戶端決定哪個資料是要留下來的。

#### 短暫停機資料處理 Hinted Handoff

`Dynamo` 需要寫的高可用性，所以 `Strict Quorum` 機制可能會導致寫入失敗，所以 `Dynamo` 採用 `Sloppy Quorum`，差別在於 `Strict Quorum` 需要維持節點資料的一致性但 `Sloppy Quorum` 可以接受暫時的不一致等節點復活後再同步就好。

`Dynamo` 的 `Sloppy Quorum` 作法是當寫入的節點無法使用的時候先將資料寫入到在 `Consistence Hash` 環中的下一個節點，並記錄這個資料原本是要寫入到哪一個節點，等節點復活後再將資料同步過去，這個作法稱為 `Hinted Handoff`。

#### 停機後復原

`Hinted Handoff` 能夠處理節點暫時的失去功能，但如果出現接收 `Hinted` 的節點在將資料傳給原始節點前就失去功能，這樣就會出現副本之間的不一致，`Dynamo` 使用 `Anti-Entropy` 協議來處理副本之間的同步，`Anti-Entropy` 會讓每一個節點都維護在 `Ring` 中各自 Key Range 的 `Merkle Tree`，當發現節點的某一顆 `Merkle Tree` 的 Hash 值不一致後就開始從源頭找到哪一部分的資料不相同並進行資料的同步，因為 `Merkle Tree` 的特性可以減少 Disk I/O 的次數。

#### 節點新增刪除

`Dynamo` 透過顯式的方法 (Terminal 或瀏覽器等) 由管理員操作增刪節點的操作，並且透過 `Gossip Protocol` 的方式，先由一個節點知道增刪操作，這個節點再將這個訊息傳給鄰近節點，持續到所有節點都收到相關訊息，透過這樣的方式就能夠保證最終所有節點都會知道操作。
 


![Alt text](tech_summary.png)