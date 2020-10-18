持有 Gap 锁：
请求 Gap 兼容，Insert Intention 冲突，Record 兼容，Next-Key 兼容

持有 Insert Intention 锁：
请求 Gap 冲突，Insert Intention 兼容，Record 兼容，Next-Key 冲突

持有 Record 锁：
请求 Gap 冲突，Insert Intention 兼容，Record 冲突，Next-Key 冲突

持有 Next-Key 锁：
请求 Gap 兼容，Insert Intention 兼容，Record 冲突，Next-Key 冲突


## 锁分类
### 粒度分类
从锁定对象的粒度大小来对锁进行划分，分别为行锁、页锁和表锁。每个层级的锁数量是有限制的，因为锁会占用内存空间，锁空间的大小是有限的。当某个层级的锁数量超过了这个层级的阈值时，就会进行锁升级。锁升级就是用更大粒度的锁替代多个更小粒度的锁，比如 InnoDB 中行锁升级为表锁，这样做的好处是占用的锁空间降低了，但同时数据的并发度也下降了

### 管理分类
从数据库管理的角度对锁进行划分，有共享锁和排它锁。共享锁也叫读锁或 S 锁，共享锁锁定的资源可以被其他用户读取，但不能修改。在进行 SELECT 的时候，会将对象进行共享锁锁定，当数据读取完毕之后，就会释放共享锁，这样就可以保证数据在读取时不被修改。排它锁也叫独占锁、写锁或 X 锁。排它锁锁定的数据只允许进行锁定操作的事务使用，其他事务无法对已锁定的数据进行查询或修改

表加共享锁
```sql
LOCK TABLE product_comment READ;
```
```sql
UPDATE product_comment SET product_id = 10002 WHERE user_id = 912178;
```
```sql
UNLOCK TABLE;
```

行加共享锁
```sql
SELECT comment_id, user_id FROM product_comment WHERE user_id = 912178 LOCK IN SHARE MODE;
```

表加独占锁
```sql
LOCK TABLE product_comment WRITE;
```
```sql
UNLOCK TABLE;
```

行加独占锁
```
SELECT comment_id, product_id, comment_text, user_id FROM product_comment WHERE user_id = 912178 FOR UPDATE;
```

进行 INSERT、DELETE 或者 UPDATE 操作的时候，会自动使用排它锁，防止其他事务对该数据行进行操作

意向锁（Intent Lock），就是给更大一级别的空间示意里面是否已经上过锁

如果给某一行数据加上了排它锁，数据库会自动给更大一级的空间，比如数据表加上意向锁，表示这个数据表已经有人上过排它锁了，这样当其他人想要获取数据表排它锁的时候，只需要了解是否有人已经获取了这个数据表的意向排他锁即可，而不需要对数据表中的行逐一排查，检查是否有行锁。如果事务想要获得数据表中某些记录的共享锁，就需要在数据表上添加意向共享锁。同理，事务想要获得数据表中某些记录的排他锁，就需要在数据表上添加意向排他锁

### 使用分类
乐观锁认为对同一数据的并发操作不会总发生，属于小概率事件，不用每次都对数据上锁，也就是不采用数据库自身的锁机制，而是在程序上采用版本号机制或者时间戳机制实现；悲观锁对数据被其他事务的修改持保守态度，会通过数据库自身的锁机制来实现，从而保证数据操作的排它性

乐观锁的版本号机制在表中设计一个版本字段 version，第一次读的时候，会获取 version 字段的取值。然后对数据进行更新或删除操作时，会根据 version 判断是否已经有事务对这条数据进行了更改

乐观锁的时间戳机制也是在更新提交的时候，将当前数据的时间戳和更新之前取得的时间戳进行比较，如果两者一致则更新成功，否则就是版本冲突


## 全局锁
全局锁就是对整个数据库实例加锁，典型使用场景是数据库逻辑备份，即把整个库都 `select` 出来存成文本

mysql 提供了一个加全局读锁的方法，可以通过执行如下命令加锁
```sql
flush tables with read lock;
```
执行该命令之后，整个库就处于只读状态。之后其他线程进行数据更新、建表、修改表结构及更新类事务的提交都会被阻塞。整个库处于只读状态，会有以下影响：
1. 如果在主库上面备份，则备份期间不能执行更新，业务停止
2. 如果在从库上面备份，则备份期间从库不能执行主库同步过来的 `binlog`，会导致主从延迟

虽然有以上缺陷，但是如果不加锁的话，那么备份得到的库就不是一个逻辑时间点，即视图的逻辑不一致

其是，如果数据库存储引擎支持事务，那么还可以通过在可重复读隔离级别下开启一个事务，这样就能拿到一致性视图。执行 `mysqldump` 命令并使用参数 `–single-transaction`，这样导数据之前就会启动一个事务，确保了拿到一致性视图。而由于 MVCC 的支持，这个过程中数据是可以正常更新的

