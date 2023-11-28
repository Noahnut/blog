---
title: Redis 源碼解析 - PubSub 實作 (Subscribe)
tags: redis
date: 2023-11-28 16:16:08
---


## 前言

`PubSub` 是 Redis 中作為訊息傳遞的一種工具，可以藉由訂閱特定 `topic (channel)` 收到來自發送自這個  `topic (channel)` 的訊息，之前工作上開發的即時通訊系統，服務之間的溝通就是透過 `PubSub` 來溝通，所以想藉由閱讀 Redis 中的 `PubSub` 的 Source code 來了解他的實作機制。

## PubSub 的訂閱 (Subscribe)

在 Redis 中訂閱有兩種方式。

1. `subscribe <channel 1> <channel 2>...`

   訂閱指定名稱的 `topic (channel)`，在 Redis 中可以一次訂閱多個。

2. `psubscribe <pattern1> <pattern2> ...`

   基於 glob-style patterns 的去訂閱符合 pattern 的 `topic (channel)`。

這兩種方式在 Redis 中有不同的實作方式，但是訂閱的部分 Redis 這邊的實作方式都是用 `Hash` + `Linked List`。

### Subscribe 的實作

PubSub 這邊會有兩個不同的 `Dict`，一個是 `Client` 自己的來紀錄自己所訂閱過的 `Channel`，另外一個是 `Server` 的，用來記錄這個 `Channel` 有哪些 `Client` 在訂閱。

```c
/*  Client 訂閱指定 topic，成功就回傳 1 如果重複訂閱則回傳 0 */
int pubsubSubscribeChannel(client *c, robj *channel, pubsubtype type) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* 將 Channel 加到 Client 的訂閱 Dict */
    if (dictAdd(type.clientPubSubChannels(c),channel,NULL) == DICT_OK) {
        retval = 1;

        // Channel 的引用次數加一
        incrRefCount(channel);
        /* 檢查這個 Channel 是否已存在 Server Side 中 */
        de = dictFind(*type.serverPubSubChannels, channel);

        // 不存在則新增一筆
        if (de == NULL) {
            // 建立訂閱這個 Channel 的 Client 的 Linked List 
            clients = listCreate();
            dictAdd(*type.serverPubSubChannels, channel, clients);
            incrRefCount(channel);
        } else {
            // 存在則將 Linked List 取出來
            clients = dictGetVal(de);
        }

        // 新訂閱的 Client Append 在尾端
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    addReplyPubsubSubscribed(c,channel,type);
    return retval;
}
```


### Psubscribe 的實作

Patterns 的訂閱也是用 `Hash` 跟 `Linked List` 來實作，Server 端跟 Client 端提供另外一個 `pubsub_patterns` 的 `Hash` 資料結構去存儲 Patterns 類性的訂閱。  

```c
/* Client 訂閱指定 topic 的 pattern，成功就回傳 1 如果重複訂閱則回傳 0  */
int pubsubSubscribePattern(client *c, robj *pattern) {
    dictEntry *de;
    list *clients;
    int retval = 0;


    // 新增 pattern 到 client 的 dict
    if (dictAdd(c->pubsub_patterns, pattern, NULL) == DICT_OK) {
        retval = 1;
        incrRefCount(pattern);
        /* 在 Server 端找這個 pattern 是否已經存在 Dict 中 */
        de = dictFind(server.pubsub_patterns,pattern);

         // 不存在則新增一筆
        if (de == NULL) {
            clients = listCreate();
            dictAdd(server.pubsub_patterns,pattern,clients);
            incrRefCount(pattern);
        } else {
            clients = dictGetVal(de);
        }

        // 新訂閱 Pattern 的 Client Append 在尾端 
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    addReplyPubsubPatSubscribed(c,pattern);
    return retval;
}
```


## 結論

PubSub 的 `Subscribe` 機制非常單純的，只是根據他性質的不同來決定他對應的資料是要放在 `Channel` 還是 `patterns` 下，接下來的章節會介紹 `Publish` 訊息是怎麼做的。


