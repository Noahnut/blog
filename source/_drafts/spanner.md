---
title: 論文閱讀：Spanner： Google’s Globally-Distributed Database
tags: paper
---


## Spanner

Spanner 是 Google 的一套提供類 SQL 語法操作的 Database，而這款 Database 提供 Scalable 跟 Globally Distributed 並且也有提供 Transaction

## Spanner 架構


Universe - Spanner deployment -> Spanner manages data globally
Spanner 由一組 Zones 組成

Zone Master -> location Proxies -> Spanservers 

Spanserver Software Stack

1. Replication
2. Distributed transcations

每一個 Spanserver 負責 100 - 1000 個 tablet 的資料，大概會是這樣的格式 (key:string, timestamp:int64) -> string 因為 timestamp 讓 Spanner 可以支持 multi-version

**tablet** 有點像是將一張表的 row 資料拆分成不同組

tablet 會以類似 B Tree 的檔案跟 WAL 存在一個名叫 Colossus 的 Distribute file system
為了要 Support Replication 每一個 tablet 都有一個 Paxos 的 state machine (存著 tablet 的 metadata 與 log)

Paxos State Machine, 每一個 Paxos 寫 logs 會寫道 tablet's log 跟 Paxos's log
Paxos 實作是 pipeline -> write inorder

Paxos Group -> Set of replica
lock table -> concurrency control -> state for two phase locking

好幾個 Replica 會組成一個 Paxos Group，而在這些 Replica 都有一個 Paxos 的 State Machine，如同我們所知道的共識演算法，這些 Paxos 會有一個 Leader，所以的寫入都是會透過這個 Leader，而這些 Replica 理應要橫跨不同的 Data Center 避免區域性而導致無法處理災難。

transaction manager -> distributed transaction 
    participant leader -> other replica is participant slave 如果 transaction 只有一個 Paxos group 就可以不用過 transaction manager
    如果有多個 Paxos group，這些 group 的leader 就必須合作 two phase commit，在這個 Paxos Group 會選出一個 `coordinate leader` 其他的為 `coordinate slave`  



## TrueTime API (clock uncertainty)

## TrueTime API -> externally-consistent distributed transaction lock free read only transactions, atomic schema updates