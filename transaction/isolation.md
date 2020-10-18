## 隔离级别
```sql
select @@tx_isolation;
```
```sql
show variables like 'transaction_isolation';
show variables like "tx_isolation";
```
```sql
set transaction isolation level read uncommitted;
set session transaction isolation level read uncommitted;
```

事务参数
```sql
show global variables like "autocommit";
show session variables like "autocommit";
```
```sql
set session autocommit=0;
```

1. 读未提交：一个事务还没提交时，它做的变更就能被别的事务看到
2. 读提交：一个事务提交之后，它做的变更能被其他事务看到
3. 可重复读：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的
4. 串行化：写会加写锁，读会加读锁。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行

```sql
create table T(c int) engine=InnoDB;
insert into T(c) values(1);
```

t1
```
A: 启动事务
A: 查询得到值 1
```
```
B: 启动事务
```

t2
```
B: 查询得到值 1
```

t3
```
B: 将 1 改成 2
```

t4
```
A: 查询得到值 v1
```

t5
```
B: 提交事务
```

t6
```
A: 查询得到值 v2
```

t7
```
A: 提交事务
```

t8
```
A: 查询得到值 v3
```

1. 若隔离级别是读未提交， 则 v1 的值就是 2。这时候事务 B 虽然还没有提交，但是结果已经被 A 看到了。因此，v2、v3 也都是 2
2. 若隔离级别是读提交，则 v1 是 1，v2 的值是 2。事务 B 的更新在提交后才能被 A 看到。所以，v3 的值也是 2
3. 若隔离级别是可重复读，则 v1、v2 是 1，v3 是 2。之所以 v2 还是 1，是因为事务在执行期间看到的数据必须前后一致
4. 若隔离级别是串行化，则在事务 B 执行将 1 改成 2 的时候，会被锁住。直到事务 A 提交后，事务 B 才可以继续执行。所以从 A 的角度看， v1、v2 值是 1，v3 的值是 2


## 隔离实现
数据库里面会创建一个视图，访问的时候以视图的逻辑结果为准

1. 在可重复读隔离级别下，这个视图是在事务启动时创建的，整个事务存在期间都用这个视图
2. 在读提交隔离级别下，这个视图是在每个 SQL 语句开始执行的时候创建的
3. 读未提交隔离级别下直接返回记录上的最新值，没有视图概念
4. 而串行化隔离级别下直接用加锁的方式来避免并行访问

在 MySQL 中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。假设一个值从 10 被按顺序改成了 20、30、40，在回滚日志里面就会有类似下面的记录：
1. 将 40 改成 30
2. 将 30 改成 20  read-view B
3. 将 20 改成 10  read-view A

当前值是 40，但是在查询这条记录的时候，不同时刻启动的事务会有不同的 read-view，即同一条记录在系统中可以存在多个版本，也就是数据库的多版本并发控制。即使现在有另外一个事务正在将 4 改成 5，这个事务跟 read-view A、B、C 对应的事务是不会冲突的

回滚日志不是一直保存的，当没有事务再需要用到这些回滚日志时，也就是说系统里面没有比这个回滚日志更早的 read-view 的时候，回滚日志就会被删除。因此要尽可能避免长事务，因为长事务意味着系统里面会存在很老的视图，在提交之前，数据库里面它可能用到的回滚记录都必须保留，这样会占用大量的存储空间。此外，长事务还占用锁资源，也可能拖垮整个库

在 MySQL 5.5 及以前的版本，回滚日志是跟数据字典一起放在 ibdata 文件里的，即使长事务最终提交，回滚段被清理，文件也不会变小

查询持续时间超过 60s 的事务
```sql
select * from information_schema.innodb_trx
where TIME_TO_SEC(timediff(now(), trx_started)) > 60;
```



通过以下面几个示例说明事务隔离的实现原理：
```sql
CREATE TABLE `t` (
    `id` int(11) NOT NULL,
    `k` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;

insert into t (1, 1), (2, 2);
```
有三个事务，分别对表 t 进行操作，执行流程如下
t1 时刻:
```sql
-- 事务 A
start transaction with consistent snapshot;
```
t2 时刻:
```sql
-- 事务 B
start transaction with consistent snapshot;
```
t3 时刻:
```sql
-- 事务 C
-- 产生版本 (1, 2)
update t set k=k+1 where id = 1;
```
t4 时刻:
```sql
-- 事务 B
-- 产生版本 (1, 3)
update t set k=k+1 where id = 1;

-- 结果是 3
select k from t where id = 1;
```
t5 时刻:
```sql
-- 事务 A
-- 结果是 1
select k from t where id = 1;
commit;
```
t6 时刻:
```sql
-- 事务 B
commit;
```

