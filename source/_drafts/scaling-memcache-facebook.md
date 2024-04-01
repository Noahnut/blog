---
title: 論文閱讀：Scaling Memcache at Facebook
tags: paper
---

本文會介紹 Facebook 在面對大流量時設計可擴展性的 `Memcache` 所遇到的問題與解法。

## 需求

## Cache Policy

### 讀取策略

### 更新策略

## 單台 Memcache 遇到的問題

### Stale Sets

### Thundering Herds

## Cluster Memcache 遇到的問題

### 提高 Throughput

#### Consistency Hash

#### Congestion Control

#### Memcache Pools

#### 將部份邏輯放到 Client 端


## Region Clusters Memcache 遇到的問題

### 資料一致性

#### 更新時同步刪除 Cache

#### 標記過期 Cache

## 總結