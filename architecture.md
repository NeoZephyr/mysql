## 基础架构
### `Server` 层
涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

#### 连接器
管理连接，权限验证

#### 查询缓存
缓存命中则直接返回结果

#### 分析器
词法分析，语法分析

#### 优化器
执行计划生成，索引选择

#### 执行器
操作引擎，返回结果

### 存储引擎层
负责数据的存储和提取，支持 InnoDB、MyISAM、Memory 等多个存储引擎。存储引擎负责存储数据，提供读写接口

#### `MyISAM`
1. 不支持事务
2. 支持表级锁
3. 存储表的总行数
4. 一个 MyISAM 表有三个文件：索引文件、表结构文件、数据文件
5. 采用非聚集索引：B+ 树的数据结构中存储的内容实际上是实际数据的地址值，也就是说它的索引和实际数据是分开的

#### `InnoDb`
1. 支持事务
2. 支持行级锁
3. 不存储总行数
4. 主键索引采用聚集索引（索引的数据域存储数据文件本身），辅索引的数据域存储主键的值；因此从辅索引查找数据，需要先通过辅索引找到主键值，再访问辅索引；最好使用自增主键，防止插入数据时，为维持 B+ 树结构，文件的大调整

## 查询语句执行流程
```sql
select * from user where id = 1;
```

### 连接器

#### 建立连接
```sh
mysql -h <ip> -P <port> -u <user> -p <password>
```

#### 身份认证

#### 获取权限

#### 管理连接
连接完成之后，如果没有后续动作，这个连接就处于空闲状态。如果连接长时间没有操作，就会被断开，这个时间由参数 `wait_timeout` 控制，默认为 8 小时。
```sql
show processlist;
```

1. 短连接：每次执行完几次查询就断开连接，下次查询重新建立连接
2. 长连接：如果客户端持续有请求，则一直使用同一个连接。由于 mysql 执行过程中临时使用的内存是管理在连接对象里面的，这些资源在连接断开的时候才释放。如果长期积累下来，可能导致内存占用太大（OOM）。可以通过执行 `mysql_reset_connection` 来重新初始化连接资源（不需要重连和重做权限验证），将连接恢复到刚刚创建完时的状态；也可以通过定期断开长连接来解决该问题。

### 查询缓存（deprecated in 8.0）
mysql 接收到查询请求后，会先查询缓存。之前执行过的语句及其结果可能会以 `key-value` 的形式缓存在内存中，其中 `key` 是查询语句，`value` 是查询结果。实际情况下，查询缓存失效非常频繁，只要对一个表做更新操作，该表上面的查询缓存都会被清空。因此除非是类似于系统配置类的静态表，一般不使用查询缓存。将参数 `query_cache_type` 设置为 `DEMAND`，这样对默认的 SQL 语句都不使用查询缓存。如果要使用查询缓存，可以在语句中显示指定：
```sql
select SQL_CACHE * from sys_settings;
```

### 分析器
对 SQL 语句进行词法分析与语法分析

### 优化器
优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联的时候，决定各个表的连接顺序。
```sql
# 可以先从表 t1 中取出 c=10 的记录的 id 值，再根据 id 值关联到表 t2，再判断 t2 里面 d 的值是否等于 20
# 也可以先从表 t2 中取出 d=20 的记录的 id 值，再根据 id 值关联到表 t1，再判断 t1 里面 c 的值是否等于 10
select * from t1 join t2 using(id) where t1.c=10 and t2.d=20;
```

### 执行器
先判断是否具有表 `user` 的执行权限，如果有权限才能继续执行。如果 `id` 字段没有索引，执行流程如下：
1. 调用 innodb 引擎接口读取表的第一行，并判断 id 是否为 1，若不是则跳过，若是则将这行存在结果集中
2. 调用引擎接口获取下一行，重复相同逻辑直到最后一行
3. 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端

如果 `id` 字段有索引，执行流程类似，第一次调用的是“取满足条件的第一行”这个接口，之后循环取“满足条件的下一行”这个接口，直到查询找出所有满足条件的记录。

## 更新语句执行流程
更新语句执行流程与查询语句执行流程大致类似，但有两点注意。一是更新语句会将对应表上面的缓存结果都清空；二是更新过程涉及到两个重要的日志模块：`redo log` 与 `binlog`