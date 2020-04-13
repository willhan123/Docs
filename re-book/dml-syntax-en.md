# TenDB Cluster DML
The DML on the TSpider node will be directly routed to the TenDB node. The below tips, use table `test_tspider` as an example to describe the usage of `select, update, insert, and delete`.

```
CREATE TABLE `test_tspider` (
  `id` bigint(20) NOT NULL,
  `name` varchar(20),
  `age` int,
  `level` int,
  PRIMARY KEY (`id`),
  KEY idx_level(`level`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
/*!50100 PARTITION BY LIST (crc32(id)%4)
(PARTITION pt0 VALUES IN (0) COMMENT = 'database "d1_0", table "test_tspider", server "SP0" ENGINE = SPIDER,
 PARTITION pt1 VALUES IN (1) COMMENT = 'database "d1_1", table "test_tspider", server "SP1" ENGINE = SPIDER,
 PARTITION pt2 VALUES IN (2) COMMENT = 'database "d1_2", table "test_tspider", server "SP2" ENGINE = SPIDER,
 PARTITION pt3 VALUES IN (3) COMMENT = 'database "d1_3", table "test_tspider", server "SP3" ENGINE = SPIDER) */
```




## SELECT
`SELECT` statement is used to get data from the TenDB node. The following will explain the usage and principle of several common types of `SELECT` statements.

### With shard_key
select * from test_spider where id = 1 ;   
If shard_key is specified in the where condition, TSpider recognizes the shard_key according to the sharding rules, and routes the sql to the designated TenDB node.

### Without shard_key
```
select * from test_spider where level > 3;  
SQL will be routed to all TenDB nodes for execution, and the result will be merged and returned on TSpider node.
```

### order by  
```
select * from test_spider where level > 3 order by age desc;  
SQL will be routed to all TenDB nodes to execute `select * from test_spider where level> 3`, and the result will be sorted on TSpider node.

```

### count/min/max/sum 
```
select count(*) from test_spider where age > 10 group by level;  
SQL will be routed to all TenDB nodes to execute  `select count(*), level from test_spider where age > 10 group by level`,
and the results of the same level will be added on TSpider node.  
```

##  INSERT

```
MariaDB [tendb_test]> insert into test_tspider(id,name,age,level) values(1,'coco',28,1);
Query OK, 1 row affected (0.01 sec)

MariaDB [tendb_test]> select age from test_tspider   where id=1;
+------+
| age  |
+------+
|   28 |
+------+
1 row in set (0.00 sec)
```


## UPDATE
```
MariaDB [tendb_test]> update test_tspider  set age=29 where id=1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [tendb_test]> select age from test_tspider   where id=1;
+------+
| age  |
+------+
|   29 |
+------+
1 row in set (0.00 sec)
```

##  DELETE
```
MariaDB [tendb_test]> delete from  test_tspider   where id=1;
Query OK, 1 row affected (0.00 sec)

MariaDB [tendb_test]>  select age from test_tspider   where id=1;
Empty set (0.00 sec)
```


>For `select, insert, update, delete`, if the shard_key is specified in the where condition, TSpider will route SQL to the designated TenDB node according to the sharding rules. If not specified, SQL will be routed to all TenDB nodes for execution.  
when `select` sql will be  routed to several TenDB nodes, if [spider_bgs_mode](tspider-parameter-en.md/#spider_bgs_mode) equals 0,
then distribute request serially; if equals 1, then distribute request parallelly.  
when `insert, update, delete` sql will be  routed to several TenDB nodes, if [spider_bgs_dml](tspider-parameter-en.md/#spider_bgs_dml) equals 0, then distribute request serially; if equals 1, then distribute request parallelly.






## New usage

### Random select
Add global parameter `SPIDER_RONE_SHARD_SWITCH`, which is default enabled, means retrieve  date from one partition randomly.
Such as `select spider_rone_shard from test_tspider limit 1` , retrieve date from one partition of test_tspider randomly.

For example:
```
MariaDB [tendb_test]> select id from test_tspider   limit 1;
+-----+
| id |
+-----+
|  28 |
+-----+
1 row in set (0.00 sec)

MariaDB [tendb_test]> select id from test_tspider   limit 1;
+-----+
| id |
+-----+
|  28 |
+-----+
1 row in set (0.00 sec)
```

```
MariaDB [tendb_test]> set global SPIDER_RONE_SHARD_SWITCH=1;
Query OK, 0 rows affected (0.01 sec)

MariaDB [tendb_test]> select  spider_rone_shard  id from test_tspider   limit 1;
+-----+
| id |
+-----+
|   4 |
+-----+
1 row in set (0.00 sec)

MariaDB [tendb_test]> select  spider_rone_shard  id from test_tspider   limit 1;
+-----+
| id |
+-----+
|  28 |
+-----+
1 row in set (0.00 sec)
```


### Specify equivalent shard_key
Add global parameter `spider_query_one_shard，spider_transaction_one_shard`
When `spider_query_one_shard=true`, restrict `update, delete, select` must have equivalent shard_key as a condition. For join sql related to 2 tables, if one of the tables specifies shard_key, it can also be supported if spider_query_one_shard is true.  
when spider_transaction_one_shard=true, restrict queries in the transaction must be routed to the same shard.  
In addition, add grammer for `config_table`, if specify a table as config_table, then skip the restriction from spider_query_one_shard, but can not skip the restriction from spider_transaction_one_shard. The usage of config_table is as follows: 
```
    CREATE TABLE `t6` (
        `id` int(11) NOT NULL,
         PRIMARY KEY (`id`)
    ) ENGINE=InnoDB  COMMENT='shard_key "id", config_table "true"'
```

SQL is as follows：
```
MariaDB [tendb_test]> set global spider_query_one_shard=1;
MariaDB [tendb_test]>  select age from test_tspider   where id>4;
ERROR 1735 (HY000): Select across multiple partitions denied to user 'use'@'ip' for table 'test_tspider'

MariaDB [tendb_test]>  update test_tspider  set age='11'  where id>4;
ERROR 1735 (HY000): Update across multiple partitions denied to user 'use'@'ip' for table 'test_tspider'
```
```
MariaDB [tendb_test]> set global spider_transaction_one_shard=1;
MariaDB [tendb_test]>  begin;
Query OK, 0 rows affected (0.00 sec)

MariaDB [tendb_test]>  select age from test_tspider   where id=1;
+-----+
| age |
+-----+
|  10 |
+-----+
1 row in set (0.00 sec)

MariaDB [tendb_test]>  update test_tspider  set age='11'  where id=2;
ERROR 1734 (HY000): Transaction across different partitions denied to user 'use'@'ip'

```
<font color="#dd0000">Note：</font>   
>spider_query_one_shard, spider_transaction_one_shard is only effective under non-super permissions.


## Suggestions
1. seldom update shard_key(update shard_key frequently affect performance, the same to the below tips)  
2. avoid full table join
3. use sum/count insead of avg
