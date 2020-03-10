## TenDB Cluster新增语法

TSpider新增了一些语法，下面将介绍如何使用

### kill thread

实现三种kill threads的方式:  
1). kill thread_id safe(kill 9 safe)  
与直接kill thread_id不同的是，只有当该thread id是sleep状态且非事务中，才会被kill,  
否则报错： ER_THREAD_IN_USING_OR_TRANSACTION , "thread %lu is in using or in transaction, can't be killed safe"  
2). kill threads all  
用来安全的kill当前threads。 sleep状态且非事务中的threads会被kill掉，其它thread则等待当前query执行完成或者事务结束才被kill  
3). kill threads all force  
用来kill当前所有运行的thread  

增加一个会话级状态: 'Max_thread_id_on_kill'， 表示在执行kill threads all (force)时，当前最大的thread id  
show status like 'Max_thread_id_on_kill';


示例SQL：
```
MariaDB [tendb_test]>  kill threads all force;
Query OK, 1 row affected (0.00 sec)

MariaDB [tendb_test]> show status like 'Max_thread_id_on_kill';
+-----------------------+--------+
| Variable_name         | Value  |
+-----------------------+--------+
| Max_thread_id_on_kill | 171010 |
+-----------------------+--------+
1 row in set (0.00 sec)
```

### 新增语法 flush  table with write lock

开启事务锁，阻塞新的事务开启,保证分布式场景下的TSpider集群主备安全切换，由unlock tables解锁。

```
执行 `flush  table with write lock`会阻塞新的事务开启。
在已有事务commit后，加锁成功，阻塞新的事务开启。
由unlock tables解锁。
```
