## count
不同的存储引擎，count 有不同的实现方式：
1. MyISAM 中把一个表的总行数存储到磁盘上，因此 count 直接返回，效率很高（没有 where 过滤条件的情况下）
2. Innodb 在执行的时候，需要把数据一行一行地从存储引擎中读出来，然后累积计数

Innodb 没有采取与 MyISAM 一样的方式将计数存起来，这跟是由于多版本并发控制的原因，要返回的数据行数在同一时刻多个查询时是不同的。Innodb 默认是可重复读隔离级别，在代码上就是通过多版本并发控制，每一行记录都要判断自己是否对这个会话可见，因此对于 count 操作来说，只能把数据一行一行地读出来依次判断，可见的行才能够加到对应查询的总行数中

在 Innodb 中，主键索引的叶子节点是数据，而普通索引的叶子节点是主键值。所以，普通索引比主键索引树小很多，而 count 这样的操作，无论遍历哪个索引树结果逻辑都是一样的，优化器就会选择最小的那棵树进行遍历

count 查询有以下多种形式：
1. count(主键 id)：遍历整张表，取出每一行的 id 返回到 server 层，server 层判断不为空之后累加
2. count(1)：遍历整张表，但不取值，server 层按行累加
3. count(*)：不取值，server 层按行累加，推荐使用

## sort
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;
```
```sql
select city, name, age from t where city='杭州' order by name limit 1000;
```
执行以上语句，流程如下：
1. 初始化 sort_buffer，放入 `city`, `name`, `age` 字段
2. 从 `city` 索引树中找到第一个满足 `city='杭州'` 条件的主键 id
3. 根据主键读取记录，并取出 `city`, `name`, `age` 字段值放入 sort_buffer 中
4. 继续从 `city` 索引树中找到下一个满足 `city='杭州'` 条件的主键 id
5. 重复之前的操作，直到出现不满足 `city='杭州'` 条件的记录为止
6. 对 `sort_buffer` 中的数据按照字段 `name` 进行快速排序
7. 取排序结果的前 1000 行返回给客户端

可以看出，`sort_buffer` 是 mysql 为了排序开辟的内存空间。其大小取决于参数 `sort_buffer_size`
```sql
show variables like "%sort_buffer_size%";
```
如果要排序的数据量小于 `sort_buffer_size` 的大小，就可以在内存中排序；否则，就需要利用磁盘临时文件辅助排序。我们可以通过以下方法，查看是否使用了临时文件
```sql
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a 保存 Innodb_rows_read 的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
/* 从 number_of_tmp_files 中查看是否使用了临时文件 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G;

/* @b 保存 Innodb_rows_read 的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算 Innodb_rows_read 差值 */
/* 表示扫描行数 */
select @b-@a;
```
通过查看 OPTIMIZER_TRACE 的结果中的 number_of_tmp_files 来确认是否使用了临时文件，number_of_tmp_files 表示排序过程中使用的临时文件数。如果需要排序的数据量小于 `sort_buffer_size`，就不需要临时文件，可以看到 number_of_tmp_files 的值为 0

需要注意的是，如果要查询的字段较多，那么 `sort_buffer` 中要存储的字段也会增多，这样数量很大的情况下，由于内存中能存下的数据有限，就要分成很多临时文件，性能会很差。因此，在单行数据很大的情况下，mysql 会采取其它的算法
```sql
show variables like "%max_length_for_sort_data%";
```
模拟单行数据很大的情况，将 `max_length_for_sort_data` 设置较小
```sql
SET max_length_for_sort_data = 16;
```
新的算法只会将需要排序的列和主键放到 `sort_buffer` 中，执行流程是这样的：
1. 初始化 sort_buffer，放入 `name`, `age` 字段
2. 从 `city` 索引树中找到第一个满足 `city='杭州'` 条件的主键 id
3. 根据主键读取记录，并取出 `name` 字段值放入 sort_buffer 中
4. 继续从 `city` 索引树中找到下一个满足 `city='杭州'` 条件的主键 id
5. 重复之前的操作，直到出现不满足 `city='杭州'` 条件的记录为止
6. 对 `sort_buffer` 中的数据按照字段 `name` 进行快速排序
7. 取排序结果的前 1000 行，并按照 id 查询表取出 `city`，`name`，`age` 字段返回客户端

可以看出，该算法多访问了一次主键索引树，需要尽量避免

其实，并不是所有的 `order by` 语句都是需要排序的，比如我进行如下操作：
```sql
alter table t add index city_name(city, name);
```
我们可以发现，在 (city, name) 索引树中，只要按照 `city='杭州'` 条件依次取出记录，那么就能保证 `name` 值是有序的。此时的查询流程简化成这样：
1. 从 `(city, name)` 索引树中找到第一个满足 `city='杭州'` 条件的主键 id
2. 根据主键读取记录，并取出 `name`, `city`, `age` 字段值作为结果集的一部分直接返回
3. 从 `(city, name)` 索引树中找到下一个满足 `city='杭州'` 条件的主键 id
4. 重复以上步骤，直到查到第 1000 条记录，或者不满足 `city='杭州'` 条件为止

如果，我们此时使用覆盖索引，查询流程会变得更加简单（不需要回表）
```sql
alter table t add index city_user_age(city, name, age);
```

```sql
select * from customer where city in ('杭州'," 苏州 ") order by name limit 100;
select * from customer where city in ('杭州'," 苏州 ") order by name limit 10000, 100;
```

## rand
```sql
CREATE TABLE `words` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `word` varchar(64) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```
```sql
delimiter ;;
create procedure words_idata()
begin
  declare i int;
  set i=0;
  while i<10000 do
    insert into words(word) values(concat(char(97+(i div 1000)), char(97+(i % 1000 div 100)), char(97+(i % 100 div 10)), char(97+(i % 10))));
    set i=i+1;
  end while;
