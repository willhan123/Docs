

tspider兼容MySQL的参数，下文主要介绍常用配置参数，以及推荐值，和新增的参数。

# 常用参数配置


`log_sql_use_mutil_partition=0`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT:
当这个参数是true时会把跨分区query记录到慢查询中。

log the sql in spider invoke partition more than 1

```







`spider_max_connections=200`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 0

VARIABLE_COMMENT:

用于控制tspider节点对单个remote mysql的最大连接数。spider_max_connections默认为0，为不打开连接池功能； spider_max_connection配置为大于0，为指定tspider节点与单个remote mysql实例中最大的连接个数； spider_max_connections的经验值为200。

the values, as the max conncetion from spider to remote mysql. Default 0, mean unlimit the connection
```


`spider_index_hint_pushdown=1`
```

DEFAULT_VALUE: OFF

VARIABLE_SCOPE: GLOBAL|SESSION

VARIABLE_COMMENT:

增加参数index_hint_pushdown，用于控制force index 下放

switch to control if push down index hint, like force_index
```




`alter_query_log=ON`

```
DEFAULT_VALUE: OFF

VARIABLE_SCOPE: GLOBAL

VARIABLE_COMMENT:

将alter query记录到表mysql.alter_log中。
Log alter queries to a table mysql.alter_log. 
```


`spider_auto_increment_mode_switch=1`

```
DEFAULT_VALUE: OFF

VARIABLE_SCOPE: GLOBAL

VARIABLE_COMMENT: 
the switch use to control if use the spider_auto_increment_mode, default true 
用于控制是否启用spider自增列功能
```


`spider_auto_increment_step=17`
```
DEFAULT_VALUE: 17

VARIABLE_SCOPE: GLOBA

VARIABLE_COMMENT: 
the values set spider auto_increment add by spider_auto_increment_step

为步长，当前spider节点以这个步长为自增单位
```


`spider_auto_increment_mode_value=1`
```
DEFAULT_VALUE: 1

VARIABLE_SCOPE: GLOBAL

VARIABLE_COMMENT: the values, as the auto_increment mode 32. auto_increment add by 32

为当前spider节点自增列值模spider_auto_increment_step后的值，需要保证各个spider节点spider_auto_increment_mode_value不同。
```

```
TSpider主要通过三个参数维护自增列：
spider_auto_increment_mode_switch用于控制是否启用spider自增列功能
spider_auto_increment_mode_value为当前spider节点自增列值模spider_auto_increment_step后的值
spider_auto_increment_step为步长，当前spider节点以这个步长为自增单位
需要保证各个spider节点spider_auto_increment_mode_value不同。
如果某spider节点参数如下：
spider_auto_increment_mode_switch=1
spider_auto_increment_mode_value=3
spider_auto_increment_step=17
那么此spider节点上自增列依次为 3 20 37 54 ...
```






`spider_net_read_timeout=60`
```

DEFAULT_VALUE: 3600(秒)

VARIABLE_SCOPE: SESSION|GLOBAL

VARIABLE_COMMENT: Wait timeout of receiving data from remote server

等待从远程服务器接收数据的超时时间，单位为秒

```


`spider_net_write_timeout=60`
```
DEFAULT_VALUE: 3600(秒)

VARIABLE_SCOPE: SESSION|GLOBAL

VARIABLE_COMMENT: Wait timeout of sending data to remote server

等待发送数据到远程服务器超时时间, 单位为秒

```






#  新增加参数

`SPIDER_SUPPORT_XA`
```
 DEFAULT_VALUE: ON

 VARIABLE_SCOPE: GLOBAL

Description: XA Protocol for mirroring and for multi-shard transactions.
默认为ON，支持XA事务
```





`SPIDER_PARALLEL_LIMIT`
```
VARIABLE_SCOPE: GLOBA

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: spider_parallel_limit defaults is false, set spider parallel process without supporting  limit
用于控制对于包含limit、group by, order by的SQL是否需要启用并行执行功能。
```



`SPIDER_PARALLEL_GROUP_ORDER`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: ON

VARIABLE_COMMENT: spider_parallel_group_order defaults is TRUE, set spider parallel process without supporting group by, order by

用于控制对于包含group by, order by的SQL是否需要启用并行执行功能。
```


 `SPIDER_CONN_RECYCLE_MODE`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: 1

VARIABLE_COMMENT: Connection recycle mode
默认值是1，表示连接资源在当前实例中循环被使用；

```


