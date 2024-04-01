---
title: Golang 記憶體管理與分配
tags: golang
---

## 前言

使用 `Golang` 的都知道，`Golang` 跟 `C/C++` 不同的是其記憶體管理並不是由開發者自行處理，那這樣交由語言本身的記憶體管理跟分配的架構跟邏輯是如何？本文會盡量用簡單的方式讓閱讀者知道整個 `Golang` 記憶體管理的架構、邏輯跟設計的巧思。

## Golang 的記憶體管理

基本上當 Process 在常見的作業系統執行起來後其在 `Virtual Memory` 中都會是以下圖的形式，本文所探討的部分主要會是在針對 `Heap` 記憶體管理的部分，因為 `Heap` 記憶體所放的內容通常為使用者自行宣告 (例如 Pointer 或者會被引用的參數)，而 `Stack` 符合 `LIFO` 的性質所以通常都是函式的參數或者 `Local Variable`。
![Alt text](virtual_memory.png)

設計不良的記憶體管理可能會讓 Process 出現碎片, 不斷向作業系統要求記憶體而導致高成本的 `Context Switch` 等等問題，而 `Golang` 因為其 `Go Routine` 的設計所以需要一個良好的記憶體管理來讓其發揮最大的效能，所以 `Golang` 就參考 `TCMalloc`設計屬於自己的記憶體管理方式。

### TCMalloc (Thread Cache Malloc)

`TCMalloc` 是 Google 所開發的記憶體管理，目的是要解決 glibc 的記憶體管理在多執行緒 (Multi-Thread) 的執行下會有效能低落的問題。
`TCMalloc` 的設計想法為減少鎖的使用跟降低鎖的粒度來降低因多執行緒 (Multi-Thread) 時的鎖競爭而導致的效能問題。
下圖為 `TCMalloc` 記憶體管理的架構，可以發現每一個執行緒 (Thead) 都有自己的記憶體分配空間，這樣當執行緒 (Thread) 需要記憶體時就可以先從自己那邊取得，這樣就可以減少鎖的使用。
![Alt text](tcmalloc.png)

從 `TCMalloc` 的設計理念就可以知道為什麼 `Golang` 會參考 `TCMalloc`，接下來會詳細介紹 `Golang` 的記憶體管理方式。

### Golang 記憶體管理架構

下圖是 `Golang` 的記憶體管理架構，從下圖可以發現非常架構類似 `TCMalloc`，每一個 `Thread` 都有自己的記憶體 `Cache` 並且針對大物件跟小物件分配有不同的處理方式降低使用鎖的時間，來達到在多執行緒環境下高效的記憶體管理方式。

![Alt text](golang_memory.png)

### mcache

每一個執行緒 (Thread) 都有一個預先分配好的記憶體空間，小於 32K 的物件都會優先從這個地方取得，因為不需要跟其他執行緒競爭物件所以是不需要鎖的。

### mcentral

由 `mheap` 所管理的物件，提供給需要記憶體的執行緒 (Thread) 根據物件大小取得對應的空間，因為是共有物件所以需要鎖來處理 `Race Condition` 的問題，`mcentral` 提供

### mheap

`mheap` 管理 `mcentral` 與由 `arenas` 所管理的目前空閒的 `page`。  

### 小物件分配

### 大物件分配

## 結論

## Reference

[Go: Memory Management and Allocation
](https://medium.com/a-journey-with-go/go-memory-management-and-allocation-a7396d430f44)
[Go 语言内存分配器的实现原理 | Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)
[TCMalloc : Thread-Caching Malloc](https://goog-perftools.sourceforge.net/doc/tcmalloc.html)