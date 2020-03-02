应用层的权限管理将在TSpider中完成，权限语句和管理模式跟单一mysql完全一致。需要注意的是mysql的授权语句不会分发给后端tendb.


FLUSH PRIVILEGES
```
FLUSH PRIVILEGES 语句可触发 Tspider 从权限表中重新加载权限的内存副本。在对如 mysql.user 一类的表进行手动编辑后，应当执行 FLUSH PRIVILEGES。使用如 GRANT 或 REVOKE 一类的权限语句后，不需要执行 FLUSH PRIVILEGES 语句。
```


GRANT

```
GRANT <privileges> 语句用于为 Tspider 中已存在的用户分配权限。
```



SHOW GRANTS 
```
用于显示与用户关联的权限列表。与在 MySQL 中一样，USAGE 权限表示登录 Tspider 的能力。
```


SHOW PRIVILEGES 
```
用于显示 Tspider 中可分配权限的列表。
```