`SPIDER_FORCE_COMMIT`
```
VARIABLE_SCOPE: Global, Session

DEFAULT_VALUE: 0

Description: Behavior when error occurs on xa prepare, xa commit, and xa rollback.
0 The error is returned.
1 The error is returned when xid doesn't exist, otherwise continues processing
2 Processing continues disregarding all the errors

控制xa prepare, xa commit, and xa rollback.阶段的行为。
0 出错即返回
1 xid不存在即返回，否则继续执行
2 继续执行，忽略错误

```


`SPIDER_INTERNAL_OFFSET`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: -1

Default Table Value: 0

Description: Skip records when acquired from the remote server.
-1 Uses the table parameter.
0 or more : Number of records to skip.

代表后端的偏移数，例如：
set SPIDER_INTERNAL_OFFSET = 2;
在spider执行 `select c from t `，将下发 `select  c from t limit 2,9223372036854775807;`到后端。默认不偏移。
```

`SPIDER_INTERNAL_LIMIT`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: -1

Description: Limits the number of records when acquired from a remote server.

-1 The table parameter is adopted.
0 or greater: Records limit.

代表请求后端记录的限制数，例如：
set SPIDER_INTERNAL_LIMIT=2；
在spider执行 `select c from t` , 将下发 ` select c from t limit 2; ` 到后端。默认不限制个数。

```

`SPIDER_SPLIT_READ`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: -1

-1 Uses the table parameter.
0 or more: Number of records.

Default Table Value: 9223372036854775807

Description: Number of records in chunk to retry the result when a range query is sent to remote backend servers.
下发到后端sql的获取的最多记录数。
默认配置，在spider执行 `select * from t1`，将下发  `select * from t1`到后端。
如果SPIDER_SPLIT_READ=3，
将下发以下sql到后端，每次最多读取3条记录：
select `c` from `alex_0`.`t1` limit 3 ;

select `c` from `alex_0`.`t1` limit 3,3; 

select `c` from `alex_0`.`t1` limit n,3;
```


`SPIDER_SEMI_SPLIT_READ`
```
VARIABLE_SCOPE: Global, Session

 DEFAULT_VALUE: 1

 VARIABLE_COMMENT: Use offset and limit parameter in SQL for split_read parameter.

 如果SPIDER_SEMI_SPLIT_READ=2， 
 在spider上执行类似limit语句 select id from test_incr limit 2;在remote每个节点是执行select id from test_incr limit 4; 每次都会将limit扩大2倍，存在无意义的性能损耗。这里是受参数spider_semi_split_read控制，为2（表示limit的两倍），现在默认值为1
```



`SPIDER_SEMI_SPLIT_READ_LIMIT`
```
VARIABLE_SCOPE: Global, Session

 DEFAULT_VALUE: -1

Description: Sets the limit value for the spider_semi_split_read system variable.
-1 Uses the table parameter.
0 or more: The limit value.

设置spider_semi_split_read系统变量的limit极限值。
例如：
spider_semi_split_read=1
spider_semi_split_read_limit=-1
在spider执行select id from  test_incr limit 4；在remote每个节点是执行select id from test_incr limit 4。
如果spider_semi_split_read_limit=2，则执行select id from test_incr limit 2;select id from test_incr limit 2,2;
```





`SPIDER_SEMI_TRX_ISOLATION`
```
VARIABLE_SCOPE: Global, Session

 DEFAULT_VALUE: -1

VARIABLE_COMMENT: Transaction isolation level during execute a sql

-1 OFF
0 READ UNCOMMITTED
1 READ COMMITTED
2 REPEATABLE READ
3 SERIALIZABLE

控制执行sql时的隔离级别：
-1 不开启
0  未提交读
1 已提交读
2 可重复读
3 可串行化
```



`SPIDER_SELUPD_LOCK_MODE`
```
VARIABLE_SCOPE: Global|Session

DEFAULT_VALUE: -1
Default Table Value: 1

VARIABLE_COMMENT: Lock for select with update
-1 Uses the table parameter.
0 Takes no locks.
1 Takes shared locks.
2 Takes exclusive locks.

决定select with update时候的加锁情况
0：不加锁
1：共享锁
2：排他锁
```


`SPIDER_SYNC_AUTOCOMMIT`
```
VARIABLE_SCOPE: Global|Session

