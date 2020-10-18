## WAL
mysql 在更新时，如果每一次更新都写进磁盘，需要磁盘找到对应记录，然后再更新，IO 成本特别高。因此 mysql 采用 WAL 技术，WAL 的全称是 Write-Ahead Logging，关键在于写日志，再写磁盘

当有数据更新时，先将记录写到 redo log 中，并更新内存。存储引擎会在适当的时候，将这个操作记录更新到磁盘里面，这通常是在空闲的时候进行

由于 redo log 大小固定，如果写入数据过多，需要存储引擎将 redo log 中的一部分操作记录更新到磁盘里面，从而腾出空间记录新的操作记录

redo log 能保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。innodb 通过参数 innodb_flush_log_at_trx_commit 控制 redo log 的写入策略：

```sql
-- 0，表示每次事务提交都只是把 redo log 留在 redo log buffer 中
-- 1，表示每次事务提交都将 redo log 直接持久化到磁盘
-- 2，每次事务提交时都只是把 redo log 写到 page cache

-- redo log buffer -> page cache -> hard disk
show variables like "innodb_flush_log_at_trx_commit";
```

innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。建议设置为 1，保证 MySQL 异常重启之后数据不丢失


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


## redo log 与 binlog
redo log 是 InnoDB 引擎特有的日志，而 binlog 则是 Server 层的日志。最开始 MySQL 只有 MyISAM 引擎，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。而 InnoDB 是另一个公司以插件形式引入 MySQL 的，既然只依靠 binlog 是没有 crash-safe 能力的，所以 InnoDB 使用 redo log 来实现 crash-safe 能力

redo log 与 binlog 都需要的原因：
1. redo log 保证持久性，即使 mysql 崩溃也能恢复之前的提交
2. 由于 redo log 是循环写固定大小的空间，不能持久保存，需要 binlog 的归档功能

这两种日志有以下三点不同：
1. redo log 是 InnoDB 引擎特有的；binlog 则是所有引擎都可以使用的
2. redo log 是物理日志，记录的是物理数据页面的修改的信息；binlog 则是逻辑日志，记录的是语句的原始逻辑，即将 sql 语句按照一定的格式记录
3. redo log 空间大小是固定的；binlog 是可以不断追加写入的，当 binlog 文件达到一定大小之后，就会切换到下一个文件


## update 内部流程
```sql
update T set c = c + 1 where id = 2;
```
1. 执行器通过存储引擎找到 id = 2 的记录，如果该记录所在的数据页在内存中，就直接返回给执行器；否则先从磁盘中读取内存，然后返回
2. 执行器更新记录，然后调用引擎接口写入最新记录
3. 存储引擎将记录更新到内存，同时将更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态，然后告知执行器可以提交事务了
4. 执行器生成该操作的 binlog 日志，并写入磁盘
5. 执行器调用存储引擎接口，将刚刚写入的 redo log 更新为 commit 状态


## 两阶段提交
如果数据需要半个月内可以恢复，那么备份系统中一定会保存最近半个月的所有 binlog，同时系统会定期做整库备份。当需要恢复到指定的某一秒时：
1. 首先，找到最近的一次全量备份，从这个备份恢复到临时库
2. 然后，从备份的时间点开始，将备份的 binlog 依次取出来，重放到中午误删表之前的那个时刻。这样临时库就跟误删之前的线上库一样了
3. 然后把表数据从临时库取出来，按需要恢复到线上库去

由于 redo log 和 binlog 是两个独立的逻辑，如果不用两阶段提交，就会出现问题。假设当前 id = 2 的行，字段 c 的值是 0，再假设执行 update 语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了 crash

1. 先写 redo log 后写 binlog。redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。但是由于 binlog 里面就没有记录这个语句，如果需要用这个 binlog 来恢复临时库的话，临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0
2. 先写 binlog 后写 redo log。由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了这次更新

数据更新过程有 prepare 阶段与 commit 阶段之后：
1. 如果在写 prepare 之后 mysql 异常重启，重启之后没有 commit，回滚；备份恢复也没有对应 binlog
2. 如果在 commit 之前 mysql 异常重启，虽然没有 commit，但是 redo log 与 binlog 完整，重启之后会自动 commit；备份恢复也有对应 binlog
