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

Paxos State Machine



## TrueTime API (clock uncertainty)

## TrueTime API -> externally-consistent distributed transaction lock free read only transactions, atomic schema updates