此外，如果需要数据库只读，还可以通过执行如下命令
```sql
set global readonly = true
```
不过，与 FTWRL 命令相比，还是推荐使用 FTWRL 命令，原因如下：
1. `readonly` 可用来做其他逻辑，比如判断一个库是主库还是从库
2. 如果执行 FTWRL 命令之后由于客户端发生异常断开，那么 mysql 会自动释放这个全局锁，整个库回到可以正常更新的状态；而将库设置为 `readonly` 之后，如果客户端异常，数据库会一直保持只读状态，风险较大


## 表锁
```sql
-- 解锁之前，只能读表 t，且其它线程不能写表 t
lock tables t read;

-- 解锁之前，只能读写表 t，不能访问其它表，且其它线程不能读写表 t
lock tables t write;
```
```sql
-- 解锁
unlock tables;
```

表锁一般是在数据库引擎不支持行锁的时候才会被用到的。如果数据库引擎支持事务，可以用 `begin`, `commit` 代替 `lock tables`, `unlock tables`


## 元数据锁（meta data lock，MDL)
MDL 不需要显式使用，在访问一个表的时候会被自动加上。作用是保证读写的正确性。
在 mysql 5.5 版本中引入了 MDL，当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁。读锁之间不互斥，可以有多个线程同时对一张表增删改查；读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行

事务中的 MDL 锁，在语句执行开始时申请，事务中的 MDL 锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。因此，在给表添加字段、修改字段、添加索引，都要特别注意：
sessionA
```sql
begin;
select * from user limit 1;
```
sessionB
```sql
select * from user limit 1;
```
sessionC
```sql
alter user add column source varchar(32);
```
sessionD
```sql
select * from user limit 1;
```

可以看到 sessionA 先启动获取 MDL 读锁；此后 sessionB 也需要的是读锁，可以顺利执行；sessionC 需要的是写锁，会被阻塞；之后的 sessionD 虽然申请的是读锁，但是会被 sessionC 阻塞，这样导致后续的增删改查操作都被阻塞，这样表就完全不能读写了。为了避免出现这种情况，就要注意长事务：

1. 事务不提交，就会一直占着 MDL 锁。在 mysql 的 `information_schema` 库的 `innodb_trx` 表中查找长事务，如果在做 DDL 变更的表有长事务在执行，要考虑先暂停 DDL，或者 kill 掉这个长事务

2. 如果要更新的表是热点表，kill 掉长事务后，马上又会有新的长事务。因此可以通过在 `alter table` 语句里面设定等待时间，如果在这个指定的等待时间里面能够拿到 MDL 写锁最好，拿不到也不要阻塞后面的业务语句，先放弃
```sql
-- mariadb
ALTER TABLE user NOWAIT add column source varchar(32);
ALTER TABLE user WAIT N add column source varchar(32);
```

mysql 5.6 支持 online ddl，减少了 ddl 操作持有 `MDL` 写锁的时间：
1. 获取 `MDL` 写锁
2. 降级成 `MDL` 读锁
3. 真正做 `DDL`
4. 升级成 `MDL` 写锁
5. 释放 `MDL` 锁


## 行锁
记录锁：针对单个行记录添加锁
间隙锁：锁住一个范围（索引之间的空隙），但不包括记录本身。采用间隙锁的方式可以防止幻读情况的产生
Next-Key 锁：锁住一个范围，同时锁定记录本身，相当于间隙锁 + 记录锁，可以解决幻读的问题

mysql 的行锁是在存储引擎中实现的，有的存储引擎支持行锁，有的不支持。在 Innodb 存储引擎中，行锁在需要的时候加上，但要等到事务结束时才释放。因此，如果事务中需要锁定多个行，要把可能造成锁冲突、最可能影响并发度的锁尽量往后放

使用行锁，可能会造成死锁发生。在出现死锁后，有以下两种策略：
1. 出现死锁状态时，直接进入等待，直到超时。这个超时时间可以通过参数 `innodb_lock_wait_timeout` 来设置
```sql
-- 默认值为 50s
show variables like "%innodb_lock_wait_timeout%";
```

2. 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 `innodb_deadlock_detect` 设置为 on，表示开启这个逻辑。
```sql
-- 默认值为 on
show variables like "%innodb_deadlock_detect%";
```
开启死锁检测虽然能够快速发现死锁并进行处理，但是会增加额外负担。当一个事务被锁的时候，就需要查看它所依赖的线程有没有被锁住，如果同时操作同一行的并发线程较多，那么死锁检测就要消耗大量 cpu 资源。这样看上去 cpu 利用率很高，但 tps 却不高

