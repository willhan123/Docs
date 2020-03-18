# Tdbctl控制层高可用
Tdbctl控制层的高可用相对比较简单，目前我们采用MySQL官方推荐的GROUP REPLICATION方案，3个节点，一主二从的方式  

## 故障发现
不同于TSpider，TenDB节点的外围探测的方式发现故障，Tdbctl的故障发现，更多的是依赖MGR方案内部的故障探测机制来决策故障。
具体决策机制实现细节可以参考MySQL GROUP REPLICATION相关文档

## 故障切换
在故障发生后，MGR会从复制组剔除故障节点，通过选举方式提升一个从节点为主节点来保证服务。
鉴于我们的mysql.servers表记录了所有MGR节点的路由信息，因此故障处理上，除了MGR本身提供的故障切换功能外，我们还需要从mysql.servers表中去掉故障节点  
```sql
delete from mysql.servers where Host='$fault_tdbctl_ip' and Port='$fault_tdbctl_port';
```

## 故障恢复
在故障节点恢复，或新的故障节点加入时(全量数据已经导入)，我们可以参考MGR的配置，将新的节点配置到MGR
- 修改my.cnf，启用MGR相关配置  
>参考集群搭建的[mgr配置](manual-install.md/#mgr-cnf)
- 配置复制权限
- 启用MGR
>```sql
START GROUP_REPLICATION;
```

在MGR同步正常后，连接tdbctl的主节点，修改mysql.servers表，将新节点信息插入

## 其他说明
我们以26000实例故障为例，演示说明tdbctl的故障切换和故障恢复
### 故障切换
关闭26000实例

- 连接26001实例，查看MGR切换情况
```sql
#查看当前节点，发现26000实例已经从组成员中移除
mysql>  select * from performance_schema.replication_group_members ;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | ae2d0e0e-68c9-11ea-b716-6c0b84d5c2e5 | 127.0.0.1   |       26001 | ONLINE       |
| group_replication_applier | d34f771c-68c9-11ea-89e1-6c0b84d5c2e5 | 127.0.0.1   |       26002 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
#查看节点状态，发现26001已经提升为主节点，且从只读变为了可写入
mysql> SELECT VARIABLE_VALUE  FROM performance_schema.global_status  WHERE VARIABLE_NAME = 'group_replication_primary_member';+--------------------------------------+
| VARIABLE_VALUE                       |
+--------------------------------------+
| ae2d0e0e-68c9-11ea-b716-6c0b84d5c2e5 |
+--------------------------------------+
```

- 查看TenDB Cluster集群情况
```sql
#连接TSpider 25000端口
mysql -umysql -pmysql -h127.0.0.1 -P25000
mysql> create database test_fail;
ERROR 4150 (HY000): Error happened before execute ddl in tdbctl when connect to tdbctl,  -1684530208
```
>如上图所示，集群此时DDL异常  

- 连接tdbctl 26001实例，将故障的26000剔除
```sql
mysql -umysql -pmysql -h127.0.0.1 -P26001
delete from mysql.servers where Server_name='TDBCTL0';
tdbctl flush routing;
```
- 查看TenDB Cluster集群工作情况
```sql
#连接TSpider 25000端口
mysql -umysql -pmysql -h127.0.0.1 -P25000
mysql> create database test_ok;
Query OK, 0 rows affected (0.00 sec)
```
>可以看到从路由中剔除故障的中控节点后，集群DDL操作恢复正常  

### 故障恢复
重新启动26000节点

- MGR中加入26000节点
```sql
#连接26000
mysql -umysql -pmysql -h127.0.0.1 -P26000
start group_replication;

#连接26001查看成员情况
mysql -umysql -pmysql -h127.0.0.1 -P26001
mysql> select * from performance_schema.replication_group_members ;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | ae2d0e0e-68c9-11ea-b716-6c0b84d5c2e5 | 127.0.0.1   |       26001 | ONLINE       |
| group_replication_applier | b2a68fb5-5fbf-11ea-9c1f-6c0b84d5c2e5 | 127.0.0.1   |       26000 | ONLINE       |
| group_replication_applier | d34f771c-68c9-11ea-89e1-6c0b84d5c2e5 | 127.0.0.1   |       26002 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
```
>组成员加入成功

- 路由表添加新节点
```sql
mysql -umysql -pmysql -h127.0.0.1 -P26001
mysql> insert into mysql.servers values('TDBCTL0','127.0.0.1','','mysql','mysql',26000,'','TDBCTL','');
Query OK, 1 row affected (0.00 sec)

mysql> tdbctl flush routing;
Query OK, 0 rows affected (0.00 sec)
```

- 验证集群操作
```sql
mysql -umysql -pmysql -h127.0.0.1 -P25000
mysql> create database test_ok;
Query OK, 0 rows affected (0.00 sec)

mysql> drop database test_ok;
Query OK, 0 rows affected (0.01 sec)
```
>集群操作正常