DEFAULT_VALUE: ON

Description: Whether to push down local auto-commits to remote backend servers.
OFF Pushes down local auto-commits.
ON Doesn't push down local auto-commits.

默认值是ON， 表示会分发auto-commits sql到remote mysql；
```


`SPIDER_SYNC_TIME_ZONE`
```
VARIABLE_SCOPE: Global|Session

 DEFAULT_VALUE: ON

 Description: Whether to push the local time zone down to remote backend servers.
OFF Doesn't synchronize time zones.
ON Synchronize time zones.

默认值是true， 表示会分发时间信息到remote mysql；
```



`SPIDER_INTERNAL_SQL_LOG_OFF`
```
VARIABLE_SCOPE: Global|Session

DEFAULT_VALUE: -1

Default Table Value: 1

VARIABLE_COMMENT: Manage SQL_LOG_OFF mode statement to the data nodes

-1 Uses the table parameter.
0  Logs SQL statements to the remote server.
1  Doesn't log SQL statements to the remote server.

在 General  Log打开的场景下，控制后端tendb sql_log_off的值，从而决定sql是否写入后端General  Log。默认不写入。
```


`SPIDER_BULK_SIZE`
```
VARIABLE_SCOPE: Global|Session

 DEFAULT_VALUE: -1
Size in bytes of the buffer when multiple grouping multiple INSERT statements in a batch, (that is, bulk inserts).
-1 The table parameter is adopted.
0 or greater: Size of the buffer.
Default Session Value: -1
Default Table Value: 16000

spider bulk insert时，缓冲区的大小，单位是字节
```


`SPIDER_BULK_UPDATE_MODE`
```
VARIABLE_SCOPE: Global|Session

DEFAULT_VALUE: 2

Description: Bulk update and delete mode. Note: If you use a non-default value for the spider_bgs_mode or spider_split_read system variables, Spider sets this variable to 2.
0 Sends UPDATE and DELETE statements one by one.
1 Collects multiple UPDATE and DELETE statements, then sends the collected statements one by one.
2 Collects multiple UPDATE and DELETE statements and sends them together.

spider bulk update 和delete 的模式
```


`SPIDER_BULK_UPDATE_SIZE`
```
VARIABLE_SCOPE: Global|Session

DEFAULT_VALUE: 16000

VARIABLE_COMMENT: Bulk update size

spider bulk  update 和delete时，缓冲区的大小，单位是字节
```




`SPIDER_LOCK_EXCHANGE`
```
DEFAULT_VALUE: OFF

VARIABLE_SCOPE: Global|Session

Description: Whether to convert SELECT... LOCK IN SHARE MODE and SELECT... FOR UPDATE statements into a LOCK TABLE statement.

是否将select lock 转换为lock tables
```













`SPIDER_CONNECT_TIMEOUT`
```
VARIABLE_SCOPE: Global, Session

 DEFAULT_VALUE: -1

Default Table Value: 0

 VARIABLE_COMMENT: Wait timeout of connecting to remote server
 -1 The table parameter is adopted.
0 Less than 1.
1 and greater: Number of seconds.
spider与后端连接的超时时间
```








`SPIDER_QUICK_PAGE_SIZE`
```
VARIABLE_SCOPE: Global, Session

DEFAULT_VALUE: 1000

VARIABLE_COMMENT: Number of records in a page when acquisition one by one

当匹取数据时一次取的行数；
```




`spider_bgs_mode`
```
VARIABLE_SCOPE: SESSION|GLOBAL
DEFAULT_VALUE: 0
VARIABLE_COMMENT: Mode of background search

参数spider_bgs_mode控制是否打开并行功能
spider_bgs_mode=1 则开启并行功能 ， tspider集群在接受应用层的SQL请求后，将判定SQL需要路由到哪些具体的remote实例执行，然后并行在各个remote实例执行，最后统一汇总结果。
limit x,y、多表关联、非跨分区行为、事务等场景的，不使用并行功能。
```




`spider_bgs_dml`
```
DEFAULT_VALUE: 0

VARIABLE_SCOPE: SESSION|GLOBAL

VARIABLE_COMMENT: Mode of background dml

当spider_bgs_mode=ON时，用来控制insert  update  delete是否是并行执行的
```









`SPIDER_TRANS_ROLLBACK`
```
VARIABLE_SCOPE: SESSION

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: spider_trans_rollback is TRUE: let transaction rollback when error;  default is FALSE

