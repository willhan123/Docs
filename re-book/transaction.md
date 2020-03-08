# TenDB Cluster事务

TSpider支持单机事务，且通过XA协议实现分布式事务，保证应用事务在分布式场景的数据一致性。本文主要介绍涉及到事务的语句、显式/隐式事务、忽略事务，TenDB Cluster分布式事务的使用和原理，以及语法兼容性的问题。
常用的变量包括 `autocommit  SPIDER_IGNORE_AUTOCOMMIT  SPIDER_INTERNAL_XA  spider_trans_rollback`
 

## 用法

### 开启普通事务(单机事务)

#### 显示事务

```
set autocommit=0；
begin;
dml;
commit/rollback;
```

#### 隐式事务
```
set autocommit=1；
dml
```

### 忽略autocommit=0

将SPIDER_IGNORE_AUTOCOMMIT打开，这个参数打开表示忽略应用中分发的set autocommit=0；一些业务会无意使用了框架中默认的aotocommit=0（非业务预期），通过全局变量spider_ignore_autocommit，在正在执行的事务提交后，主协设置autocommit=1让后续query都变成自动提交。


### 开启分布式事务

将SPIDER_INTERNAL_XA打开，表示使用分布式事务；

注意：
 XA事务需要打开spider_trans_rollback，保证分布式事务原子性，即事务中某个SQL请求失败则事务自动回滚；保证要么全部成功，要么全部失败。

 ```
 set global SPIDER_INTERNAL_XA=1;
 begin;
 dml;
 commit/rollback;
 ```



## 分布式事务原理

开启分布式事务后，在TSpider侧执行如下SQL：

```
begin; 
insert into t1(c1, c2) values(1, 2);
update t2 set c2=3 where c1=2;
commit;
```

上述事务在发送给TSpider后会被识别并按XA协议进行转发成如下SQL（假设后端有两个存储节点TenDB）

```
tendb1: xa start xid 
tendb1:insert into tendb1.t1(c1,c2) values(1,2); 
tendb2: xa start xid 
tendb2: update tendb2.t2 set c2=3 where c1=2; 
tendb1: xa end xid; xa prepare xid; 
tendb2: xa end xid; xa prepare xid; 
tendb1: xa commit xid; 
tendb2: xa commit xid;
```


### 异常处理

上述XA事务只有全部正常执行，才是一个完整的事务流程，才能给应用返回正确。但在事务实现执行过程中各个阶段，都有可能出现软硬件异常，那么TenDB Cluster是如何保证最终的数据一致性呢 ？(注意，基于对业务需求及对性能的考虑，TenDB Cluster只保证事务的最终一致性，不严格保证事务执行过程的隔离性问题）

首先，我们定义应用层事务的3种返回结果，分别是成功，失败，超时。成功表示当前事务完全执行；失败表示当前事务失败且未修改数据；超时则表示当前事务可能成功也可能失败，需要业务主动确认（一般是查询）前事务是成功还是失败。


```
1. xa start失败：
返回失败；XA事务回滚并结束

2. 执行sql失败：
返回失败；XA事务回滚并结束

3. xa end失败：
返回失败；XA事务回滚并结束

4. xa prepare失败：
返回失败；XA事务回滚并结束；

5. xa commit/rollback失败：
返回超时；若存在悬挂事务交由悬挂事务恢复逻辑处理,执行commit or rollback；

6. xa commit one phase失败
返回超时；若存在悬挂事务交由悬挂事务恢复逻辑处理,执行commit or rollback;
```

## 语法兼容性


### 不支持save_point

```
ariaDB [test1]> create table t1(c1 int primary key);
Query OK, 0 rows affected (0.03 sec)

MariaDB [test1]> begin;
Query OK, 0 rows affected (0.00 sec)

MariaDB [test1]> insert into t1 values(1);
Query OK, 1 row affected (0.00 sec)

MariaDB [test1]> savepoint a;
ERROR 1178 (42000): The storage engine for the table doesn't support SAVEPOINT

```



### 新增语法 flush  table with write lock

开启事务锁，阻塞新的事务开启,保证分布式场景下的TSpider集群主备安全切换，由unlock tables解锁。

```
执行 `flush  table with write lock`会阻塞新的事务开启。
在已有事务commit后，加锁成功，阻塞新的事务开启。
由unlock tables解锁。
```