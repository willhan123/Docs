# TenDB Cluster 权限管理

应用层的权限管理将在TSpider中完成，权限语句和管理模式跟单一mysql完全一致。


## FLUSH PRIVILEGES

FLUSH PRIVILEGES 语句可触发 Tspider 从权限表中重新加载权限的内存副本。在对如 mysql.user 一类的表进行手动编辑后，应当执行 FLUSH PRIVILEGES。使用如 GRANT 或 REVOKE 一类的权限语句后，不需要执行 FLUSH PRIVILEGES 语句。



## GRANT

GRANT <privileges> to user@'ip' identified by password  
用于为 Tspider 创建用户，并分配权限。

```
MariaDB [tendb_test]> GRANT SELECT, INSERT, UPDATE, DELETE, DROP, EXECUTE on *.* to spider@'***' identified by '***';
Query OK, 0 rows affected (0.00 sec)
```


## SHOW GRANTS 

用于显示与用户关联的权限列表。

```
MariaDB [tendb_test]> show grants;

 GRANT SELECT, INSERT, UPDATE, DELETE, DROP, EXECUTE ON *.* TO 'spider'@'***' IDENTIFIED BY PASSWORD '*6CFA6C059D69E70E1E4A37045E474F9D761BB5EB' 

1 row in set (0.00 sec)
```



## SHOW PRIVILEGES 
用于显示 Tspider 中可分配权限的列表。
