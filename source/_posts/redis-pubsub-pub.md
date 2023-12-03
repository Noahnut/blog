---
title: Redis 源碼解析 - PubSub 實作 (Publish)
tags: redis
date: 2023-12-03 10:41:21
---


## 前言

上一篇介紹完 `PubSub` 中的 `Subscribe` 這篇就會開始介紹 `Redis` 是怎麼針對 `Topic` 發送訊息還有訂閱的是 `Pattern` 的使用者怎麼收到符合 `Pattern` 的訊息。

## PubSub 的發送 (Publish)

`Redis` 中發送的指令如下

`PUBLISH <Topic> <Message>`

## Publish 的實作

`Publish` 的實作分為兩個部分，分別為發送到訂閱這個 `topic` 的使用者跟訂閱 `pattern` 符合這個 `topic` 的使用者。

`Publish` 的實作內容都在這個裡面。
```c++
/*
 * Publish a message to all the subscribers.
 */
int pubsubPublishMessageInternal(robj *channel, robj *message, pubsubtype type) {
    ...
}
```

第一部分會從 `topic` 的 Dict 中找是否有符合，並且將訊息發送到這個下面的所有用戶。
```c++

    // 找到有訂閱這個 topic 的用戶
    de = dictFind(*type.serverPubSubChannels, channel);
    if (de) {
        list *list = dictGetVal(de);
        listNode *ln;
        listIter li;

        listRewind(list,&li);
        while ((ln = listNext(&li)) != NULL) {
            client *c = ln->value;
            addReplyPubsubMessage(c,channel,message,*type.messageBulk);
            updateClientMemUsageAndBucket(c);
            receivers++;
        }
    }
```

第二部分就是找目前用戶訂閱的 `Pattern` 是否有符合發送訊息的 `topic`。
```c++
    // 歷遍 Pattern，並檢查 Pattern 是否符合 Topic，如果符合就發送訊息給訂閱這個 Pattern 的使用者。
    di = dictGetIterator(server.pubsub_patterns);
    if (di) {
        channel = getDecodedObject(channel);
        while((de = dictNext(di)) != NULL) {
            robj *pattern = dictGetKey(de);
            list *clients = dictGetVal(de);
            if (!stringmatchlen((char*)pattern->ptr,
                                sdslen(pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) continue;

            listRewind(clients,&li);
            while ((ln = listNext(&li)) != NULL) {
                client *c = listNodeValue(ln);
                addReplyPubsubPatMessage(c,pattern,channel,message);
                updateClientMemUsageAndBucket(c);
                receivers++;
            }
        }
        decrRefCount(channel);
        dictReleaseIterator(di);
    }
    return receivers;
```

如果 Redis 有 Sharding 是無法訂閱 Pattern，發送訊息對應的 `Pattern` 也是收不到。
```c++
 if (type.shard) {
        /* Shard pubsub ignores patterns. */
        return receivers;
 }
```

## 結論

Redis 的 `PubSub` 的實作相當簡單。