集群的存储层tendb是一个MySQL分支，在兼容MySQL的参数的基础上新增了一些参数。
其主要参数以及配置可以参照mysql官方手册，本文将主要介绍新增的参数。



`LOG_BIN_COMPRESS`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: Whether the binary log can be compressed

当LOG_BIN_COMPRESS=ON。表示开启Binlog压缩特性，默认不开启。
```


`LOG_BIN_COMPRESS_MIN_LEN`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 256

VARIABLE_COMMENT: Minimum length of sql statement(in statement mode) or record(in row mode)
	that can be compressed

在开启压缩功能的前提下，binlog压缩字段长度大于256的event，符合压缩条件。
```



`RELAY_LOG_UNCOMPRESS`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: ON

VARIABLE_COMMENT: Whether IO_Thread need to uncompress the relay log

默认on表示解压,Master启用binlog压缩后，slave的IO线程在拉取binlog时，可通过此参数控制是否立即解压binlog;如果为OFF，会在SQL线程执行时由SQL线程解压。
```


`DATETIME_PRECISION_USE_V1`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: This option causes CREATE TABLE to create all DATETIME/TIMESTAMP/TIME columns "
       "as old type, Without this option,

默认为off；开启后在5.6创表时,DATETIME/TIMESTAMP/TIME默认使用低版本格式(兼容低版本)
```



`BLOB_COMPRESSED`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: Set all blob/text field can be compressed when create table. 
当BLOB_COMPRESSED=ON。表示开启blob列压缩特性，默认不开启。
```



`READ_BINLOG_SPEED_LIMIT`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: OFF

VARIABLE_COMMENT: Maximum speed(KB/s) to read binlog from master (0 = no limit)

slave拉取binlog限速功能。
由于start slave后，slave会尽可能快地拉取master的binlog，可能导致master的流量较高，影响master的性能，因此新增动态参数read_binlog_speed_limit，单位是KB，可控制拉取binlog的速度。
set global read_binlog_speed_limit = 30720; //限速30M
set global read_binlog_speed_limit = 0 ; //不限速
```


`MAX_XA_COMMIT_LOGS`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 1000000

VARIABLE_COMMENT: The max rows of mysql_xa_commit_log
mysql.xa_commit_log的最大行数，此表是循环使用的
```


`sort_when_partition_prefix_order`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: TRUE
VARIABLE_COMMENT: using file sort when query with partition table + prefix index + order by
默认开启，用于修复涉及到分区表+前缀索引+order by的bug
```