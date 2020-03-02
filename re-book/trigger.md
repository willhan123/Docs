# 触发器
触发器是与表有关的数据库对象，在满足定义条件时触发，并执行触发器中定义的语句集合。TenDB Cluster兼容MySQL触发器语法。



<font color="#dd0000">注意：</font> 
```
不建议在TenDB Cluster中使用复杂的存储过程，触发器，函数。   
在对数据一致性高要求的场景中使用存储过程或者触发器，建议配置分布式事务；否则TenDB Cluster只能支持单机事务。
```


示例：

```
CREATE TABLE account (acct_num INT, amount DECIMAL(10,2));
INSERT INTO account VALUES(137,14.98),(141,1937.50),(97,-100.00);
```

创建触发器
```
delimiter $$
    CREATE TRIGGER upd_check BEFORE UPDATE ON account
    FOR EACH ROW
    BEGIN
    IF NEW.amount < 0 THEN
    SET NEW.amount = 0;
    ELSEIF NEW.amount > 100 THEN
    SET NEW.amount = 100;
    END IF;
    END$$
delimiter ;
```

触发
```
update account set amount=-10 where acct_num=137;

select * from account;

+----------+---------+
| acct_num | amount  |
+----------+---------+
|      137 |    0.00 |
|      141 | 1937.50 |
|       97 | -100.00 |
+----------+---------+
```

删除触发器
```
DROP TRIGGER upd_check;
```
