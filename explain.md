## `id`
`id` 相同，执行顺序由上而下；`id` 不同，值越大越先被执行

## `select_type`
1. `SIMPLE`：表示此查询不包含 `UNION` 查询或子查询
2. `PRIMARY`：表示此查询是最外层的查询
3. `SUBQUERY`：子查询中的第一个 `SELECT`
4. `UNION`：表示此查询是 `UNION` 的第二或随后的查询
5. `DERIVED`：衍生，表示导出表的 `SELECT`（`FROM` 子句的子查询）

## `key_len`
查询优化器使用了索引的字节数