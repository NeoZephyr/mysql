MyCAT
逻辑库
对应后端多个物理数据库
不保存数据

配置
```
conf/wrapper.conf
```
修改内存大小

环境变量配置
```
export MyCAT_HOME=/usr/local/mycat
```

启动 mycat
```
mycat start
```

server.xml
配置系统相关参数
配置用户访问权限
配置 SQL 防火墙及 SQL 拦截功能
```
<mycat:server xmlns:mycat="http://io.mycat/">
  <system>
    <property name="serverPort">3306</property>
  </system>

  <user name="test">
    <property name="password">123456</property>
    <property name="schemas">machine,access</property>
    <property name="readOnly">false</property>
  </user>
</mycat>
```

配置 MyCAT 访问用户及权限
dml 代码 insert,update,select,delete
```
<user name="test">
  <property name="password">123456</property>
  <property name="schemas">machine,access</property>
  <privileges check="true">
    <schema name="machine" dml="0110">
      <table name="machine0" dml="0000"></table>
      <table name="machine1" dml="1111"></table>
    </schema>
  </privileges>
</user>
```

使用加密密码
```
java -cp Mycat-server-1.6.5-release.jar io.mycat.util.DecryptUtil 0:root:123456
```
```
<user name="test" defaultAccount="true">
  <property name="usingDecrypt">1</property>
  <property name="password">123456</property>
  <property name="schemas">machine,access</property>
</user>
```

log4j2.xml 配置
日期：%d{yyyy-MM-dd HH:mm:ss.SSS}
日志级别：%5p
线程号：[%t]

rule.xml 配置
配置水平分片的分片规则
配置分片规则所对应的分片函数

```
<tableRule name="hash-mod-4_id">
  <rule>
    <columns>id</columns>
    <algorithm>hash-mod-4</algorithm>
  </rule>
</tableRule>
```
分片算法
```
<function name="hash-mod-4" class="io.mycat.route.function.PartitionByHashMod">
  <property name="count">4</property>
</function>
```

常用分片算法
1. 简单取模：PartitionByMod
```
<tableRule name="customer">
  <rule>
    <columns>customer_id</columns>
    <algorithm>mod-long</algorithm>
  </rule>
</tableRule>
<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
  <property name="count">2</property>
</function>
```

2. 哈希取模：PartitionByHashMod
```
<tableRule name="customer">
  <rule>
    <columns>customer_name</columns>
    <algorithm>mod-long</algorithm>
  </rule>
</tableRule>
<function name="mod-long" class="io.mycat.route.function.PartitionByHashMod">
  <property name="count">2</property>
</function>
```

3. 分片枚举：PartitionByFileMap
```
<function name="hash-int" class="io.mycat.route.function.PartitionByFileMap">
  <property name="mapFile">partition-hash-int.txt</property>
  <property name="type">0</property>
  <property name="defaultNode">0</property>
</function>
```
type 为 0 表示枚举值为整数类型
type 为非 0 表示枚举值为字符串类型

4. 字符串范围取模分片
```
<function name="sharding-by-prefix-pattern" class="io.mycat.route.function.PartitionByPrefixPattern">
  <property name="patternValue">128</property>
  <property name="prefixLength">2</property>
  <property name="mapFile">prefix-partition-pattern.txt</property>
</function>
```

$MYCAT/conf 增加 MapFile 配置取模范围同节点的对应关系

schema.xml 配置
配置逻辑库与逻辑表
配置逻辑表所存储的数据节点
配置数据节点所对应的物理数据库服务器信息

定义逻辑库
```
<schema name="machine" checkSQLschema="false" sqlMaxLimit="1000">
</schema>
```
定义逻辑表
```
<table name="customer" primaryKey="customer_id" dataNode="customer01,customer02" rule="customer" />
```
dataNode 定义表数据所存储的数据节点
rule 定义逻辑表分片规则（rule.xml）

定义逻辑表所存储的物理数据库
```
<dataNode name="customer01" dataHost="mysql001" database="customer" />
```