需要明确一点，`begin/start transaction` 命令并不是一个事务的起点，在执行到它们之后的第一个操作 InnoDB 表的语句时，事务才真正启动。可以通过 `start transaction with consistent snapshot` 这个命令马上启动一个事务

事务 C 没有显示地使用 `begin/commit`，表示这个更新语句本身就是一个事务，执行完成后自动提交

启动一个数据库事务，会开启一个事务视图。除了自己的更新总是可以看见以外，还有三种情况：
1. 版本未提交，不可见
2. 版本已提交，但是在视图创建后提交的，不可见
3. 版本已提交，而且是视图创建前提交的，可见

对于事务 A 而言，事务 B 产生的版本还未提交，不可见；事务 C 产生的版本虽然提交，但却是在视图创建前后提交的，不可见
对于事务 B 而言，事务 C 产生的版本是在视图创建后提交的，理论上不可见，但是事务 B 做的事更新操作，更新操作都是先读后写，而且是当前读，所以能看到事务 C 产生的版本数据

关于当前读，除了 `update` 语句外，`select` 语句如果加锁，也是当前读，示例如下：
```sql
-- 加读锁，即共享锁，S 锁
select k from t where id = 1 lock in share mode;

-- 写锁，即拍他锁，X 锁
select k from t where id = 1 for update;
```

现在更改事务 A, B, C 三个事务执行流程如下：
t1 时刻:
```sql
-- 事务 A
start transaction with consistent snapshot;
```
t2 时刻:
```sql
-- 事务 B
start transaction with consistent snapshot;
```
t3 时刻:
```sql
-- 事务 C
start transaction with consistent snapshot;
update t set k=k+1 where id = 1;
```
t4 时刻:
```sql
-- 事务 B
update t set k=k+1 where id = 1;
select k from t where id = 1;
```
t5 时刻:
```sql
-- 事务 A
select k from t where id = 1;
commit;

-- 事务 C
commit;
```
t6 时刻:
```sql
-- 事务 B
commit;
```
与之前流程不同的是，事务 C 更新数据之后没有马上提交。此时数据行 (1, 2) 写锁还没有释放，而事务 B 更新需要当前读，也要加锁，因此必须等待事务 C 释放锁才能继续进行














### READ UNCOMMITED
1. 普通 `select` 使用快照读，不加锁。该隔离级别下，没有视图概念，读取数据时直接返回记录上的最新值
```sql
select * from user where id = 1;
```

### READ COMMITED
1. 普通 `select` 使用快照读，不加锁。该隔离级别下，快照读总是能读取到已提交事务写入的数据，即事务在每次读操作时，都会建立 Read View
```
A1: start transaction;
B1: start transaction;
A2: select * from user;
B2: insert into user values (4, 'jack');
A3: select * from user;
B3: commit;
A4: select * from user;
```
```
A2 读到的结果集是 {1, 2, 3}
A3 读到的结果集也是 {1, 2, 3}，因为 事务 B 还没有提交
A4 读到的结果集是 {1, 2, 3, 4}，因为事务 B 已经提交
```

2. 加锁 `select` 以及 `update`、`delete`，除了在外键约束检查以及重复键检查时会封锁区间，其他时刻都只使用记录锁
```sql
select * from user where id = 1 for update;
select * from user where id = 1 in share mode;
```

### REPEATABLE READ
1. 普通 `select` 使用快照读，不加锁。该隔离级别下，首次读记录的时间为 T，此后的无法读取到 T 时间之后事务提交写入的数据，即事务在首次读操作时建立 Read View
```
A1: start transaction;
B1: start transaction;
A2: select * from user;
B2: insert into user values (4, 'jack');
A3: select * from user;
B3: commit;
A4: select * from user;
```
```
A2 读到的结果集肯定是 {1, 2, 3}，这是事务 A 的首次读取，假设为时间 T
A3 读到的结果集也是 {1, 2, 3}，因为 事务 B 还没有提交
A4 读到的结果集还是 {1, 2, 3}，因为事务 B 是在时间 T 之后提交的，A4 得读到和 A2 一样的数据
```

