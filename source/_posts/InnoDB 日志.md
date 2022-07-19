---
title: InnoDB 日志
mathjax: true
---
# 参考文献
[解析 roll_pointer](http://blog.itpub.net/7728585/viewspace-2284045/)

[庖丁解牛 ](https://zhuanlan.zhihu.com/p/427911093)

[InnoDB之UNDO LOG介绍](https://zhuanlan.zhihu.com/p/453169285)


# Undo Logs

Undo Log 是一条或者多条 Undo Log Record 的集合，每一条 Undo Log Record 都与一个读写事务相关。每条 Undo Log 记录包含了有关如何撤销事务最新更改的信息[[?]](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-logs.html)。

## 1. Undo Tablespaces
Undo Tablesapces 包含许多 Undo Log。

> MySQL 最多支持 127 个 Undo Tablespace。默认为 2 个。

InnoDB 是一个多版本的存储引擎。它可以保留有关更改行的旧版本的信息，以支持事务功能，例如并发和回滚。此信息以一个称为回滚段的数据结构，存储于 undo 表空间。回滚段驻留在 undo 表空间和全局临时表空间中[[1]](s)。



<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/undo-tablespace.png">


## 2. Rollback Segment
InnoDB 在 Undo Tablespace 中使用 Rollback Segment 来组织 Undo Log，最多支持 128 个 Rollback Segment。

其中第 0 号、33-127号针对普通表设计，1-32 号针对临时表设计。

> 一个事务可能即操作了临时表，也操作了物理表，因此，一个事务是可以使用多个 Rollback Segment。


## 3. Rollback Segment Array Header
Undo Tablespace 文件中的第 3 个 Page 固定作为这 128 个 Rollback Segment 的目录，即 Rollback Segment Array Header


## 4. Rollback Segment Header
通过 Rollback Segment Header 来管理 Rollback Segment，Rollback Segment Header 通常在 Rollback Segment 第 1 页。

<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/rollback_segment_header.png" width="300">

|<div style="width:200px">字段</div>|说明|
|-|-|
|Max Size|参数名 TRX_RSEG_MAX_SIZE， 回滚段可用的最大 Page 数|
|History Size|参数名 TRX_RSEG_HISTORY_SIZE，History List 包含的 Page 数|
|History List Base Node|参数名 TRX_RSEG_HISTORY |


History List 把所有已经提交，但还没有被 purge 的事务的 Undo Log 连接起来，purge 线程可以通过此 List 对已经没有事务使用的 Undo Log 进行 purge。

每个事务在需要记录 Undo Log 时都会申请 1 个或者 2 个 Slot（INSERT 和 UPDATE 分开），同时把事务的第一个 Undo Page 放入对应 Slot 中



### 5. Undo Page
Undo Page 一般可以分为两种：Header Page 和 Normal Page。

<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/undo-page.png">


Undo Header Page 是事务需要写 Undo Log 时申请的第一个 Undo Page


Undo Header Page 是当活跃事务产生的 Undo Record 超过 Undo Header Page 容量后，单独分配的 Undo Page

### 6. Undo Page Header

<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/undo-page-header.png" width="400">

|<div style="width:200px">字段</div>|说明|
|---|----|
|Undo Page Type|TRX_UNDO_PAGE_TYPE，使用该页事务的类型<br>可选值: TRX_UNDO_INSERT、TRX_UNDO_UPDATE |
|Latest Log Record Offset|最新事务开始记录 Undo Log 的位置|
|Free Space Offset|页内空闲空间起始地址，在此之后可记录 Undo Log|
|Undo Page List Node|undo page list节点，可以把同一个事务所用到的所有undo page双向串联起来|

### 6. Undo Segment

InnoDB 中的 Undo Tablespace 中准备了大量的 Undo Segment 槽位，默认按照 1024 一组划分为 Rollback Segment。

> 每个 Undo Tablespace 最多会包含128 个 Rollback Segment。1 个 Undo Slot 对应 1 个 Undo Segment

每个写事务开始写操作之前都需要持有一个 Undo Segment。在任何时刻，每个 Undo Segment 都是被一个事务独占的。

对于较大的 Undo Log 随着不断地写入，按需分配足够多的 Undo Page 分散承载。



每个 Undo Segment  至少持有 1 个 Undo Page，每个 Undo Page 会在开头 38 - 56 字节记录 Undo Page Header。

> Rollback Segment 中 Undo Slot 具体的数值是 $\frac {Page Size}{16}$，见[15.6.6 Undo Logs](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-logs.html)。因为默认 Page Size = 16 KB，因此默认以 1024 一组划分为一个 Rollback Segment。




### 7. Undo Segment Header

Undo Segment 中的第 1 个 Undo Page 还会在 56~86 字节记录 Undo Segment Header。

<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/undo-segment-header.png" style="width: 400px">

|<div style="width:200px">字段</div>|说明|
|---|----|
|State|TRX_UNDO_STATE，Undo Segment 的状态|
|Last Log Offset|TRX_UNDO_LAST_LOG，当前页最后一个 Undo Log Header 的位置|
|Undo Segment FSEG Entry|TRX_UNDO_FSEG_HEADER，segment对应的inode的（space_id，page_no，offset等）|
|Undo Segment Page List Base Node|TRX_UNDO_PAGE_LIST,undo page list的Base Node，对于同一个事务下的undo header page和undo normal page构成双向链表|


TRX_UNDO_PAGE_LIST：对于一般事务来说，不会出现一页写不下的情况，所以，对于大多数事务该链表长度是 1。


在事务结束 (commit / rollback) 的时候，会依次检查一些条件：

```c
// trx_undo_set_state_at_finish()
if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) < 
TRX_UNDO_PAGE_REUSE_LIMIT) {
    // 如果占用 Page == 1，而且本页使用空间偏移量小于 3 / 4
    // 那么，标记为 TRX_UNDO_CACHED
    state = TRX_UNDO_CACHED;
} else if (undo->type == TRX_UNDO_INSERT) {
    // 如果类型为 INSERT
    // 那么，标记为 TRX_UNDO_TO_FREE
    state = TRX_UNDO_TO_FREE;
} else {
    // 最后就是类型为 UPDATE 而且占用空间较多
    state = TRX_UNDO_TO_PURGE;
}
```

对于标记为 `TRX_UNDO_CACHED` 的 Undo Segment 会在 `trx_undo_insert_cleanup` / `trx_undo_update_cleanup` 中添加到 insert cached list / update cached list 头部。


对于 INSERT 类型的清理在 `trx_commit_in_memory()` 会直接释放掉标记为 `TRX_UNDO_TO_FREE` 的 Undo Segment。

UPDATE 类型的 Undo Segment 会等待 Purge 完毕回收。

### 8. Undo Log Header

每个写事务会修改一些数据记录，对应产生一些 Undo Log Record。这些 Undo Log Record 连接在一起形成该事务的 Undo Log。这些 Undo Log Record 开头存在一个 Undo Log Header 记录一些信息。


<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/undo_log_header.png" width="300">


|字段|占用|说明|
|:-|:-|:-|
|Transaction ID|8|事务 ID|
|Delete Mark| 2| 表示该 Undo Log 是否存在 TRX_UNDO_DEL_MARK_REC 类型的 Undo Log Record，避免 Purge 时不必要的扫描|
|Log Start Offset|2|记录 Undo Log Header 的结束位置，便于之后 Header 增加内容时的兼容|
|Next Undo Log|2|后一个 Undo Log|
|Prev Undo|2|前一个 Undo Log|

### 8. Undo Log Record 结构
主要分为两大类：
- insert undo log record
- update undo log record
其中，update undo log record 还有其他更多的类别


#### 8.1. Insert Undo Log Record

TRX_UNDO_INSERT_REC
| TRX_UNDO_INSERT_REC | 说明 |
|:-|:-|
| next (2)  | 下一个 undo log 的位置
|  type_cmpl (1) | Undo 类型，TRX_UNDO_INSERT_REC: 11
| Undo Number | 在一个事务中从 0 开始递增 |
| Table ID |
| Key Field 1 Length |
| Key Field 1 Content |
| ... |
| Key Field n Length |
| Key Field n Content|
| start | undo 开始的位置
> INSERT 操作的 undo log record 在事务提交后就可以删除



#### 8.2. Update Undo Log Record

该类别的 Undo Log Record 可以再分为三种：

- TRX_UNDO_DEL_MARK_REC
- TRX_UNDO_UPD_DEL_REC
- TRX_UNDO_UPD_EXIST_REC

TRX_UNDO_UPD_EXIST_REC
| <div style="width: 200px">字段</div> | 占用 | 说明 |
|:-|:-|:-|
|end of record  | 2 | 本页中，该记录的末尾偏移量。只有当记录完全写完才能写入，事先不知道大小。 |
| type_cmpl | 1 | TRX_UNDO_UPD_EXIST_REC|
| undo_no | |在一个事务中从 0 开始递增 |
| table id | | 表 ID
| info bits  | 1 | 
| trx_id | 压缩 | 旧记录的 trx_id |
| roll_pointer | 压缩 | 旧记录的 roll_pointer |
| clustered index 1 length | | 聚簇索引 1 长度 |
| clustered index 1 value | | 聚簇索引 1 值|
| ... | ...| ...|
| clustered index n length||聚簇索引 n 长度|
| clustered index n value||聚簇索引 n 值|
| n_updated | | 共有多少个列被更新了 |
| len of index_col_info |
| 索引列各列信息 |
| start of record | 2 | 本页中，该记录的起始偏移量


TRX_UNDO_DEL_MARK_REC
| <div style="width: 200px">字段</div>  | 占用 | 说明 |
|:-|:-|:-|
| end of record (2) |||
|  Type and Flags (1) || TRX_UNDO_DEL_MARK_REC|
| Undo Number | |在一个事务中从 0 开始递增 |
| Table ID |
| Info Bits  |
| trx_id | | 旧记录的 trx_id |
| roll_pointer || 旧记录的 roll_pointer |
| clustered index 1 length | | 聚簇索引 1 长度 |
| clustered index 1 value | | 聚簇索引 1 值|
| ... | ...| ...|
| clustered index n length||聚簇索引 n 长度|
| clustered index n value||聚簇索引 n 值|
| start of record |
事务中 DELETE 仅将记录的 deleted_flag 标识设置为 1
当对每一条数据记录进行 delete mark 操作前，需要把该数据记录的 trx_id 和 roll_pointer 的旧值记录到 undo log record，再将 trx_id 和 roll_pointer 更新。



> 撤销日志是为了实现事务原子性而出现的产物。事务处理过程中，如果出现了错误或者用户执行了 rollback 语句，MySQL 可以利用 undo log 中的信息将数据恢复到事务开始之前的状态
> 撤销日志在 MySQL InnoDB 存储引擎中还用来实现多版本并发控制。

在全局临时表空间中的 Undo Log 用于事务修改用户定义的临时表中的数据。这些 Undo Log 不会记录 Redo Log，因为崩溃恢复不需要它们。它们仅在服务器运行时用于回滚。这种类型的 Undo Log 通过避免 Redo 日志 I/O 对性能有帮助。
每个 undo 表空间和全局临时表空间最多支持 128 个回滚段。`innodb_rollback_segments` 变量定义了回滚段的数量。
事务最多分配 4 个 undo 日志，每个对应下面的操作类型：
1. `INSERT` 用户定义的表
2. `UPDATE` 和 `DELETE` 用户定义的表
3. `INSERT` 用户定义的临时表
4. `UPDATE` 和 `DELETE` 用户定义的临时表
根据需要分配 undo 日志。例如，执行常规表和临时表上的 `INSERT`，`UPDATE`，以及 `DELETE` 操作的事务需要完全分配 4 个 undo 日志；仅在常规表上执行 `INSERT` 操作的事务只需要 1 个 undo 日志。
- 如果每个事务执行 `INSERT` 或者 `UPDATE` 或者 `DELETE` 操作之一，那么 InnoDB 可以支持的并发独写事务数是：
```bash
(innodb_page_size / 16) * innodb_rollback_segments * number of undo tablespaces
```
- 如果每个事务执行 `INSERT` 加上 `UPDATE` 或者 `DELETE` 操作之一，那么 InnoDB 可以支持的并发独写事务数是：
```bash
(innodb_page_size / 16 / 2) * innodb_rollback_segments * number of undo tablespaces
```
- 如果每个事务都在临时表上执行 `INSERT` 操作，那么 InnoDB 可以支持的并发独写事务数是：
```bash
(innodb_page_size / 16) * innodb_rollback_segments
```



### Undo Log 分配
当开启一个事务的时候，会调用 `trx_assign_rseg_durable` 分配一个 Rollback Segment。
只读事务
```c
trx_assign_rseg_temp();
-> get_next_temp_rseg();
-> trx_sys->tmp_rsegs
```
读写事务
```c
trx_assign_rseg_durable() 
-> get_next_redo_rseg()
    ->get_next_redo_rseg_from_trx_sys() -> (trx_sys->rsegs)
      get_next_redo_rseg_from_undo_spaces() -> (undo_space->rsegs())
```
当 InnoDB 没有配置独立 Undo Tablespace 时， trx_sys->regs 为读写事务分配回滚段；否则从 undo_spaces->regs() 分配回滚段
当第一次真正产生修改需要写 Undo Log Record 的时候，调用 `trx_undo_assign_undo` 来获得一个 Undo Segment
```c
trx_undo_assign_undo(*trx. *undo_ptr, type) {
    /*
     尝试获取缓存中可用的 Undo Log
     1. 对于 type == TRX_UNDO_INSERT
          从 rseg->insert_undo_cached 链表上获取 Undo Log 对象，并从链表移除
          之后调用 trx_undo_insert_header_reuse 重新初始化 Undo Page Header
     2. 对于 type == TRX_UNDO_UPDATE
          从 rseg->update_undo_cached 链表上获取 Undo Log 对象，并从链表移除
          之后调用 trx_undo_header_create 创建新的 Undo Log Header
    */
          
    undo = trx_undo_reuse_cached();
    if (undo == nullptr) {
// 如果没有缓存的 Undo Log 对象，调用 trx_undo_create 从回滚段上分配一个空闲的 Undo Slot
        trx_undo_create();
    }
}
```
### Undo Log 写入

#### 1. 分配回滚段

事务从调用 `trx_start_low` 函数开始。

当该事务被判定为读写模式时，会分配 TRX_ID 以及回滚段。
```c
// 来自 trx_start_low 片段
if (!trx->read_only &&
      (trx->mysql_thd == nullptr || read_write || trx->ddl_operation)) {
    // 分配 Rollback Segment
    trx_assign_rseg_durable(trx);
    // 分配 TRX_ID
    trx->id = trx_sys_allocate_trx_id();
}
```


当写事务开始时，会先调用 `trx_assign_rseg_durable`  分配一个 Rollback Segment。

分配策略：依次尝试下一个活跃的 Rollback Segment。

```c
/ Assign a durable rollback segment to a transaction in a round-robin
fashion.
@param[in,out]	trx	transaction that involves a durable write. */
void trx_assign_rseg_durable(trx_t *trx) {
  ut_ad(trx->rsegs.m_redo.rseg == nullptr);

  trx->rsegs.m_redo.rseg = srv_read_only_mode ? nullptr : get_next_redo_rseg();
}
```

#### 2. 使用回滚段

当第一次真正产生修改需要写 Undo Record 的时候，会从 `trx_undo_report_row_operation` 进入，接着调用 `trx_undo_assign_undo` 获得一个 Undo Segment。优先复用 `trx_rseg_t` 上 Cached List 中的 trx_undo_t，也就是已经分配出来但没有被正在使用的 Undo Segment。



如果没有缓存的 Undo Segment，才调用 `trx_undo_create` 创建新的 Undo Segment，`trx_undo_create` 会轮询选择当前 Rollback Segment 中可用的 Slot，申请新的 Undo Page，初始化 Undo Page Header，Undo Segment Header


#### 3. 写入

对于 INSERT UNDO LOG 写入的入口函数 `trx_undo_page_report_insert`

对于 UPDATE UNDO LOG 写入的入口函数 `trx_undo_page_report_modify`



在写入过程中，可能出现 Undo Page 空间不足的情况，当出现这种情况，会调用 `trx_undo_erase_page_end` 来清除刚刚写入的区域，然后调用 `trx_undo_add_page` 申请一个新的 Undo Page 加入到 Undo Page List，同时 undo->last_page_no 指向新的 Undo Page，重新尝试写入。



# 表空间
## 系统表空间 
The System Tablespace

系统表空间是更改缓冲区的存储区域。如果在系统表空间中创建表，而不是每张表一个文件或者常规表空间，它也可能包含表和索引数据。在过于的版本中，系统表空间包含 InnoDB 的数据字典。在 MySQL 8.0 中，InnoDB 将元数据存储在数据字典中。
系统表空间可以有一个或者多个数据文件。默认地，会在数据文件夹下创建一个系统表空间数据文件，名为 `ibdata`。系统表空间的大小和数量由 innodb_data_file_path 启动项定义。

## File-Per-Table Tablespaces
对于单个 InnoDB 表，file-per-table 表空间包含了该表的数据以及索引，并存储于文件系统的单个文件中
# Redo Log 重做日志用来实现事务的持久性           
Redo Log 是在崩溃期间使用的基于磁盘的数据结构，以纠正不完整事务写入的数据。
在正常操作期间，Redo Log 将那些来自于 SQL 语句或者低级 API 调用的表数据修改操作请求进行编码。
在初始化并接受连接之前，那些由于无法预期的关闭导致未能将数据文件更新的修改操作会被重新执行。
> 重做日志用来实现事务的持久性
默认地，Redo Log 在物理上表现为磁盘上两个名为 `ib_logfile0` 和 `ib_logfile1` 的文件。MySQL 以循环的方式写入 Redo Log 文件。根据受影响的记录，Redo Log 将它们编码；这些数据统称为 redo。
## Changing the Number or Size of Redo Log Files
如果要修改 Redo Log 的大小数量，需要执行以下步骤：
1. 停止 MySQL Server 并确保它没有错误关闭
2. 编辑 my.cnf 更改日志文件配置。要更改日志文件大小，配置 `innodb_log_file_size`。为了增加日志文件的数量，需配置 `innodb_log_files_in_group`
3. 再次启动 MySQL 服务
## Group Commit for Redo Log Flushing
与其他符合 ACID 数据库引擎一样，InnoDB 在提交事务之前会刷写（flush） Redo Log。InnoDB 使用组提交功能，将多个 flush 请求组合在一起，以避免为每个提交进行一次 flush 操作。使用组提交，InnoDB 向日志文件发出单个的写入，用于为同一时间的多个用户事务执行提交动作，这可以显著提高吞吐量。
## Redo Log Archiving
复制 Redo Log 记录的备份工具有时候可能会在进行备份操作时无法跟上 Redo Log 的生成速度，导致由于这些记录被覆盖而导致 Redo Log 记录丢失。在备份操作期间，存在着显著的 MySQL 服务活动，并且 Redo Log 文件存储介质比备份存储介质更快的速度运行时，最常常发生此问题。在 MySQL 8.0.17 中引入的重做记录归档功能，通过在 Redo Log 文件之外将 Redo Log 记录顺序写入归档文件来解决此事。
## Performance Considerations
![请添加图片描述](https://img-blog.csdnimg.cn/55d9b5977dcd478b9035dc0faee7a8fc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA572Q6KOF6Z2i5YyF,size_20,color_FFFFFF,t_70,g_se,x_16)



# Undo Tablespaces
撤销表空间包含 undo 日志，这些记录是包含有关如何撤销事务的最新的更改的信息。
InnoDB 是一个多版本的存储引擎。它可以保留有关更改行的旧版本的信息，以支持事务功能，例如并发和回滚。此信息以一个称为回滚段的数据结构，存储于 undo 表空间。回滚段驻留在 undo 表空间和全局临时表空间中。
## Default Undo Tablespaces
初始化 MySQL 实例的时候，会创建两个默认的 undo 表空间。
默认 undo 表空间的创建位置由 innodb_undo_directory 变量定义。如果 innodb_undo_directory 变量未定义，则再数据目录中创建默认的 undo 表空间。默认 undo 表空间数据文件名为 `undo_001` 和 `undo_002`。数据字典中定义的相应的 undo 表空间名称是 innodb_undo_001 和 innodb_undo_002
## Undo Tablespace Size
在 MySQL 8.0.23 之前，undo 表空间的大小取决于 innodb_page_size。对于默认的 16K 页大小，初始 undo 表空间是 10MB。

## Dropping Undo Tablespaces

MySQL 8.0.14 可以使用 `DROP UNDO TABLESPACES` 语法在运行时删除使用 `CREATE UNDO TABLESPACES` 语法创建的表空间。


## MVCC

ReadView，每个事务在读取数据的时候都会被分配一个视图，通过视图就可以判断其他事务对数据的可见性。

分配：通过 `trx_assign_read_view()` 分配视图

回收：事务结束时，会通过 `view_close()` 对其视图进行回收。

`m_low_limit_id`：读取行为不应该看到 `trx_id` >= `m_low_limit_id` 的事务，即高水位。分配时取 `trx_sys::max_trx_id`，即当前还没有被分配的事务最大 ID

`m_up_limit_id`：读取行为应该可以看到所有 trx_id < `m_up_limit_id` 的事务，即低水位。低水位，如果m_ids不为空，取其最小值，否则取trx_sys::max_trx_id，即与高水位相等。

> 关于 `m_low_limit_id` 和 `m_up_limit_id` 的解释以及高水位和低水位的比喻均来自于源码注释。

`m_ids`：在此视图初始化时，通过 `copy_trx_ids()` 从 `trx_sys::rw_trx_ids` 拷贝一份活跃事务ID(不包含当前事务ID)。