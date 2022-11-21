---
title: undo 清理
date: 2022-05-02 02:09:23
tags:
- MySQL
---

# purge_queue

purge_queue中记录了所有等待Purge的Rollback Segment和其History中trx_no最小的事务


# 预处理


storage/innobase/trx/trx0undo.cc

事务结束的时候，对于需要Purge的Update类型的Undo Log，会按照事务提交的顺序trx_no，挂载到Rollback Segment Header的History List上。

在 trx_undo_update_cleanup 函数中调用

```c
void trx_undo_update_cleanup(trx_t *trx, trx_undo_ptr_t *undo_ptr, page_t *undo_page, bool update_rseg_history_len, ulint n_added_logs, mtr_t *mtr) {

  ...

  // 将 update undo 添加到 history
  trx_purge_add_update_undo_to_history(
      trx, undo_ptr, undo_page, update_rseg_history_len, n_added_logs, mtr);
  ...
}
```

# 流程


<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/purge.svg">



`srv_do_purge()` 为入口函数


`trx_purge()` 获取最老的视图复制给 `purge_sys->view`，方便之后真正 purge undo log 时判断其是否不会再被访问到了。

```c
trx_sys->mvcc->clone_oldest_view(&purge_sys->view);
```


通过 `trx_purge_attach_undo_recs()` 获取需要被 purge 的undo log：

```c
n_pages_handled = trx_purge_attach_undo_recs(n_purge_threads, batch_size);
```

trx_purge_attach_undo_recs

通过 `trx_purge_fetch_next_rec()` 循环获取可以被purge 的 undo log，默认最多获取 300 个 undo 页

> 可以通过 innodb_purge_batch_size 来调整


## trx_purge_fetch_next_rec 流程

1. trx_purge_choose_next_log

该函数作用：选择下一个需要清理的 undo log，并且更新 purge_sys 的信息

通过 `trx_purge_choose_next_log()` 从`purge_sys::purge_queue` 取出第一个回滚段

<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/trx_purge_choose_next_log.svg">

2. trx_purge_get_next_rec

从其 history list 上读取最老还未被 purge 的事务的undo log header。
从此 undo log header 依次读取undo log record，每次读取会设置purge_sys的变量

读取完毕后，重新统计此回滚段最老还未被purge的事务的位点，然后重新放入purge_sys::purge_queue；最后回到第一步。


## trx_purge_choose_next_log

依次从 purge_queue 中 pop 出拥有全局最小 trx_no 的 Undo Log。



## TrxUndoRsegsIterator::set_next() 流程 

如果满足 `m_iter != m_trx_undo_rsegs.end()`，表示当前即将处理的 rollback segment 还未到最后一个，即还有更多的 rollback segment 需要处理。


如果当前即将处理的 rollback segment 已经到最后一个了，那么将会进入一个 `while(!m_purge_sys->purge_queue->empty())` 循环：

- 如果满足 `m_trx_undo_rsegs.get_trx_no() == UINT64_UNDEFINED`，表示当前需要处理的 rollback segment 对应的事务并不存在（一条伪记录），那么将会将 purge_queue 顶部的元素赋值给 `m_trx_undo_rsegs`。弹出顶部 rollback segment

- 其次，如果满足 `purge_sys->purge_queue->top().get_trx_no() == m_trx_undo_rsegs.get_trx_no()`，表示 purge_queue 顶部的 rollback segment 对应的事务就是当前待处理的事务，那么将purge_queue 顶部的 rollback segment 添加到 `m_trx_undo_rsegs`。弹出顶部 rollback segment

- 否则，表示当前待处理的 rollback segment 集合对应的事务并不等于 purge_queue 顶部事务，跳出循环