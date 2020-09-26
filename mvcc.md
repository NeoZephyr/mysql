## 快照读
读取的是快照数据。不加锁的简单的 SELECT 都属于快照读
```sql
SELECT * FROM player WHERE id = 1;
```


## 当前读
读取最新数据，而不是历史版本的数据。加锁的 SELECT，或者对数据进行增删改都会进行当前读
```sql
SELECT * FROM player LOCK IN SHARE MODE;
```
```sql
SELECT * FROM player FOR UPDATE;
```

快照读就是普通的读操作，而当前读包括了加锁的读取和 DML 操作


## MVCC 实现
### 事务版本号
每开启一个事务，都会从数据库中获得一个事务 ID（也就是事务版本号），这个事务 ID 是自增长的，通过 ID 大小可以判断事务的时间顺序

### 隐藏列
InnoDB 的叶子段存储了数据页，数据页中保存了行记录，而在行记录中有一些重要的隐藏字段
db_row_id：隐藏的行 ID，用来生成默认聚集索引。如果没有指定聚集索引，就会用这个隐藏 ID 来创建聚集索引
db_trx_id：操作这个数据的事务 ID，也就是最后一个对该数据进行插入或更新的事务 ID
db_roll_ptr：回滚指针，也就是指向这个记录的 Undo Log 信息

### Undo Log
InnoDB 将行记录快照保存在了 Undo Log 里，可以在回滚段中找到它们

回滚指针将数据行的所有快照记录都通过链表的结构串联了起来，每个快照的记录都保存了当时的 db_trx_id，也是那个时间点操作这个数据的事务 ID。如果我们想要找历史快照，就可以通过遍历回滚指针的方式进行查找

### Read View
在 MVCC 机制中，多个事务对同一个行记录进行更新会产生多个历史快照，这些历史快照保存在 Undo Log 里。Read View 保存了当前事务开启时所有活跃（还没有提交）的事务列表，在 Read View 中有几个重要的属性：
1. trx_ids，系统当前正在活跃的事务 ID 集合
2. low_limit_id，活跃的事务中最大的事务 ID
3. up_limit_id，活跃的事务中最小的事务 ID
4. creator_trx_id，创建这个 Read View 的事务 ID

假设当前有事务 creator_trx_id 想要读取某个行记录，这个行记录的事务 ID 为 trx_id，那么会出现以下几种情况：

1. trx_id < up_limit_id，也就是说这个行记录在这些活跃的事务创建之前就已经提交了，那么这个行记录对该事务是可见的
2. trx_id > low_limit_id，这说明该行记录在这些活跃的事务创建之后才创建，那么这个行记录对当前事务不可见
3. up_limit_id < trx_id < low_limit_id，说明该行记录所在的事务 trx_id 在目前 creator_trx_id 这个事务创建的时候，可能还处于活跃的状态，因此需要在 trx_ids 集合中进行遍历，如果 trx_id 存在于 trx_ids 集合中，证明这个事务 trx_id 还处于活跃状态，不可见。否则，如果 trx_id 不存在于 trx_ids 集合中，证明事务 trx_id 已经提交了，该行记录可见

多版本并发控制下查询一条记录：
1. 获取事务自己的版本号，也就是事务 ID
2. 获取 Read View
3. 查询得到的数据，然后与 Read View 中的事务版本号进行比较
4. 如果不符合 ReadView 规则，就需要从 Undo Log 中获取历史快照
5. 最后返回符合规则的数据

MVCC 通过 Undo Log + Read View 进行数据读取，Undo Log 保存了历史快照，而 Read View 规则用来判断当前版本的数据是否可见

在隔离级别为读已提交时，一个事务中的每一次 SELECT 查询都会获取一次 Read View。这时如果 Read View 不同，就可能产生不可重复读或者幻读的情况

t1
```sql
# A
begin;
```
```sql
# B
begin;
```

t2
```sql
# A
# 排它锁
select * from player where height > 2.08 for update;
```

t3
```sql
# B
insert into player values(100, 103, 'jack', 2.16);
commit;
```

t4
```sql
# A
select * from player where height > 2.08;
commit;
```
在读已提交的情况下，只采用记录锁（Record Locking）


当隔离级别为可重复读的时候，一个事务只在第一次 SELECT 的时候会获取一次 Read View，而后面所有的 SELECT 都会复用这个 Read View，避免了不可重复读

在可重复读的情况下，通过 Next-Key 锁和 MVCC 来解决幻读问题。还是上面的示例，当插入数据的时候，事务 B 会超时，无法插入该数据。这是因为采用了 Next-Key 锁，会将 height > 2.08 的范围都进行锁定，就无法插入符合这个范围的数据了。然后事务 A 重新进行条件范围的查询，就不会出现幻读的情况
