# 存储过程
存储过程（Stored Procedure）是一种在数据库中存储复杂程序，以便外部程序调用的一种数据库对象。TenDB Cluster兼容MySQL存储过程语法。


<font color="#dd0000">注意：</font> 
```
不建议在TenDB Cluster中使用复杂的存储过程，触发器，函数。   
在对数据一致性高要求的场景中使用存储过程或者触发器，建议配置分布式事务；否则TenDB Cluster只能支持单机事务。 

```



示例：

创建存储过程

```
DELIMITER // 
CREATE PROCEDURE GreetWorld( ) SELECT CONCAT(@greeting,' World');
END 
// 
DELIMITER ;
```

调用存储过程
```
SET @greeting='Hello';
CALL GreetWorld( );

+----------------------------+
| CONCAT(@greeting,' World') |
+----------------------------+
| Hello World                |
+----------------------------+
```

删除存储过程
```
drop PROCEDURE GreetWorld;
```