2. 加锁的 `select` 以及 `update`、`delete`
```sql
# 在唯一索引上使用唯一查询条件，使用记录锁，不会封锁记录之间的间隔
select * from user where id = 1 for update;
select * from user where id = 1 in share mode;

update user set name = "pain" where id = 1;
```
```sql
# 其他的查询条件和索引条件，会封锁被扫描的索引范围，并使用间隙锁与临键锁，避免索引范围区间插入记录
select * from user where id < 5 for update;
```

>需要说明的是，如果更新的是聚集索引记录，则对应的普通索引记录也会被隐式加锁。这是由 InnoDB 索引的实现机制决定的：普通索引存储 PK 的值，检索普通索引需要扫描聚集索引

3. <b>mysql 默认隔离级别</b>

### SERIALIZABLE
1. 普通 `select` 转换为 `select ... in share mode`，与 `update`、`delete` 操作互斥
```sql
select * from user where id = 1;
```

#### case1
事务 A 先执行，且处于未提交状态；此时由于事务 A 在 `id=1` 记录上面加了行所，因此事务 B 会阻塞
```sql
# 事务 A
begin;
update user set name = 'jack01' where id = 1;
```
```sql
# 事务 B
begin;
update user set name = 'jack02' where id = 2;
```

#### case2
事务 A 先执行，删除一条不存在的记录，且处于未提交状态；事务 B 会阻塞。在该隔离级别下，事务 A 使用 `next-keylock` 的锁算法，相当于 `gaplock + recordlock` 的组合
```sql
# 事务 A
begin;
delete from user where id = 10;
```
```sql
# 事务 B
begin;
insert into user values(10, 'polobo');
```

```sql
# 事务 A
begin;
delete from user where id = 20;
```
```sql
# 事务 B
begin;
insert into user values(15, 'nojo');
```

```sql
# 事务 A
begin;
delete from user where id = 30;
```
```sql
# 事务 B
begin;
insert into user values(35, 'copo');
```






## 隔离级别
InnoDB 里面每个事务有一个唯一的事务 ID，叫作 `transaction id`。它是在事务开始的时候向 InnoDB 的事务系统申请的，是按申请顺序严格递增的

mysql 中一条记录有多个版本，每次事务更新数据的时候，都会生成一个新的数据版本，并且把 `transaction id` 赋值给这个数据版本，记为 `row_trx_id`。每个事务都有自己的一致性视图，普通查询根据 `row_trx_id` 和一致性视图决定数据版本的可见性

对于读提交与可重复读隔离级别，有如下结论
1. 可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务中的其他查询都共用这个一致性视图。查询只承认在事务启动前就已经提交完成的数据
2. 在读提交隔离级别下，每一个语句执行前都会重新算出一个新的视图，因此 `start transaction with consistent snapshot;` 就相当于普通的 `start transaction` 语句。查询只承认在语句启动前就已经提交完成的数据
3. 当前读，总是读取已经提交完成的最新版本

表结构不支持“可重复读”，这是因为表结构没有对应的行数据，也没有 `row_trx_id`，因此只能遵循当前读的逻辑


```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
insert into t(id, c) values(1,1),(2,2),(3,3),(4,4);
```
```sql
begin;
select * from t;
update t set c = 0 where id = c;

-- 出现无法更新的问题
-- 由于更新是当前读，读取数据的时候，数据已经被修改过了
select * from t;
```

情况一
t1 时刻：
```sql
-- session A
begin;
select * from t;
```
t2 时刻：
```sql
-- session B
update t set c = c + 1;
```
t3 时刻：
```sql
-- session A
update t set c = 0 where id = c;
select * from t;
```

情况二
t1 时刻：
```sql
-- session B
begin;
select * from t;
```
t2 时刻：
```sql
-- session A
begin;
select * from t;
```
t3 时刻：
```sql
-- session B
update t set c = c + 1;
commit;
```
t4 时刻：
```sql
-- session A
update t set c = 0 where id = c;
select * from t;
```