在事务begin/commit过程中某query出错，后续可继续执行； 但其中某出错的query可能导致数据不一致（部分提交）；
解决该问题，某个query出错则事务回滚，保证数据一致性。 spider_trans_rollback控制该逻辑，默认关闭， 使用xa事务的环境要求打开

```

`SPIDER_WITH_BEGIN_COMMIT`
```
VARIABLE_SCOPE: SESSION|GLOBAL

 DEFAULT_VALUE: OFF

VARIABLE_COMMENT: Set if spider transmit sql to db with begin and commit.

默认为false，避免非显式事务分发begin/commit到remote mysql。即非显式事务不分发begin/commit到remote mysql，提升性能。
```















`SPIDER_CLIENT_FOUND_ROWS`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: return the matched row when use mysql_affected_rows

使用mysql_affected_rows的时候，控制是否返回匹配的行数
```



`SPIDER_DIRECT_DUP_INSERT`
```
VARIABLE_SCOPE: SESSION|GLOBAL

 DEFAULT_VALUE: 1

VARIABLE_COMMENT: Execute "REPLACE" and "INSERT IGNORE" on remote server and avoid duplicate check on local server

表示对insert on duplicate类语句时，直接分发到remote实例中，无须先执行select。 性能高，但返回的行数可能不正确；
```



`SPIDER_REMOTE_ACCESS_CHARSET`
```
VARIABLE_SCOPE: SESSION|GLOBAL

 DEFAULT_VALUE: NULL

VARIABLE_COMMENT: Set remote access charset at connecting for improvement performance of connection if you know

与后端建立连接时设置的字符集，用于提升性能
```



`SPIDER_REMOTE_AUTOCOMMIT`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: -1

Description: Sets the auto-commit mode when connecting to backend servers. This can improve connection time performance.
-1 Doesn't change the auto-commit mode.
0 Sets the auto-commit mode to 0.
1 Sets the auto-commit mode to 1.

控制改变后端auto-commit值。默认不改变。
```


`SPIDER_REMOTE_TIME_ZONE`
```
VARIABLE_SCOPE: GLOBAL

 DEFAULT_VALUE: NULL

VARIABLE_COMMENT: Set remote time_zone at connecting for improvement performance of connection if you know

与后端建立连接时设置的时区，用于提升性能
```


`SPIDER_REMOTE_SQL_LOG_OFF`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 0

VARIABLE_COMMENT: Set SQL_LOG_OFF mode on connecting for improved performance of connection, if you know

默认为1，即remote mysql在打开general_log能看到分发的query；
```



`SPIDER_REMOTE_TRX_ISOLATION`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: -1

Description: Sets the Transaction Isolation Level when connecting to the backend server.
-1 Doesn't set the Isolation Level.
0 Sets to the READ UNCOMMITTED level.
1 Sets to the READ COMMITTED level.
2 Sets to the REPEATABLE READ level.
3 Sets to the SERIALIZABLE level.

与后端建立连接时设置的隔离级别
-1 不设置
0  未提交读
1 已提交读
2 可重复读
3 可串行化
```



`SPIDER_REMOTE_DEFAULT_DATABASE`
```
VARIABLE_SCOPE: GLOBAL

 VARIABLE_COMMENT: Set remote database at connecting for improvement performance of connection if you know

 与后端建立连接时，设置的库名
```



 `SPIDER_CONNECT_RETRY_INTERVAL`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: 1000

VARIABLE_COMMENT: Connect retry interval

表示建立连接失败后重试的时间间隔；
```


`SPIDER_CONNECT_RETRY_COUNT`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: 20

VARIABLE_COMMENT: Connect retry count

表示建立连接失败后重试的次数；
```


`SPIDER_CONNECT_MUTEX`
```
VARIABLE_SCOPE: GLOBAL

 DEFAULT_VALUE: OFF

 VARIABLE_COMMENT: Use mutex at connecting

 表示建立连接时，是否使用锁
```










`SPIDER_READ_ONLY_MODE`
```
VARIABLE_SCOPE: SESSION|GLOBAL

Default Table Value: 0
DEFAULT_VALUE: -1

Description: Whether to allow writes on Spider tables.
-1 Uses the table parameter.
0 Allows writes to Spider tables.
1 Makes tables read- only.
决定spider相关表是否只读。
```


`SPIDER_GENERAL_LOG`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: Log query to remote server in general log

控制是否记录general log
```


