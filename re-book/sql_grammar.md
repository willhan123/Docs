TSpider能够兼容绝多MySQL标准用法，但还是有些地方和单实例MySQL有一些差异，下文将阐述Tspider新增的语法，以及与MySQL的兼容性差异。




## 新增语法

### kill thread

实现三种kill threads的方式：
1). kill thread_id safe(kill 9 safe)
与直接kill thread_id不同的是，只有当该thread id是sleep状态且非事务中，才会被kill,
否则报错： ER_THREAD_IN_USING_OR_TRANSACTION , "thread %lu is in using or in transaction, can't be killed safe"
2). kill threads all
用来安全的kill当前threads。 sleep状态且非事务中的threads会被kill掉，其它thread则等待当前query执行完成或者事务结束才被kill
3). kill threads all force
用来kill当前所有运行的thread

另外，增加一个会话级状态: 'Max_thread_id_on_kill'， 表示在执行kill threads all (force)时，当前最大的thread id
show status like 'Max_thread_id_on_kill';



### 新增语法 flush  table with write lock

开启事务锁，阻塞新的事务开启,保证分布式场景下的spider集群主备安全切换，由unlock tables解锁。

```
执行 `flush  table with write lock`会阻塞新的事务开启。
在已有事务commit后，加锁成功，阻塞新的事务开启。
由unlock tables解锁。
```



###  中控节点切换指令

切换指令是指当增加对TSpider节点进行扩缩、对TenDB节点进行扩缩及故障处理时需要使用的指令。

TDBCTL FLUSH ROUTING [ FORCE ] 用于主动将路由信息刷新的到各个tspider节点；

TDBCTL FLUSH ROUTING SERVER SERVER_NAME [ FORCE ] 用于主动将路由信息刷新到某个特定SERVER_NAME节点；

TDBCTL DROP [TSPIDER|TENDB|CENTER] ROUTING SERVER SERVER_NAME ，用于主动删除某个节点路由信息。

TDBCTL CHANGE [TSPIDER|TENDB|CENTER] SERVER SERVER_NAME ROUTING OPTIONS ，用于change某个节点的路由信息，默认为改变TENDB节点路由信息


其中OPTIONS定义如下：
```
option: 
{ HOST character-literal 
| DATABASE character-literal 
| USER character-literal 
| PASSWORD character-literal 
| SOCKET character-literal 
| OWNER character-literal 
| PORT numeric-literal }
```


## TSpider跟原生MySQL的差异：



| 功能 | TSpider | MySQL | 备注 |
| :-----| :----- | :----- | :----- |
| 存储过程、函数、视图支持 | 支持 | 支持 | 在TSpider线上环境一般会禁用 |
| distinct、join、not in效率 | 差 | 一般 | 在TSpider中Join的时候驱动表选择非常重要；|
| 走PK（ShardKey）的SQL | 定位到某个远程节点执行，效率高 | 本地执行，效率高 | 可以认为是一致的，无差别 |
| 非走PK（ShardKey）的SQL | 在远程节点执行，结果合并后返回，效率低 | 本地执行，效率高 | TSpider的劣势，效率差 |
| 自增列	 | 局部递增，全局唯一，且必须是bigint类型 | 顺序递增 | 自增列要求bigint |
| 主键+唯一键 | 只能有主键或者一个唯一键 | 允许多个唯一键 | TSpider的表结构中若包含两个及以上的唯一键(含主键)，只能保留一个唯一键，其他唯一键需变更为普通索引,应用层来保证唯一性 |
| 外键约束 | 不支持 | 支持	 |  |
| DDL | 开启ddl_execute_by_ctl时，由中控节点分发DDL语句到后端 | 灵活支持 | 当ddl_execute_by_ctl=OFF时，允许使用truncate table,但是ddl语句不会传给后端，需要分别在spider和tendb上执行DDL语句。 |
| 分区键更新 | 禁止 | 支持 | 在TSpider中，主键或者分区键最好不要涉及更新 |
| limit x,y(不带order by) | 从RemoteDB节点拉取一部分数据在Tspider汇集后返回 | 直接取,按主键顺序返回 | limit x,y用法通常用于业务逻辑中分批，分页拉取数据。当x非常大的时候，要拉取x+y的数据再取得y行数据，性能很差。TSpider通过在limit x,y时先对相关分片count(*)计算行数，优化了查询效率 |
| limit x,y(带order by) | 从RemoteDB节点各拉取一部分数据在TSpider排序后再取 | 按要求排序后再取 | 需要按序更新或删除，需要先select后逐行update/delete |
| save_point | 不支持 | 支持 |  |
| Innodb特有的操作，如不支持FLUSH TABLES tbl_name ... FOR EXPORT | 不支持 | 支持 |  |




##  TSpider使用要求

业务中不允许表的唯一键（含主键）超过一个，在存在主键的情况下，其他索引只能为普通索引
不推荐使用存储过程、函数、视图
不频繁有对主键的更新，频繁使用会影响性能
不频繁使用join操作，若使用join要求业务通过straight_join指定驱动表
不频繁使用聚集函数（avg，group by, having），使用会影响性能
如果有自增列，必须使用bigint类型，只保证唯一，不保证递增
建表不应该使用外键约束