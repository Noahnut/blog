---
title: MySQL 源碼解析 - MVCC ReadView 建立
tags: mysql
date: 2023-11-23 21:29:24
---


## 前言

最近在複習 MySQL 的基礎知識，對於 MySQL 如何在一個交易中去產生 `ReadView` 來達到隔離性很有興趣所以就追了一下 Source Code，這篇主要會看事務啟動時 `ReadView` 的建立。
MySQL 的 `MVCC` 實作是利用 `ReadView` 與 `UndoLog` 來達成，`UndoLog` 是實際的歷史資料而 `ReadView` 則是當下快照取到的事務 ID，用這個事務 ID 來去判斷這個 `ReadView` 的資料可見性。

目前看的 MySQL 版本為 `8.0.35`

### ReadView 的結構

`ReadView` 用來判斷哪些資料是否可見，主要是用下面這幾個結構。

**m_creator_trx_id**
建立這個 `ReadView` 的事務 ID。

**m_low_limit_id**
這個 `ReadView` 是無法看到這個 ID 後事務的資料 (比 m_low_limit_id 大的事務 ID)，就算已經 Commit 也是無法。

**m_up_limit_id**
代表小於這個事務 ID 的所有事務都是這個 `ReadView` 可見的。 

**m_ids;**
當下進行中的事務 ID。


```c++
class ReadView {
  ...
  private:
  /** The read should not see any transaction with trx id >= this
  value. In other words, this is the "high water mark". */
  trx_id_t m_low_limit_id; // 代表目前可見的事務 ID，如果其他事務的 ID 大於這個值就代表那些事務都是不可見的

  /** The read should see all trx ids which are strictly
  smaller (<) than this value.  In other words, this is the
  low water mark". */
  trx_id_t m_up_limit_id; // 小於這個值得所有事務是都可見的

  /** trx id of creating transaction, set to TRX_ID_MAX for free
  views. */
  trx_id_t m_creator_trx_id; // 創建這個 ReadView 的事務 ID

  /** Set of RW transactions that was active when this snapshot
  was taken */
  ids_t m_ids; // 可以當作是一個 vector，存著目前進行中的事務
  
}
```


### 交易開始

在 MySQL 中交易開始後會去檢查 `MYSQL_START_TRANS_OPT_WITH_CONS_SNAPSHOT` 這個 flag 是否為 true，這個 flag 代表是否啟用 `consistency snapshot`，如果是 true 就會往下呼叫到 `ha_start_consistent_snapshot` 然後在這裡面會去呼叫一個名為 `snapshot_handlerton` callback function 的參數，這個參數在 innodb 初始化時就會註冊相關的實作。

```c++
/** Initialize the InnoDB storage engine plugin.
@param[in,out]  p       InnoDB handlerton
@return error code
@retval 0 on success */
staticint innodb_init(void *p) {
    ...
  innobase_hton->start_consistent_snapshot =
      innobase_start_trx_and_assign_read_view;
    ...
}
```

### 取得 ReadView

接下來就會呼叫到 `innobase_start_trx_and_assign_read_view`，功能就是開始事務跟取得 `ReadView`。

```c++
/** Creates an InnoDB transaction struct for the thd if it does not yet have
 one. Starts a new InnoDB transaction if a transaction is not yet started. And
 assigns a new snapshot for a consistent read if the transaction does not yet
 have one.
 @return 0 */
static int innobase_start_trx_and_assign_read_view(
    handlerton *hton, /*!< in: InnoDB handlerton */
    THD *thd)         /*!< in: MySQL thread handle of the user for
                      whom the transaction should be committed */
{
    ...
}
```

其中有一段會去判斷現在的隔層層級是否為 `Repeatable Read` 如果是才會去取 `ReadView`，因為這個隔離層級必須要確保資料的獨立性所以在交易開始就需要取得 `ReadView`。

```c++
 /* Assign a read view if the transaction does not have it yet.
  Do this only if transaction is using REPEATABLE READ isolation
  level. */
  trx->isolation_level =
      innobase_trx_map_isolation_level(thd_get_trx_isolation(thd));

    // 隔離層級是  `Repeatable Read` 
  if (trx->isolation_level == TRX_ISO_REPEATABLE_READ) { 
    trx_assign_read_view(trx); 
  } else {
    push_warning_printf(thd, Sql_condition::SL_WARNING, HA_ERR_UNSUPPORTED,
                        "InnoDB: WITH CONSISTENT SNAPSHOT"
                        " was ignored because this phrase"
                        " can only be used with"
                        " REPEATABLE READ isolation level.");
  }
```

