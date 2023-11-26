---
title: MySQL 源碼解析 - 事務開始
tags: mysql
date: 2023-11-26 10:54:21
---


## 前言

從 Source Code 來看 MySQL 在事務開始後會有哪些操作。

## 事務開始的時機

MySQL 中所有的語法其實都是一個隱性的事務，但是都無法由使用者自行控制邏輯上的 Commit 或 Rollback，而顯性的事務控制是由以下關鍵字組成。

* `START TRANSACTION READ WRITE`

    開啟一個允許讀寫的事務。

* `START TRANSACTION READ ONLY` 
    
    開啟一個只允許讀的事務，這個事務只能讀不能對任何值做修改。

* `START TRANSACTION WITH CONSISTENT SNAPSHOT` 
    
    開始一個事務並獲得當前事務的 `ReadView`。

* `BEGIN`
    
    功能類似 `START TRANSACTION`，但實際是以上的哪一個模式主要是看用戶的設定，預設是 `START TRANSACTION READ WRITE`。


以下是 MySQL 實際執行語法的函式，上述的事務語法都會被解析成 `SQLCOM_BEGIN`，並且帶入不同模式的參數。
```c++
int mysql_execute_command(THD *thd, bool first_level) {
    ...
case SQLCOM_BEGIN:
    // 以上的關鍵字都會被處理成這個代碼，lex->start_transaction_opt 主要是看是輸入什麼模式來決定。
      if (trans_begin(thd, lex->start_transaction_opt)) goto error;
      my_ok(thd);
      break;
    ...
}
```

## 事務開始


```c++
// THD 是這個 Connection 的相關資訊，flags 是這個事務的類型
bool trans_begin(THD *thd, uint flags) {
    ...
}
```

事務開始前會先將這個連線尚未結束的事務與上的鎖都先釋放掉。

```c++

    // 釋放目前連線上的鎖
  thd->locked_tables_list.unlock_locked_tables(thd);

    // 釋放目前連線的其他事務
  if (thd->in_multi_stmt_transaction_mode() ||
      (thd->variables.option_bits & OPTION_TABLE_LOCK)) {
    thd->variables.option_bits &= ~OPTION_TABLE_LOCK;
    thd->server_status &=
        ~(SERVER_STATUS_IN_TRANS | SERVER_STATUS_IN_TRANS_READONLY);
    DBUG_PRINT("info", ("clearing SERVER_STATUS_IN_TRANS"));
    res = ha_commit_trans(thd, true);
  
   /*
   釋放這個事務已經被 Commit 的 Meta Data Lock
  */
  thd->mdl_context.release_transactional_locks();
```


## 設置事務相對應狀態

會根據用戶在開始事務時的指定 `RO (Read Only)` 或 `RW (Read Write)`，來針對這些事務 `tx_read_only` 的 flag。

```c++

    // 如果是 START TRANSACTION READ ONLY 
  if (flags & MYSQL_START_TRANS_OPT_READ_ONLY) {
    thd->tx_read_only = true;
    if (tst) tst->set_read_flags(thd, TX_READ_ONLY);
    // 如果是 START TRANSACTION READ WRITE
  } else if (flags & MYSQL_START_TRANS_OPT_READ_WRITE) {
    /*
      Explicitly starting a RW transaction when the server is in
      read-only mode, is not allowed unless the user has SUPER priv.
      Implicitly starting a RW transaction is allowed for backward
      compatibility.
    */
    if (check_readonly(thd, true)) return true;
    thd->tx_read_only = false;
    /*
      This flags that tx_read_only was set explicitly, rather than
      just from the session's default.
    */
    if (tst) tst->set_read_flags(thd, TX_READ_WRITE);
  }
```

## 開始事務

這個 Connection 針對事務的狀態改成開始事務。

```c++
thd->variables.option_bits |= OPTION_BEGIN;
thd->server_status |= SERVER_STATUS_IN_TRANS;
```

如果是隔離層級為 `Repeatable Read` 或 `START TRANSACTION WITH CONSISTENT SNAPSHOT`，在這邊就會去產生出當下的 `ReadView`。

```c++
 /* ha_start_consistent_snapshot() relies on OPTION_BEGIN flag set. */
  if (flags & MYSQL_START_TRANS_OPT_WITH_CONS_SNAPSHOT) {
    
    // 產生 ReadView 
    res = ha_start_consistent_snapshot(thd);
  }
```

## 結論

MySQL 開始事務的邏輯基本上都是在設置相對應的參數，實際上 `undo log`、上鎖或者衝突時的處理可能會再往下看到 `Commit` 或 `Rollback` 的部分。