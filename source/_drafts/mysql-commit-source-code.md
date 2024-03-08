---
title: MySQL 源碼解析 - 事務 Commit
tags: mysql
---

## 前言

從開始事務後做完任務就需要將事務給提交 (Commit)，這篇將會從 Source Code 的部分去看 MySQL 在事務提交的時候做些什麼事。
MySQL 除了實體資料的 Commit，但是 Commit 的資料暫時會存在 Buffer Pool 並不會直接 Flush 到 Disk 中，原因是每次更新都要寫入 Disk 會讓 MySQL 變很差，所以 MySQL 這邊會先將資料寫到 Buffer Pool 後，為避免斷電或 Server 壞掉可以讓資料不掉失，會將操作寫入到屬於連續寫入的 `RedoLog` 另外為了要實現主從複製，MySQL 都是用 `BinLog` 來做為主從複製的資料傳遞，而要讓這兩者的資料有一致性，MySQL 利用 `Two Phase Commit` 來保證這兩個資料的一致性，這也是 `事務提交` 的主要做的事情。

而且 `Commit` 除了單純的資料持久化之外，還必須考慮的 Replica。

## Commit

MySQL 為了因應不同 Storage Layer 的引擎，從 MySQL 開始執行 `Commit` 會先去呼叫 `trans_commit` 但這裡面主要都是一些參數的設定。

```c++
/**
  Commit the current transaction, making its changes permanent.

  @param[in] thd                       Current thread
  @param[in] ignore_global_read_lock   Allow commit to complete even if a
                                       global read lock is active. This can be
                                       used to allow changes to internal tables
                                       (e.g. slave status tables, analyze
  table).

  @retval false  Success
  @retval true   Failure
*/

bool trans_commit(THD *thd, bool ignore_global_read_lock) {
    ...
}
```

最主要 `commit` 的實作是在這個函式，這裡面有很多針對 `多層事務`, `Cluster` 的 `commit` 設計，但目前會先主要探討 `innodb` 的 `two phase commit`。
這邊先省略掉其他部分的實作，可以在 `ha_commit_trans` 中看到 `two phase commit` 兩個很重要的階段，`prepare` 跟 `commit` 在這邊被實作出來。

MySQL 為了因應不同 Storage Layer 的引擎，所以除了整個事務的邏輯必須要造著 MySQL 設計的方式來實作外，`prepare` 跟 `commit` 的實作則是會跟引擎不同而有差別，
這兩個函式是 callback function，這邊主要會介紹 `innodb` 針對 `redo log` 與 `binlog` 的實作。

```c++
int ha_commit_trans(THD *thd, bool all, bool ignore_global_read_lock) {
    
  if (ha_info && !error) {
    ...
        if (!trn_ctx->no_2pc(trx_scope) && (trn_ctx->rw_ha_count(trx_scope) > 1))
            error = tc_log->prepare(thd, all);
  }
 
  if (error || (error = tc_log->commit(thd, all))) {
    ha_rollback_trans(thd, all);
    error = 1;
    goto end;
  }
}
```

## Two Phase Commit

`Two Phase Commit` 代表這個 `Commit` 包含兩個階段通常會有 `prepare` 跟 `commit` 這兩個階段，會有這兩個階段的用意是當多個不同機制進行時，可以用這個階段的方式來確保某個操作出現問題後，能夠有效的恢復正常而不會有嚴重的副作用。

### MySQL 為什麼需要 Two Phase Commit

MySQL 在事務 Commit 後並不會馬上把資料寫入到硬碟中，因為事務的操作多為所謂的隨機寫入，在硬碟上這樣的寫入速度是相當慢的，所以都是批次將這些資料一次寫入但這樣就有可能遇到一個問題就是在資料尚未寫到硬碟中時，如果 MySQL 掛掉或者服務器斷電的話這樣就會有資料遺失的問題，所以 MySQL 就用 `RedoLog` 來避免這件事情，`RedoLog` 會紀錄事務 Commit 時的資料，並且 `RedoLog` 是順序寫入所以其寫入速度是大於隨機寫入的。

