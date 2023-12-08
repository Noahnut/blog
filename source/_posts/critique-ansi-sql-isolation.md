---
title: 論文閱讀：A Critique of ANSI SQL Isolation Levels
tags: paper
mathjax: true
date: 2023-12-08 16:53:58
---


## 前言

ANSI SQL-92 是在 1992 年所定製的相關 SQL 規範，其中就定義四種 `Isolation Level` 分別有 `Read Uncommitted`, `Read Committed`, `Repeatable Read`, `Serializable` 跟四種分別要解決的三種問題，分別有 `Dirty Read`, `Non-Repeatable Read`, `Phantom`，隨著隔離層級越高就代表能夠解決的問題越多但平行度也越低。

但是這些規範跟提到的問題並沒有提供一個明確的解釋跟在定義時沒有將 `Lock` 的概念一起考慮進去，所以導致目前主流資料庫的設計方式都會跟 ANSI 的 `Isolation Level` 有點出入，並且有些問題也是 ANSI SQL-92 沒有定義的，例如：`Write Skew`, `Read Skew`。

這篇文章會介紹論文作者們怎麼去定義現在的 `Isolation Level`。

## Database 中會出現問題的現象跟錯誤

在論文中用 ***P*** 表示事務執行順序的現象，***A*** 代表會出現問題的事務執行順序，論文中詳細的定義這些錯誤發生的流程與原因。

而這邊的邏輯符號是這樣解讀的：
    $w1[x]$ ：事務 1 寫入到 $x$。
    $r1[x]$ ：事務 1 從 $x$ 讀取資料。
    $c1$： 事務 1 `Coomit`。
    $a1$：事務 1 `Abort`。
    $P$: 資料區間，類似 `WHERE` 的語法。

**Dirty Read (P1)**

另外一個事務尚未 `Commit` 或 `Rollback`，現在的事務就讀到其更新過後的值。

$P1:\space w1[x]...r2[x]...$

$A1:\space w1[x]...r2[x]...$

**Non-Repeatable or Fuzzy Read (P2)**

當下事務讀一次資料，然後另外一個事務更新同一個資料並 `Commit`，導致事務在讀一次得到的資料跟上一次的不一樣。

$P2:\space r1[x]...w2[x]...$

$A2:\space r1[x]...w2[x]...c2...r1[x]...c1$

**Phantom (P3)**

當下事務讀一個區間中的資料，然後另外一個事務在更改這個區間中的資料 (Insert 或 Update)，然後 `Commit`，導致當下事務在讀一起得到跟上次不同的資料。

$P3:\space r1[P]...w2[y\space in \space P]...$

$A3:\space r1[P]...w2[y\space in \space P]...c2...r1[P]...c1$

**Lost Update (P4)**

當下事務讀出資料並根據資料狀態準備寫入，但另外一個事務先寫入，而當下事務寫入後並 `Commit`，導致另外一個事務的寫入消失。

$P4:\space r1[x]...w2[x]...w1[x]...c1$

**Read Skew (A5A)**

當下事務的值因為被其他更改而導致值已經跟事務開始時不一樣。

$A5A:\space r1[x]...w2[x]...w2[y]...c2...r1[y]...(c1 \space or \space a1)$

**Write Skew (A5B)**

當前事務從 x 讀出某個值並用這個值去更新 y，另外一個事務則是先讀出 y 值後去更新 x 值，這樣就導致 x 跟 y 值得不一致性。

$A5B:\space r1[x]...r2[y]...w1[y]...w2[x]...(c1 \space and \space c2)$

## 解決錯誤的隔離層級

ANSI SQL-92 提供的是一個概念，而現在的 Database 設計實作時都會採用一些不同的概念 (MVCC, Lock...) 來實作，有些時候這個隔離層級雖然在某個資料庫被定義為 `Repeated Read` 但是搭配一些其他的實作卻可以解決要在上一個層級才能解決的現象，所以下方這張圖表提供一個大概念但也會讓人跟現代資料庫的實作搞混，像是 MySQL 的 `Repeated Read` 搭配鎖可以解決 `Phantom` 的問題。 

![Alt text](ansi_sql_table.jpeg)

所以論文就介紹在現代資料庫中最常出現的兩個實作概念，並且搭配 ANSI SQL-92 來完整解決錯誤的規範。

### Snapshot 

在事務開始的時候，將當下要讀寫的資料做成一個副本，並且在 `Commit` 的時候去處理衝突，這個技術根據不同 Database 都有不一樣的實作，例如 MySQL 就是用 `Undo Log` 搭配 `ReadView`，這邊就不特別介紹詳細的技術細節，但是可以知道這樣的做法可以提高資料處理的平行化。 


### Lock

論文中把上鎖的隔離層級分成 4 種 Degree，根據讀鎖跟寫鎖的是否持有與持有時間來決定其等級，數字越大代表等級越高也代表其鎖的持有時間越長，平行度越低。

**Degree 0**: 可以允許 `Dirty Read/Write` 只需要保持 `Atomicity`
**Degree 1 (Locking Uncommitted)**: 讀不上鎖，寫上鎖並到事務結束
**Degree 2 (Locking Read Committed)**: 讀上鎖但只有操作資料的時候上鎖，寫上鎖並到事務結束
**Degree 3 (Locking Serializable)**:  讀上鎖直到事務結束，寫上鎖後也是直到事務結束

根據上述實作與 ANSI SQL-92 綜合起來就整理出一份能解決問題實作的圖表。

![Alt text](total_isolation.jpeg)


## 結論

我個人對於這篇論文的最後解讀是，`Isolation Levels` 在各個不同 Database 的實作都不同，能夠解決的問題範圍也是不同，所以在使用 `Isolation Levels` 不應該相信其所代表的意思，而是能夠解決的問題。