end;;
delimiter ;

call words_idata();
```
随机排序取前 3 个
```sql
select word from words order by rand() limit 3;
```
我们使用 `explain` 分析发现，这个查询语句需要使用临时表，并在临时表上面排序。对于 innodb 表来说，执行全字段排序会减少磁盘访问，会优先选择；而对于内存表，回表过程只是简单地根据数据行的位置，直接访问内存得到数据，不会导致多访问磁盘，那么用于排序的行尽量小，此时就选择 rowid 排序。语句执行流程是这样的：
1. 创建临时表，使用 `memory` 引擎，有两个字段，第一个为 `double` 类型，第二个为 `varchar` 类型。该表没有建立索引
2. 从 words 表中按主键顺序取出所有 word 值，对每一个 word 值调用 rand 函数生产一个大于 0 小于 1 的随机小数，并将 word 与随机小数存入到临时表中
3. 初始化 `sort_buffer`，`sort_buffer` 中包含两个字段：double 类型与整型
4. 从内存临时表中取出随机小数值与位置信息（rowid）存入 `sort_buffer`
5. 在 `sort_buffer` 中根据随机小数的值进行排序
6. 排序完成，取出前三个结果的位置信息，依次到内存临时表中取出 word 值，返回给客户端

上面的临时表是内存表，这是因为临时表的大小不大。如果临时表的大小超过了 `tmp_table_size`，那么就会使用磁盘临时表
```sql
show variables like "%tmp_table_size%";
```
磁盘临时表使用的引擎默认是 InnoDB，是由参数 internal_tmp_disk_storage_engine 控制
```sql
show variables like "%internal_tmp_disk_storage_engine%";
```

```sql
set tmp_table_size=1024;
set sort_buffer_size=32768;
set max_length_for_sort_data=16;

/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* 执行语句 */
select word from words order by rand() limit 3;

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
```
由于将 `max_length_for_sort_data` 设置为 16，小于 word 字段的长度，因此可以看到 sort_mode 中显示的是 rowid 排序，参与排序的是随机值字段和 rowid 字段组成的行

有一个需要注意的地方，随机值字段字节数是 8，rowid 字节数是 6，数据行的字节数乘以扫描行数的总字节超过了 `sort_buffer_size` 定义的字节大小，可是 `number_of_tmp_files` 的值却是 0，即没有使用临时文件，这说明排序过程没有使用归并算法，而是使用了堆排序的算法（filesort_priority_queue_optimization 这个部分的 chosen=true，就表示使用了优先队列排序）。这是因为我们只需要排序后的前 3 个值，如果使用归并算法的话，算法结束后所有数据都有序了。如果 `limit` 的数字比较大，当需要维护的堆大小超过 `sort_buffer_size` 时，就会转为归并排序

通过以上分析可以得知 `order by rand()` 的计算过程非常复杂，可以使用以下方法替代：
```sql
-- 由于 id 空洞的存在，可能导致概率分布不均匀
select max(id), min(id) into @M, @N from words;
set @X = floor((@M-@N+1)*rand() + @N);
select * from words where id >= @X limit 1;
```
```sql
-- 概率分布均匀
select count(*) into @C from words;
set @Y = floor(@C * rand());
set @sql = concat("select * from words limit ", @Y, ",1");
prepare stmt from @sql;
execute stmt;
DEALLOCATE prepare stmt;
```
```sql
select count(*) into @C from words;
set @Y1 = floor(@C * rand());
set @Y2 = floor(@C * rand());
set @Y3 = floor(@C * rand());
select * from words limit @Y1, 1;
select * from words limit @Y2, 1;
select * from words limit @Y3, 1;
```
```sql
-- id1
select * from words limit @Y1, 1;
-- id2
select * from words where id > id1 limit @Y2 - @Y1, 1;
select * from words where id > id2 limit @Y3 - @Y2, 1;
```

## 条件字段函数
```sql
CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
```sql
-- 对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能
-- 扫描 t_modified 整个索引的所有值
select count(*) from tradelog where month(t_modified)=7;
```
```sql
-- 改造成基于字段本身的范围查询
select count(*) from tradelog where
(t_modified >= '2016-7-1' and t_modified<'2016-8-1') or
(t_modified >= '2017-7-1' and t_modified<'2017-8-1') or 
(t_modified >= '2018-7-1' and t_modified<'2018-8-1');
```