如果要解决这个问题，就要控制并发度，如果同一行记录只有少量的线程在更新，那么死锁检测的成本很低，就不会出现 cpu 利用率很高，但 tps 不高的情况了。而控制并发度，通常有以下方法：
1. 客户端做并发控制，减少客户端的并发线程。缺点是客户端比较多，即使每个客户端只有少量线程，汇总到数据库服务之后，峰值并发数仍会很高
2. 中间件，对于向同行的更新，在进入存储引擎之前进行排队
3. 考虑在业务上，将一行的逻辑改为多行来减少锁冲突


删除一个表里面的前 10000 行数据
方式一：单个语句执行占用时间长，锁的时间也比较长，而且大事务还会导致主从延迟
```sql
delete from T limit 10000;
```
方式二
```sql
-- 循环执行 20 次
delete from T limit 500;
```
方式三：产生锁冲突
```sql
-- 在 20 个连接中同时执行
delete from T limit 500;
```


## 间隙锁
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```
在可重复读隔离级别下，普通的查询是快照读，无法看到别的事务插入的数据。在当前读情况下会出现幻读，此时就需要用到间隙锁（可以看出，间隙锁在可重复读隔离级别下才有效）。幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，间隙锁 (Gap Lock) 的作用就是锁住两个值之间的空隙。例如我们执行以下语句：
```sql
select * from t where d=5 for update;
```
除了给数据库中已存在的 6 个记录加上行锁外，还同时加上了 7 个间隙锁，这样确保了无法插入新的记录。值得注意的是，间隙锁之间是不会发生冲突的，跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作。例如：
sessionA
```sql
begin;
-- 加间隙锁 (5, 10)
select * from t where c = 7 lock in share mode;
```
sessionB
```sql
begin;
-- 加间隙锁 (5, 10)
select * from t where c = 7 for update;
```
sessionA 与 sessionB 都是加的间隙锁，不会发生冲突，只有在往间隙中插入记录时才发生冲突

间隙锁与行锁合称 `next-key lock`，每个 `next-key lock` 都是前开后闭区间
```sql
-- 将整个表锁起来，形成了 7 个 next-key lock
-- (-∞,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20, 25]、(25, +supremum]
select * from t for update;
```

间隙锁的引入，加大了锁的范围，容易造成死锁，影响并发度
t1:
sessionA
```sql
begin;
select * from t where id = 9 for update;
```
t2:
sessionB
```sql
begin;
select * from t where id = 9 for update;
```
t3:
sessionB
```sql
-- blocked
insert into t values(9, 9, 9);
```
t4:
sessionA
```sql
-- innodb 检测到死锁，报错返回
insert into t values(9, 9, 9);
```

间隙锁是在可重复读隔离级别下才会生效的，如果把隔离级别设置为读提交，就没有间隙锁了。不过，为了解决可能出现的数据和日志不一致问题，需要把 binlog_format 设置为 row
```sql
show variables like "%binlog_format%";
```

### 等值查询间隙锁
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```
t1:
sessionA
```sql
begin;
update t set d = d + 1 where id = 7;
```
t2:
sessionB
```sql
-- blocked
insert into t values(8, 8, 8);
```
t3:
sessionC
```sql
-- ok
update set d = d + 1 where id = 10;
```
可重复读隔离级别下，加锁单位是 `next-key lock`，sessionA 的加锁范围是 (5, 10]。由于这是一个索引上面的等值查询，向右遍历时且最后一个值不满足等值条件的时候，`next-key lock` 退化为间隙锁，即最终加锁范围是 (5, 10)

### 非唯一索引等值锁
t1:
sessionA
```sql
begin;
select id from t where c = 5 lock in share mode;
```
t2:
sessionB
```sql
-- ok
update set d = d + 1 where id = 5;
```
t3:
sessionC
```sql
-- blocked
insert into t values(7, 7, 7);
```
可重复读隔离级别下，加锁单位是 `next-key lock`，sessionA 的加锁范围是 (0, 5]。由于 c 是普通索引，仅访问 `c = 5` 这一条记录是不能马上停下来的，需要继续向右遍历，查到 `c = 10` 为止，查询过程中访问到的都需要加锁，即给 (5, 10] 加锁。同时，由于这是一个索引上面的等值查询，向右遍历时且最后一个值不满足等值条件的时候，`next-key lock` 退化为间隙锁，加锁范围变为 (5, 10)

查询使用覆盖索引，并不需要访问主键索引，因此主键索引上没有加任何锁，这样 sessionB 才能执行成功，sessionC 会被间隙锁锁住

`lock in share mode` 只锁覆盖索引，如果换成 `for update` 就表示要更新数据，系统也会给主键索引上满足条件的行加上行锁