`SPIDER_INDEX_HINT_PUSHDOWN`
```

VARIABLE_SCOPE: GLOBAL|SESSION

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: switch to control if push down index hint, like force_index

增加参数index_hint_pushdown，用于控制force index 下放

switch to control if push down index hint, like force_index
```



`SPIDER_MAX_CONNECTIONS`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 0

 VARIABLE_COMMENT: the values, as the max conncetion from spider to remote mysql. Default 0, mean unlimit the connections

用于控制tspider节点对单个remote mysql的最大连接数。spider_max_connections默认为0，为不打开连接池功能； spider_max_connection配置为大于0，为指定tspider节点与单个remote mysql实例中最大的连接个数； spider_max_connections的经验值为200。

the values, as the max conncetion from spider to remote mysql. Default 0, mean unlimit the connection
```


`SPIDER_CONN_WAIT_TIMEOUT`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 20

VARIABLE_COMMENT: the values, as the max waiting time when spider get a remote conn

表示从连接池中获取连接时的等待时间；
```


`SPIDER_IGNORE_AUTOCOMMIT`
```
 DEFAULT_VALUE: OFF

 VARIABLE_SCOPE: GLOBAL

 VARIABLE_COMMENT:

控制autocommit=0不在remote上执行

当spider_ignore_autocommit=1。若应用层(tendb)使用了autocommit=0，在启用这个参数后，当前事务提交，再次使用到该tspider到remote实例的，会自动将session级状态为设置autocommit=1（不再是应用层指定的0），后续执行的SQL都为自动提交。
```








`SPIDER_LOG_RESULT_ERRORS`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 1

VARIABLE_COMMENT: Log error from remote server in error log
默认值为1， 即tspider上执行出错信息记录到错误日志中；
```


`SPIDER_LOG_RESULT_ERROR_WITH_SQL`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 3

VARIABLE_COMMENT: Log sql at logging result errors

默认值为3， 即在执行出错时，在错误日志中记录主线程query及分发到remote执行出错的query；
```


`SPIDER_VERSION`
```
VARIABLE_SCOPE: GLOBAL

VARIABLE_COMMENT: The version of Spider

spider引擎版本号

```



`SPIDER_INTERNAL_XA_ID_TYPE`
```
VARIABLE_SCOPE: GLOBAL|SESSION

DEFAULT_VALUE: 0

VARIABLE_COMMENT: The type of internal_xa id

xid的类型
```





`SPIDER_DELETE_ALL_ROWS_TYPE`
```
VARIABLE_SCOPE: GLOBAL|SESSION

DEFAULT_VALUE: 1

VARIABLE_COMMENT: The type of delete_all_rows

默认值为1，走正常delete逻辑，会成功返回delete影响的行数；

```


`SPIDER_CONNECT_ERROR_INTERVAL`
```
VARIABLE_SCOPE: GLOBAL

 DEFAULT_VALUE: 1

VARIABLE_COMMENT: Return same error code until interval passes if connection is failed

连接失败后返回错误码的时间间隔
```


`SPIDER_QUICK_MODE_ONLY_SELECT`
```
VARIABLE_SCOPE: Global

 DEFAULT_VALUE: ON

  VARIABLE_COMMENT: let spider_quick_mode=1 effective only simple select

 控制复杂select语句（如：delete/insert into from select .. 通过limit x, y分批拉取数据）使用quick_mode为0的逻辑。
```





`SPIDER_IDLE_CONN_RECYCLE_INTERVAL`
```
ARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 3600

空闲连接回收。tspider的连接使用现状是只分配不回收，因此对空闲时间达3600秒的连接进行回收。空闲回收时间由全局变更spider_idle_conn_recycle_interval控制，默认为3600秒。
```







`SPIDER_IGNORE_XA_LOG`
```
VARIABLE_SCOPE: SESSION|GLOBAL

 DEFAULT_VALUE: ON

  VARIABLE_COMMENT: spider_ignore_xa_log defaults is TRUE, do not log the spider_xa/spider_xa_member

  ，默认为TRUE， 即不记录spider中的xa事务执行日志。 （记录日志非常消耗性能）
```
  

  `SPIDER_INTERNAL_XA`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: the switch use to control if start xa, default false 

