# 函数
函数是存在数据库中的一段sql集合，调用函数可以减少很多工作量。TenDB Cluster兼容MySQL函数语法。



<font color="#dd0000">注意：</font> 
```
不建议在TenDB Cluster中使用复杂的存储过程，触发器，函数。   
在对数据一致性高要求的场景中使用存储过程或者触发器，建议配置分布式事务；否则TenDB Cluster只能支持单机事务。 

```



示例：

创建函数
```
CREATE FUNCTION f2(num1 SMALLINT UNSIGNED, num2 SMALLINT UNSIGNED)
RETURNS FLOAT(10,2) UNSIGNED
RETURN (num1+num2)/2;
```
调用函数
```
SELECT f2(1,2);
+---------+
| f2(1,2) |
+---------+
|    1.50 |
+---------+
```

删除函数
```
DROP FUNCTION f2;
```
