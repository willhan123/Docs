# TenDBCluster授权
应用在请求TenDBCluster时, 需要在TSpider节点通过grant语句申请相应的权限，TSpider将grant语句发到Tdbctl节点，Tdbctl转发到其他TSpider节点执行，完成整个集群的授权操作  
权限的语法与普通MySQL相同，在使用上没有太大的区别

## 授权示例
连接任意TSpider节点
>mysql -umysql -pmysql -h127.0.0.1 -P25000
- 授权
```sql
create user abc@'127.0.0.1' identified by 'abc' ;
grant select,insert,alter on test1.* to abc@'127.0.0.1';
```
- 验证
> mysql -uabc -pabc -h127.0.0.1 -P25000
```
#查看权限
show grants;
+------------------------------------------------------------------------------------------------------------+
| Grants for abc@127.0.0.1                                                                                   |
+------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'abc'@'127.0.0.1' IDENTIFIED BY PASSWORD '*0D3CED9BEC10A777AEC23CCC353A8C08A633045E' |
| GRANT SELECT, INSERT, ALTER ON `test1`.* TO 'abc'@'127.0.0.1'                                              |
+------------------------------------------------------------------------------------------------------------+
use test1;
insert into t1 values(9,10,11);
```
