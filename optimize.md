## 分页
```sql
select * from `order` order by order_no limit 10000, 20;
```

```sql
select * from `order` where id > (
    select id from `order` order by order_no limit 10000, 1
) limit 20;
```

## innodb
innodb_buffer_pool_size
默认的内存大小是 128M，推荐配置为服务器物理内存的 80%

innodb_buffer_pool_instances
