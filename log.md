## `redo log`
mysql 在更新时，如果每一次更新都写进磁盘，需要磁盘找到对应记录，然后再更新，IO 成本特别高。因此 mysql 采用 WAL 技术，关键就在先写日志，再写磁盘。

当有数据更新时，先将记录写到 `redo log` 中，并更新内存。存储引擎会在适当的时候，将这个操作记录更新到磁盘里面，这通常是在空闲的时候进行。不过，由于 `redo log` 大小固定，如果写入数据过多，页需要存储引擎将 `redo log` 中的一部分操作记录更新到磁盘里面，从而腾出空间记录新的操作记录

`redo log` 能保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 `crash-safe`。innodb 通过参数 `innodb_flush_log_at_trx_commit` 控制 `redo log` 的写入策略：
```sql
-- 0，表示每次事务提交都只是把 redo log 留在 redo log buffer 中
-- 1，表示每次事务提交都将 redo log 直接持久化到磁盘
-- 2，每次事务提交时都只是把 redo log 写到 page cache

-- redo log buffer -> page cache -> hard disk
show variables like "innodb_flush_log_at_trx_commit";
```
`innodb_flush_log_at_trx_commit` 这个参数设置成 1 的时候，表示每次事务的 `redo log` 都直接持久化到磁盘。建议设置为 1，保证 MySQL 异常重启之后数据不丢失


## `binlog`
`redo log` 是 InnoDB 引擎特有的日志，`binlog` 则是 Server 层的日志。两者有以下不同：
1. `redo log` 是 InnoDB 引擎特有的；`binlog` 则是所有引擎都可以使用的
2. `redo log` 是物理日志，记录的是物理数据页面的修改的信息；`binlog` 则是逻辑日志，记录的是语句的原始逻辑，即将 sql 语句按照一定的格式记录
3. `redo log` 空间大小是固定的；`binlog` 是可以不断追加写入的，当 `binlog` 文件达到一定大小之后，就会切换到下一个文件

`binlog` 的写入逻辑比较简单：事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 `binlog` 文件中。一个事务的 `binlog` 是不能被拆开的，不论这个事务多大，都要确保一次性写入。每个线程都拥有自己的 binlog cache，大小由参数 `binlog_cache_size` 控制
```sql
show variables like "%binlog_cache_size%";
```
事务提交的时候，执行器把 binlog cache 里的完整事务写入到 `binlog` 中，并清空 binlog cache。写入 `binlog` 持久化由参数 `sync_binlog` 控制：
```sql
-- 0，表示每次提交事务都把日志写入到 page cache，并且持久化到磁盘
-- 1，表示每次提交事务都持久化到磁盘
-- n，表示每次提交事务都把日志写入到 page cache，但积累到 n 个事务后才持久化到磁盘
show variables like "sync_binlog";
```
建议设置为 1，保证 MySQL 异常重启之后 `binlog` 不丢失

数据更新流程
```sql
update T set c = c + 1 where id = 2;
```
1. 执行器通过存储引擎找到 `id = 2` 的记录，如果该记录所在的数据页在内存中，就直接返回给执行器；否则先从磁盘中读取内存，然后返回
2. 执行器更新记录，然后调用引擎接口写入最新记录 
3. 存储引擎将记录更新到内存，同时将更新操作记录到 `redo log` 里面，此时 `redo log` 处于 prepare 状态，然后告知执行器可以提交事务了
4. 执行器生成该操作的 `binlog` 日志，并写入磁盘
5. 执行器调用存储引擎接口，将刚刚写入的 `redo log` 更新为 commit 状态

数据更新过程需要 prepare 阶段与 commit 阶段的原因是，防止写完一个日志后，写另一个日志之前发生 crash。比如：
1. 先写 `redo log`，之后 mysql 异常重启，这样 `binlog` 中就没有记录更新操作，之后如果需要用 `binlog` 恢复数据，就会少了这次更新
2. 先写 `binlog`，之后 mysql 异常重启，这样 `redo log` 没有写入，这样崩溃恢复之后，之前的更新操作无效。但由于 `binlog` 记录了，所以之后用 `binlog` 恢复数据的时候，就会多恢复一次更新

数据更新过程有 prepare 阶段与 commit 阶段之后：
1. 如果在写 prepare 之后 mysql 异常重启，重启之后没有 commit，回滚；备份恢复也没有对应 `binlog`
2. 如果在 commit 之前 mysql 异常重启，虽然没有 commit，但是 `redo log` 与 `binlog` 完整，重启之后会自动 commit；备份恢复也有对应 `binlog`

`redo log` 与 `binlog` 都需要的原因：
1. `redo log` 保证持久性，即使 mysql 崩溃也能恢复之前的提交
2. 由于 `redo log` 是循环写固定大小的空间，不能持久保存，需要 `binlog` 的归档功能