如果只有 `RedoLog` 事情相對單純，就不需要用到 `Two Phase Commit`，但 MySQL 為了要能夠有集群的功能，用 `BinLog` 來作為主從服務之間資料複製的方式。
這樣就會出現問題，如果 `RedoLog` 寫入成功但 `BinLog` 寫入失敗，這樣就會出現主從的資料不一致性，如果是反過來，這樣主要的 MySQL 在從災難復原後也會出現跟從服務有資料不一致的現象。
所以 MySQL 在這邊引入 `Two Phase Commit` 的用意就是可以根據階段跟這兩個 Log 的狀態來決定是要 `Rollback` 還是當作成功。 


### Prepare

MySQL 的 Prepare 階段主要是會確保 `RedoLog` 有寫入到硬碟中，如果在這個階段就失敗，那其實也代表資料其實沒有被正確寫入。
`RedoLog` 寫入成功後會有一個 XID 傳給 `BinLog`，如果失敗後發現有 `RedoLog` 但其 XID 不再 `BinLog` 中就會 `Rollback`，反之則當作成功。

這邊會介紹 MySQL 的 `innodb` 在 Prepare 這個階段的 Source Code。

`innodb` 在 Prepare 階段的時候會不斷向下呼叫從
`innobase_xa_prepare` -> `trx_prepare_for_mysql` -> `trx_prepare` -> `trx_prepare_low`，`innobase_xa_prepare` 跟 `trx_prepare_for_mysql` 這兩個函式做很多判斷跟檢查，那這篇文章會先跳過那些部分，並把焦點放在 `trx_prepare` 跟 `trx_prepare_low`，因為這兩個函式就是 `innodb` 做寫入 `RedoLog` 的地方。

```c++
static lsn_t trx_prepare_low(trx_t *trx,               /*!< in/out: transaction */
    trx_undo_ptr_t *undo_ptr, /*!< in/out: pointer to rollback
                              segment scheduled for prepare. */
    bool noredo_logging)      /*!< in: turn-off redo logging. */
{
    // 檢查事務是否有新增或更新的操作
    if (undo_ptr->insert_undo != nullptr || undo_ptr->update_undo != nullptr) {
    mtr_t mtr;
    trx_rseg_t *rseg = undo_ptr->rseg;
    // 啟動一個小事務給 Two Phase Commit 做關聯
    mtr_start_sync(&mtr);


    if (undo_ptr->insert_undo != nullptr) {
        // 這邊主要是將小事務所產生的 XID 寫到 Undo Log 的 Header 中
      trx_undo_set_state_at_prepare(trx, undo_ptr->insert_undo, false, &mtr);
    }

    if (undo_ptr->update_undo != nullptr) {
        // 同上
      trx_undo_set_state_at_prepare(trx, undo_ptr->update_undo, false, &mtr);
    }


    // 將 Redo Log 寫到 Disk
    mtr_commit(&mtr);

    }
}
```

```c++
/**
 這邊就是將 RedoLog 寫到 Log Buffer 等待 Flush 到 Disk 中
 **/
void mtr_t::Command::execute() {
  ut_ad(m_impl->m_log_mode != MTR_LOG_NONE);

#ifndef UNIV_HOTBACKUP
  ulint len = prepare_write();

  if (len > 0) {
    mtr_write_log_t write_log;

    write_log.m_left_to_write = len;

    auto handle = log_buffer_reserve(*log_sys, len);

    write_log.m_handle = handle;
    write_log.m_lsn = handle.start_lsn;

    m_impl->m_log.for_each_block(write_log);

    ut_ad(write_log.m_left_to_write == 0);
    ut_ad(write_log.m_lsn == handle.end_lsn);

    log_wait_for_space_in_log_recent_closed(*log_sys, handle.start_lsn);

    DEBUG_SYNC_C("mtr_redo_before_add_dirty_blocks");

    add_dirty_blocks_to_flush_list(handle.start_lsn, handle.end_lsn);

    log_buffer_close(*log_sys, handle);

    m_impl->m_mtr->m_commit_lsn = handle.end_lsn;

  } else {
    DEBUG_SYNC_C("mtr_noredo_before_add_dirty_blocks");

    add_dirty_blocks_to_flush_list(0, 0);
  }
#endif /* !UNIV_HOTBACKUP */

  release_all();
  release_resources();
}
```

### Commit