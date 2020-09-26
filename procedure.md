```sql
DELIMITER //
CREATE PROCEDURE `add_num`(IN n INT)
BEGIN
       DECLARE i INT;
       DECLARE sum INT;
       
       SET i = 1;
       SET sum = 0;
       WHILE i <= n DO
              SET sum = sum + i;
              SET i = i + 1;
       END WHILE;
       SELECT sum;
END //
DELIMITER ;
```
```sql
CALL add_num(50);
```


## 参数类型
### IN
不用返回。向存储过程传入参数，存储过程中修改改参数的值，不能被返回

### OUT
需要返回。把存储过程计算的结果放到改参数中，调用者可以得到返回值

### INOUT
需要返回。IN 和 OUT 的结合，既用于存储过程的传入参数，同时又可以把计算结果放到参数中，调用者可以得到返回值

```sql
CREATE PROCEDURE `get_hero_scores`(
       OUT max_max_hp FLOAT,
       OUT min_max_mp FLOAT,
       OUT avg_max_attack FLOAT,  
       s VARCHAR(255)
       )
BEGIN
       SELECT MAX(hp_max), MIN(mp_max), AVG(attack_max) FROM heros WHERE role_main = s INTO max_max_hp, min_max_mp, avg_max_attack;
END
```
```sql
CALL get_hero_scores(@max_max_hp, @min_max_mp, @avg_max_attack, '战士');
SELECT @max_max_hp, @min_max_mp, @avg_max_attack;
```


## 流控制语句
### BEGIN END
BEGIN END 中间包含了多个语句，每个语句都以 ; 号为结束符

### DECLARE
声明变量

### SET
赋值语句

### SELECT INTO
把从数据表中查询的结果存放到变量中

### IF THEN ENDIF
条件判断语句

### CASE
用于多条件的分支判断

### LOOP LEAVE ITERATE
LOOP 是循环语句，使用 LEAVE 可以跳出循环，使用 ITERATE 则可以进入下一次循环

### REPEAT-UNTIL-END REPEAT
首先会执行一次循环，然后在 UNTIL 中进行表达式的判断，如果满足条件就退出，即 END REPEAT；如果条件不满足，则会就继续执行循环，直到满足退出条件为止

### WHILE-DO-END WHILE
这个语句需要先进行条件判断，如果满足条件就进行循环，如果不满足条件就退出循环