定义后端数据库主机信息
```
<dataHost name="mysql001" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="1">
  <heartbeat>select user()</heartbeat>
  <writeHost host="192.168.1.3" url="192.168.1.3306" user="im_mycat" password="123456">
    <readHost host="192.168.1.4" url="192.168.1.4:3306" user="im_mycat" password="123456">
  </writeHost>
</dataHost>
```
balance
0 表示不开启读写分离机制
1 表示全部的 readHost 与 stand by writeHost 参与 select 语句的负载均衡
2 表示所有的 readHost 与 writeHost 参与 select 语句的负载均衡
3 表示所有的 readHost 参与 select 语句的负载均衡

备份
```
mysqldump --master-data=2 --single-transaction -uroot -p machine > machine_bak.sql
```

```
change master to master_host='192.168.1.2', master_user='im_impl',master_password='123456',MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=4532519;

change replication filter replicate_rewrite_db=((imooc_db,product_db));

start slave;

show slave status;
```

垂直切分
schema.xml
```
<mycat:schema xmlns:mycat="http://io.mycat/">

  <schema name="pain_db" checkSQLschema="false" sqlMaxLimit="100">
    <table name="order" primarykey="order_id" dataNode="orderDB" />
    <table name="order_detail" primarykey="order_detail_id" dataNode="orderDB" />

    <table name="product" primarykey="product_id" dataNode="productDB" />
    <table name="product_category" primarykey="product_category_id" dataNode="productDB" />

    全局表
    <table name="region_info" primarykey="region_id" dataNode="orderDB,productDB" type="global" />
  </schema>

  <dataNode name="orderDB" dataHost="mysql001" database="order_db" />
  <dataNode name="productDB" dataHost="mysql002" database="product_db" />

  <dataHost name="mysql001" maxCon="1000" minCon="10" balance="3" writeType="0" dbType="mysql" dbDriver="native" switchType="1">
    <heartbeat>select user()</heartbeat>
    <writeHost host="192.168.1.1" url="192.168.1.1:3306" user="test" password="123456" />
  </dataHost>

  <dataHost name="mysql002" maxCon="1000" minCon="10" balance="3" writeType="0" dbType="mysql" dbDriver="native" switchType="1">
    <heartbeat>select user()</heartbeat>
    <writeHost host="192.168.1.2" url="192.168.1.2:3306" user="test" password="123456" />
  </dataHost>
</mycat:schema>
```
server.xml
```
<mycat:server xmlns:mycat="http://io.mycat/">
  <system>
    <property name="serverPort">8066</property>
  </system>

  <user name="app_mall" defaultAccount="true">
    <property name="password">123456</property>
    <property name="schemas">pain_db</property>
  </user>
</mycat:server>
```

停止主从同步
```
stop slave;
reset slave all;
```

全局表实现跨分片查询

水平分片
选择分片键
1. 尽可能的比较均匀分布数据到各个节点上
2. 该业务字段是最频繁的或者最重要的查询条件

```
<dataNode name="orderDB01" dataHost="mysql001" database="orderDB01" />
<dataNode name="orderDB02" dataHost="mysql001" database="orderDB02" />
<dataNode name="orderDB03" dataHost="mysql002" database="orderDB03" />
<dataNode name="orderDB04" dataHost="mysql002" database="orderDB04" />
```
```
<schema name="pain_db" checkSQLschema="false" sqlMaxLimit="100">
  <table name="order" primarykey="order_id" dataNode="orderDB01,orderDB02,orderDB03,orderDB04" rule="order_rule" />
  <table name="order_detail" primarykey="order_detail_id" dataNode="orderDB" />

  <table name="product" primarykey="product_id" dataNode="productDB" />
  <table name="product_category" primarykey="product_category_id" dataNode="productDB" />

  全局表
  <table name="region_info" primarykey="region_id" dataNode="orderDB,productDB" type="global" />
</schema>
```
rule.xml
```
<mycat:rule xmlns:mycat="http://io.mycat/">
  <tableRule name="order_rule">
    <rule>
      <columns>customer_id</columns>
      <algorithm>mod-long</algorithm>
    </rule>
  </tableRule>

  <function name="mod-long" class="io.mycat.route.function.PartitionByMod">
    <property name="count">4</property>
  </function>
</mycat:rule>
```