如果是 `Repeatable Read` 就會去呼叫 `trx_assign_read_view` 去取得 `ReadView`，在兩種情況下不會取得 `ReadView`。
1. 目前 Database 只允許讀
2. 已經產生過 ReadView 

```c++
/** Assigns a read view for a consistent read query. All the consistent reads
 within the same transaction will get the same read view, which is created
 when this function is first called for a new started transaction.
 @return consistent read view */
ReadView *trx_assign_read_view(trx_t *trx) /*!< in/out: active transaction */
{
  ut_ad(trx_can_be_handled_by_current_thread_or_is_hp_victim(trx));
  ut_ad(trx->state.load(std::memory_order_relaxed) == TRX_STATE_ACTIVE);

    // 如果目前 Database 是只允許讀，不需要產生 ReadView
  if (srv_read_only_mode) { 
    ut_ad(trx->read_view == nullptr);
    return (nullptr);

    // 如果已經產生 ReadView，就不用在去產生
  } else if (!MVCC::is_view_active(trx->read_view)) { 
    trx_sys->mvcc->view_open(trx->read_view, trx);
  }

  return (trx->read_view);
}

```

### 產生 ReadView

這邊就是產生 `ReadView` 的主要邏輯。

```c++
/** Allocate and create a view.
@param view     View owned by this class created for the caller. Must be
freed by calling view_close()
@param trx      Transaction instance of caller */
void MVCC::view_open(ReadView *&view, trx_t *trx) {
  ...
}
```

這邊針對 `ReadView` 的邏輯可以分成兩塊，第一塊是如果在最後一個 `ReadView` 之後都沒有新事務，而且這個事務是**只讀的事務**那這樣代表可以重複使用之前的 `ReadView`。

```c++
 ut_ad(!srv_read_only_mode);

  /** If no new RW transaction has been started since the last view
  was created then reuse the the existing view. */

  // 檢查是否已經有 ReadView
  if (view != nullptr) {                                
    uintptr_t p = reinterpret_cast<uintptr_t>(view);

    view = reinterpret_cast<ReadView *>(p & ~1);

    ut_ad(view->m_closed);

    /* NOTE: This can be optimised further, for now we only
    reuse the view if there are no active RW transactions.

    There is an inherent race here between purge and this
    thread. Purge will skip views that are marked as closed.
    Therefore we must set the low limit id after we reset the
    closed status after the check. */

    // 檢查目前事務是否已讀且沒有任何進行中尚未提交的事務
    if (trx_is_autocommit_non_locking(trx) && view->empty()) {   
      view->m_closed = false;

      // 檢查當前事務的最大能見事務 ID 是否與下一個分配的事務 ID 一樣
      // 如果一樣代表這個 ReadView 就是最新
      if (view->m_low_limit_id == trx_sys_get_next_trx_id_or_no()) { 
        return;
      } else {
        view->m_closed = true;
      }
    }
  }
```

根據目前事務情況建立 `ReadView`，並且呼叫 `prepare` 去將 `ReadView` 所需要的相關事務 ID 塞到裡面。

```c++
  if (view != nullptr) {
    view->prepare(trx->id);
    ...
  }
```

### 針對 ReadView 處理對應的事務 ID

```c++
/**
Opens a read view where exactly the transactions serialized before this
point in time are seen in the view.
@param id               Creator transaction id */

void ReadView::prepare(trx_id_t id) {

    // 當前事務的 ID
  m_creator_trx_id = id; 

    // 取得下一個事務的 ID
  m_low_limit_id = trx_sys_get_next_trx_id_or_no(); 

    // 檢查目前是否有進行中的事務
  if (!trx_sys->rw_trx_ids.empty()) { 
    // 整份拿過來
    copy_trx_ids(trx_sys->rw_trx_ids); 
  } else {
    m_ids.clear();
  }

  /* The first active transaction has the smallest id. */
  // 如果有進行中的事務，那最後可見的事務就是小於目前最舊的那個進行中事務
  // 如果沒有代表目前事務都是可見，除了後面新產生的事務
  m_up_limit_id = !m_ids.empty() ? m_ids.front() : m_low_limit_id; 
}
```


### 結論

1. `ReadView` 是用事務 ID 來做可見及不可見的區分
2. 詳細的資料還是存在 `undo log` 中
3. 目前理解起來，事務越多執行效率越差，因為要歷遍整個 `undo log` 才能找到目標的可見資料。