SPIDER_INTERNAL_XA=ON 表示使用分布式事务
```




`SPIDER_IGNORE_SINGLE_SELECT_INDEX`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: ON

VARIABLE_COMMENT: spider_ignore_single_select_index defaults is TRUE, let single select in spider ignore all index

默认开启，表示在spider上，单表select不走索引
```


`SPIDER_IGNORE_SINGLE_UPDATE_INDEX`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: ON

VARIABLE_COMMENT: spider_ignore_single_update_index defaults is TRUE, let single update in spider ignore all index

默认开启，表示在spideer上，单表update不走索引
```


`SPIDER_GROUP_BY_HANDLER`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: spider_group_by_handler defaults is FALSE, disable this function

默认为fase， 表示禁止掉spider的direct join行为
```



`SPIDER_RONE_SHARD_SWITCH`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: ON

VARIABLE_COMMENT: enable the SPIDER_RONE_SHARD option effective

默认开启，表示随机访问某个分区， 比如select spider_rone_shard from tb limit 1。 随机从tb中某一个分片获取结果。
```


`SPIDER_SLOW_LOG`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: enable spider_slow_log, print each remote execute sql when print slow log

将tspider节点中记录的慢查询中增加明细，即tspider是具体是分发了哪些sql到remote mysql实例中。通过打开global参数spider_slow_log来使用。
```


`SPIDER_QUERY_ONE_SHARD`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: limit tspider select/update/delete query must specify shard_key

当spider_query_one_shard指定为true时,控制update/delete/update必须带等值的shard_key作为条件。对于两表的join，如果其中一个表指定shard_key，在spider_query_one_shard为true亦可支持。
```

`SPIDER_TRANSACTION_ONE_SHARD`
```
VARIABLE_SCOPE: GLOBAL
DEFAULT_VALUE: OFF

VARIABLE_COMMENT: limit tspider  query must use the same shard in transaction

当spider_transaction_one_shard指定为true时，控制事务中多个query必须帖路由到同一个shard中。
另外增加 config_table语法，指定某表为config_table，则跳过spider_query_one_shard的限制，但不能跳过spider_transaction_one_shard的限制。config_table用法如下。
    CREATE TABLE `t6` (
        `id` int(11) NOT NULL,
         PRIMARY KEY (`id`)
    ) ENGINE=InnoDB  COMMENT='shard_key "id", config_table "true"'

需要注意的是，前面两个参数spider_query_one_shard、spider_transaction_one_shard仅在非super权限下生效。
```


`SPIDER_IGNORE_CREATE_LIKE`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: ON

VARIABLE_COMMENT: spider_ignore_create_like defaults is TRUE, create table like ignore comment for SPIDER table

在tspider节点上执行create table like时，忽略每个分区中comment信息。 避免tspider使用create table like后的误操作。
```



`SPIDER_DIRECT_LIMIT_IN_GROUP`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: FALSE  

VARIABLE_COMMENT: limit with group would not direct send limit to remote when FALSE

默认为false，表示对gropu by + limit 的query，不分发limit 到remote中
```


`SPIDER_GET_STS_OR_CRD`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: if need to get table status

spider_get_sts_or_crd用于控制是否启用统计信息。
```


`SPIDER_MODIFY_STATUS_INTERVAL`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 28800

VARIABLE_COMMENT: the values means the interval update spider_table_status

用于控制spider_table_status的更新频率

```
```
通过后台线程将统计信息落地到各个tspider节点的spider_table_status表中，提升读取效率。
说明如下3个参数
1. spider_modify_status_interval, 全局参数，用于控制spider_table_status的更新频率，默认为8小时
2. spider_sts_interval，会话级参数，用于控制share 对象读取spider_table_status的频率，默认为3小时
3. spider_get_sts_or_crd，全局参数。用于控制是否启用统计信息，当前默认为false。
```

`DDL_EXECUTE_BY_CTL`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: use tdbctl to process ddl query

通过开启ddl_execute_by_ctl，将DDL转发给tdbctl处理
```

`TDBCTL_WRAPPER_NAME`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: TDBCTL

VARIABLE_COMMENT: wrapper name of tdbctl server

中控节点的名称，默认为"TDBCT"
```


`TDBCTL_SKIP_DDL_CONVERT_DB`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: performance_schema,information_schema,mysql,test,db_infobase

默认值为"performance_schema,information_schema,mysql,test,db_infobase"， 即这些库下在ddl操作不起ddl分发到中控节点的逻辑
```
