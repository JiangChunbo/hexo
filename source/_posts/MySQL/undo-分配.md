---
title: undo 分配
date: 2022-05-09 20:52:54
tags:
---

分配临时回滚段和持久回滚段的函数名不同只在于末尾的单词不同：

trx_assign_rseg_temp

trx_assign_rseg_durable


当一个事务被判定为读写模式时，会为其分配 trx_id 以及持久回滚段。


分配临时回滚段是当调用 `trx_undo_report_row_operation` 时，判断是否是临时表而动态分配的。