## flush
innodb 在数据更新时，只做了写日志这一个磁盘操作，即 `redo log`，然后更新内存就返回了。可以看出，内存页与数据页内容可能不一致，即出现脏页。而将内存中的数据更新到磁盘的过程，就叫刷脏页，也即是 flush 操作。一般在以下情况，会进行刷脏页操作：
1. 因为 `redo log` 空间大小固定，当写满时，此时会停止所有更新操作，通过刷脏页留出空间继续写
2. 内存不足，需要淘汰一些内存页，如果淘汰的是脏页，就需要将脏页写回磁盘。这样保证了每个数据页只有两个状态：在内存中的数据页，是最新的数据，可以直接返回；内存中没有数据，就可以肯定磁盘上面是最新的数据
3. mysql 空闲的时候刷脏页
4. mysql 正常关闭，会将内存中的脏页都刷新到磁盘上

第一种情况要尽量避免的，因为这种情况下，整个系统不能再接受更新了，此时更新次数跌为 0
第二种是比较常见的现象。innodb 用缓冲池（buffer pool）管理内存，缓冲池中的内存页有以下三种状态：
1. 还未使用的内存页
2. 使用了仍是干净的数据页
3. 使用了已经是脏页

innodb 的策略是尽量使用内存，但是每当要读入的数据页不在内存中的时候，就必须要到缓冲池中申请一个数据页。此时需要把最久不使用的数据页从内存中淘汰掉：如果要淘汰的是一个干净页，就直接释放出来复用；但如果是脏页，就必须先刷到磁盘，变成干净页后才能复用。因此长时间运行后，未必使用的页面很少

根据以上分析可以得知，尽管刷脏页是常见现象，但是出现以下情况，会明显影响性能：
1. 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长
2. 日志写满，更新全部堵住，写性能跌为 0

我们可以通过控制脏页比例来避免明显影响性能的刷脏页情况：
1. 正确地告诉 innoDB 所在主机的 IO 能力
```sql
-- 建议你设置成磁盘的 IOPS，设置太小可能会导致刷脏页特别慢，进而导致内存脏页太多，redo log 写满
show variables like "innodb_io_capacity";
```
2. 根据脏页比例级 `redo log` 写入速度控制刷脏页速度
```sql
-- 脏页比例上限，默认为 75%
show variables like "innodb_max_dirty_pages_pct";
```

查看脏页比例
```sql
select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
select @a/@b;
```

如果一个查询在执行过程中需要 flush 掉一个脏页，这个查询就比较慢。但 mysql 有一个机制：在刷脏页的时候，如果这个数据页旁边的数据页也是脏页，那么就会一起刷掉。在 innodb 中，`innodb_flush_neighbors` 参数为 1 表示脏页刷新具有传递性，为 0 则表示刷新不会影响邻居脏页
```sql
show variables like "innodb_flush_neighbors";
```

刷新邻居脏页的特性在机械硬盘时代，可以减少很多随机 IO。如果使用的是 SSD 这种 IOPS 比较高的设备，可以将 `innodb_flush_neighbors` 设置为 0，这样就能更快的执行刷脏页操作，减少 sql 语句的响应时间。在 mysql8.0 中，`innodb_flush_neighbors` 参数的默认值已经设置为 0 了










### 释放
`binlog` 的默认是保持时间由参数 `expire_logs_days` 配置

数据库事务提交后，必须将更新后的数据刷到磁盘上，以保证 ACID 特性。磁盘随机写性能较低，如果每次都刷盘，会极大影响数据库的吞吐量。优化方式是，将修改行为先写到 redo 日志里（此时变成了顺序写），再定期将数据刷到磁盘上，这样能极大提高性能

假如某一时刻，数据库崩溃，还没来得及刷盘的数据，在数据库重启后，会重做redo日志里的内容，以保证已提交事务对数据产生的影响都刷到磁盘上



undo log: 保证原子性、一致性
redo log: 保证持久性
bin log: 记录事务提交时的二进制日志


### 作用
确保事务的持久性，防止在发生故障的时间点，尚有脏页未写入磁盘，在重启 mysql 服务的时候，根据 `redo log` 进行重做，从而达到事务的持久性这一特性


### 产生
事务开始之后就产生 `redo log`，`redo log` 的落盘并不是随着事务的提交才写入的，而是在事务的执行过程中，便开始写入 `redo log` 文件中

### 释放
当对应事务的脏页写入到磁盘之后，`redo log` 的使命也就完成了，重做日志占用的空间就可以重用（被覆盖）

### 大小
重做日志有一个缓存区 `Innodb_log_buffer`，`Innodb_log_buffer` 的默认大小为 8M
```sql
show variables like 'innodb_log_buffer_size';
```

## `undo log`
### 作用
保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读

### 内容
逻辑格式的日志

### 产生
事务开始之前，将当前是的版本生成 `undo log`，`undo` 也会产生 `redo` 来保证 `undo log` 的可靠性

### 释放
当事务提交之后，`undo log` 并不能立马被删除，而是放入待清理的链表

