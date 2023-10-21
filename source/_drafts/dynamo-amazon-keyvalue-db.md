---
title: Dynamo Amazon 的高可用性 Key-Value Database
tags:
- paper
- database
---

## Dynamo 的意義

一般的 Strong Consistent Database 在處理大流量時通常都會需要做 Replication 但是一筆寫入要同步到其他 Replication 時出現 Network Failure 或 Node 出現問題時，就會導致這筆寫入是無法成功的，Amazon 意思到這樣的現象會嚴重造成顧客的體驗差，所以自行設計一套 Key-Value Database，犧牲掉 Strong Consistent (他們常用的操作情境其實不需要 Strong Consistent) 取而代之的是 最終一致性 (Eventually-consistent)、High Available 與讀寫高效能，這套 Database 的名稱就叫做 **Dynamo**。

## Dynamo 的實作理論

Dynamo 具有 Scalability 與 High Available 並保證最終一致性 (Eventually-consistent)，且做 Scalability 時不管是增長機器還是減少機器都不需要人工介入也不需要做任何資料上的調整。
這些強大的功能，Dynamo 根據不同的需求使用不同的技術。

### High Available 跟 Scalability 

* 資料的分區 (Partition) 與副本 (Replication)，這樣可以將資料放在不同的 Data Center 避免意外發生導致資料遺失
* Gossip Protocol 去監控是否有死掉的節點

### Scalability
    
* Dynamo 使用 Consistency Hash 來達到成本較低的 Node 增加與減少

### Eventually-consistent

強一致性的 DB 會犧牲 HA，在出現 Network Failure 時，強一致性的 Database 就必須等到恢復正常後才能繼續提供服務

### Conflict Resolve

#### 什麼時候處理 Conflict

Amazon 的 HA 建立不是在讀上而是寫上，原因是顧客無法更新購物車是會個不好的體驗，所以 Dynamo 這件事交給 Read

#### 誰去處理 Conflict

Dynamo 由 Database 自行處理，採用 Last write wins

### Dynamo 目標

1. 就算網路出現或部份機器出現問題都還可更新跟寫入
2. 所有的節點因為是內部服務，所以都需要被當可信的
3. 不用支援複雜度 file namespace 或 Schema (只有 Key/Value)
4. 高效能與低延遲，Amazon 希望 Dynamo 的 SLA 能夠維持在 99.9% 的讀寫操作都能夠在幾百個毫秒內完成

### Dynamo 的核心技術

1. Partition
2. Replication
3. Versioning (object Conflict)
4. Failure Handling
5. Scaling

* Version object 來處理衝突，跟 Quorum like Algorithm 來決定最後的資料 (新的 Amazon DynamoDB 已改用其他方式) 


Dynamo 怎麼去達到 HA 與 Scalability 從論文中的一張圖片可以知道，他們遇到什麼問題，跟用什麼技術去解決 
![Alt text](tech_summary.png)