## 隐式类型转换
```sql
-- 在 MySQL 中，字符串和数字做比较的话，是将字符串转换成数字
explain select * from tradelog where tradeid=110717;
explain select * from tradelog where tradeid="110717";
```

## 隐式字符编码转换
```sql
CREATE TABLE `trade_detail` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `trade_step` int(11) DEFAULT NULL, /* 操作步骤 */
  `step_info` varchar(32) DEFAULT NULL, /* 步骤信息 */
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
分析一下语句：优化器第一行表示先在交易记录表 tradelog 上查到 id=2 的行（使用主键索引），从 tradelog 表中取到 tradeid 字段，然后去 trade_detail 中查询匹配字段（没有使用索引）
```sql
explain select r.* from tradelog l, trade_detail r where r.tradeid=l.tradeid and l.id=2;
```
没有使用索引的原因是两个表的字符集不同，由于 字符集 utf8mb4 是 utf8 的超集，当两个类型的字符串进行比较时，总是先把 utf8 字符串转成 utf8mb4 字符集，然后进行比较。就好像执行下面这条语句一样：
```sql
-- 索引字段上面加上函数操作，导致全表扫描
select * from trade_detail where CONVERT(traideid USING utf8mb4)=$L2.tradeid.value;
```
如果要避免这种情况，有以下两种办法：
1. 修正字符集编码
```sql
alter table trade_detail modify tradeid varchar(32) CHARACTER SET utf8mb4 default null;
```
2. 修改 sql 语句
```sql
select r.* from tradelog l, trade_detail r where r.tradeid=CONVERT(l.tradeid USING utf8) and l.id=2;
```

## join
```sql
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100);
```

t1 是驱动表，t2 是被驱动表
```sql
select * from t1 straight_join t2 on (t1.a=t2.a);
```
执行流程是这样的：
1. 从表 t1 中读入一行数据 R
2. 从数据行 R 中，取出 a 字段到表 t2 里去查找
3. 取出表 t2 中满足条件的行，跟 R 组成一行，作为结果集的一部分
4. 重复执行步骤 1 到 3，直到表 t1 的末尾循环结束

在这个流程中，对驱动表进行全表扫描，这个过程需要扫描 100 行。对于每一行的 R 根据 a 字段去表 t2 查找，由于走的是树搜索过程，每次的搜索过程都只扫描一行，也是扫描 100 行。总共扫描 200 行

使用 join 时，尽量使用小表作为驱动表

```sql
select * from t1 straight_join t2 on (t1.a=t2.b);
```
如果被驱动表上没有可用的索引，算法的流程是这样的：
1. 把表 t1 的数据读入线程内存 join_buffer 中，由于我们这个语句中写的是 `select *`，因此是把整个表 t1 放入了内存
2. 扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回

这个过程对表 t1、t2 都进行了全表扫描，总的扫描行数是 1100，总共做了 100 * 1000 次判断

join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256k。如果驱动表 t1 特别大，就进行分段存放。执行流程变成这样：
1. 扫描表 t1，顺序读取数据行放入 join_buffer 中，直到 join_buffer 满了
2. 扫描表 t2，把 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回
3. 清空 join_buffer
4. 继续扫描表 t1，顺序读取后面的数据放入 join_buffer 中，继续执行第 2 步

可以通过调大 `join_buffer_size`，减少分段，这样就减少了对被驱动表的扫描次数

```sql
-- join_buffer 只需要放入 t1 100 行
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;

-- join_buffer 只需要放入 t2 的前 50 行，此时 t2 是小表
select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
```

```sql
-- 表 t1 和 t2 都是只有 100 行参加 join

-- 表 t1 只查字段 b，t1 作为驱动表只需要把字段 b 放到 join_buffer 中
-- 表 t2 要查询所有字段，t2 作为驱动表需要把字段 id, a, b 放到 join_buffer 中

select t1.b,t2.* from  t1  straight_join t2 on (t1.b=t2.b) where t2.id<=100;
select t1.b,t2.* from  t2  straight_join t1 on (t1.b=t2.b) where t2.id<=100;
```

在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表