### 主键索引范围锁
t1:
sessionA
```sql
begin;
select * from t where id >= 10 and id < 11 for update;
```
t2:
sessionB
```sql
-- ok
insert into t values(8, 8, 8);
-- blocked
insert into t values(13, 13, 13);
```
t3:
sessionC
```sql
-- blocked
update t set d = d + 1 where id = 15;
```
sessionA 执行时，找到第一个 `id = 10` 的行，加锁 (5, 10]。索引上的等值查询，给唯一索引加锁的时候，`next-key lock` 退化为行锁，只对 `id = 10` 加锁。范围查找就往后继续找，找到 `id = 15` 这一行停下来，因此还需要在 (10, 15] 上面加锁

### 非唯一索引范围锁
t1:
sessionA
```sql
begin;
select * from t where c >= 10 and c < 11 for update;
```
t2:
sessionB
```sql
-- blocked
insert into t values(8, 8, 8);
```
t3:
sessionC
```sql
-- blocked
update t set d = d + 1 where c = 15;
```
sessionA 执行时，第一次用 `c = 10` 定位记录的时候，在索引 c 上加了 (5, 10] 的 `next-key lock`，由于索引 c 是非唯一索引，不会退化为行锁，最终在索引 c 上面加上 (5, 10] 和 (10, 15] 两个 `next-key lock`

### 唯一索引范围锁 bug
t1:
sessionA
```sql
begin;
select * from t where id > 10 and i <= 15 for update;
```
t2:
sessionB
```sql
-- blocked
update t set d = d + 1 where id = 20;
```
t3:
sessionC
```sql
-- blocked
insert into t values(16, 16, 16);
```
理论上来说，sessionA 应该是索引 id 上只加 (10,15] 的 `next-key lock`，并且因为 id 是唯一键，所以循环判断到 `id = 15` 这一行。但实际上，innodb 会继续向前扫描到第一个不满足条件的行为止，所以索引 id 还会加上 (15, 20] 的 `next-key lock`

### 非唯一索引上存在等值
```sql
insert into t values(30,10,30);
```
表里有两个 `c = 10` 的行，分别为 (c = 10, id = 10) 和 (c = 10, id = 30)

t1:
sessionA
```sql
begin;
delete from t where c = 10;
```
t2:
sessionB
```sql
-- blocked
insert into t values(12, 12, 12);
```
t3:
sessionC
```sql
-- ok
update t set d = d + 1 where c = 15;
```
sessionA 在遍历时，先访问 `c = 10` 的记录，加上 (c = 5, id = 5) 到 (c = 10, id = 10) 的 `next-key lock`，然后一直向后遍历，直到遇上 (c = 15, id = 15) 为止，加上 (c = 10, id = 10) 到 (c = 15, id = 15) 的间隙锁

### limit 语句加锁
t1:
sessionA
```sql
begin;
delete from t where c = 10 limit 2;
```
t2:
sessionB
```sql
-- ok
insert into t values(12, 12, 12);
```
delete 语句明确加了 `limit 2` 的限制，因此在遍历到 (c = 10, id = 30) 这一行之后，满足条件的语句已经有两条，循环结束。因此，索引 c 上的加锁范围就变成了从（c = 5, id = 5) 到 (c = 10, id = 30) 这个前开后闭区间。因此，在删除数据的时候尽量加 `limit`，这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁范围

## 死锁
t1:
sesssionA
```sql
begin;
select id from t where c = 10 lock in share mode;
```
t2:
sessionB
```sql
-- blocked
update t set d = d + 1 where c = 10;
```
t3:
sessionA
```sql
insert into t values(8, 8, 8);
```
t4:
sessionB
Deadlock found!

sessionA 启动事务，在索引 c 上加了 (5, 10] 的 `next-key lock` 和 (10, 15) 的间隙锁；sessionB 的 update 语句也要在索引 c 上面加上 (5, 10] 的 `next-key lock`。之后 sessionA 插入 (8, 8, 8)，结果被 sessionB 的间隙锁锁住，由于出现了死锁，InnoDB 让 session B 回滚

根据上面的分析得知，`next-key lock` 操作分为两步，先加 (5, 10) 的间隙锁，加锁成功；之后加 c = 10 的行锁，加锁失败

以上分析都是在可重复读的前提下，如果是在读提交隔离级别下，语句执行过程中加上的行锁，在语句执行完成后，就要把不满足条件的行上的行锁直接释放了，不需要等到事务提交。也即是说，读提交隔离级别下，锁的范围更小，锁的时间更短

我们可以通过以下命令查看，有一节 LATESTDETECTED DEADLOCK，就是记录的最后一次死锁信息
```sql
show engine innodb status
```

### 共享锁
sessionA
```sql
select * from product_comment where user_id = 912178 lock in share mode;
```
sessionB
```sql
UPDATE product_comment SET product_id = 10002 WHERE user_id = 912178;
```
