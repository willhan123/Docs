# TenDB Cluster DDL


TSpider跟普通分区表一样支持范围/hash/列表分区，TSpider扩展了partition子句的comment字段，可指定分区后端db的地址(server标识)及库表名。server字段会读取表mysql.servers中的后端TenDB ip地址和port等信息。  
当ddl_execute_by_ctl=ON的时候，开发者在TSpider节点上使用的DDL会路由给Tdbctl，由Tdbctl对集群中的TSpider+TenDB节点进行统一变更处理。  
当ddl_execute_by_ctl=OFF的时候,中控节点不会对DDL进行转发，需要分别在TSpider和后端TenDB上执行ddl操作。



# 一. 开启Tdbctl

当ddl_execute_by_ctl=ON时，由Tdbctl对集群中的TSpider+TenDB节点进行统一变更处理。  
需要注意的的是默认在：`performance_schema,information_schema,mysql,test,db_infobase` 这些库下的ddl操作，不会转发到后端TenDB。 
## 1. create table
用户建一个InnoDB表，建表SQL如下
```
create table t1(c int primary key) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
上述建表SQL会转发到中控节点Tdbctl，再由Tdbctl分发各个接入层TSpider节点和各个存储节点TenDB。在用户得到建表成功的反馈后， 执行show create table展示如下：
```
show create table t1;
```

```
Create Table: CREATE TABLE `t1` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```
此时，各存储实例TenDB分别有库tendb_test_0、tendb_test_1、tendb_test_2、tendb_test_3，每个库下都有一个t1表。


```
mysql> show create table t1\G
********************* 1. row *********************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
```
<font color="#dd0000">注意：</font>   
如上在TenDB Cluster中建一个InnoDB表，会转换为spider引擎的分区表。作为分区表，那么需要额外遵守分区表的规则：建表语句必须要有主键或者唯一键，并且分区字段必须是唯一索引的一部分。


### 1.1  MySQL 兼容性


#### 不支持空间类型 GEOMETRY
```
CREATE TABLE IF NOT EXISTS t1(f1 INT PRIMARY KEY, f3 LINESTRING NOT NULL)ENGINE=InnoDB ;
create table if not exists t1(a int not null primary key, b geometry not null, d int ) engine=innodb;
```

#### 不支持临时表
```
create temporary table t1 (c int primary key, a char(1) ) engine=innodb;
```


#### 不支持CREATE TABLE ... SELECT

```
MariaDB [tendb_test]>  create table if not exists t2(a int not null, b int, primary key(a)) engine=innodb;
MariaDB [tendb_test]>  create table if not exists t4(a int not null, b int, primary key(a)) engine=innodb select * from t2; 
ERROR 4148 (HY000): SQL TYPE: CRREATE TABLE ... SELECT ,can not be supported when set ddl_execute_by_ctl
```

#### 不支持blob列作为主键
```
CREATE TABLE bug58912 (a BLOB, b TEXT, PRIMARY KEY(a(1))) ENGINE=InnoDB;
```

#### 不支持too more unique key
```
 create table t1 (id int unsigned not null auto_increment, code tinyint unsigned not null, name char(20) not null, primary key (id), key (code), unique (name)) engine=innodb;
```

#### 必须指定key
```
create table t1 (a int) engine=innodb;
```



#### 分区字段必须是唯一索引的一部分
```
MariaDB [mysql]>  CREATE TABLE IF NOT EXISTS t1 ( id int auto_increment primary key, name varchar(32) not null, value text not null, uid int not null, unique key(name,uid))  comment='shard_key "id"' ;
ERROR 4151 (HY000): Failed to execute ddl, Error code: 12021, Detail Error Messages: CREATE TABLE ERROR: ERROR: id as TSpider key, but not in some unique key
```


#### 多个key，但是没有指定 unique key or set shard key

```
mysql> CREATE TABLE t1 ( id int(11) NOT NULL auto_increment, parent_id int(11) DEFAULT '0' NOT NULL, level tinyint(4) DEFAULT '0' NOT NULL, KEY (id), KEY parent_id (parent_id), KEY level (level) );
ERROR 4150 (HY000): Failed to execute ddl, Error code: 5000, Detail Error Messages: CREATE TABLE ERROR: ERROR: too many key more than 1, but without unique key or set shard key
```



#### no support key
```
 CREATE TABLE IF NOT EXISTS t2 (c INT NOT NULL, d INT NOT NULL, PRIMARY KEY (c,d), CONSTRAINT c1 FOREIGN KEY c2 (c) REFERENCES t1 (a) ON DELETE NO ACTION, CONSTRAINT c2 FOREIGN KEY (c) REFERENCES t1 (a) ON UPDATE NO ACTION) engine=innodb;
 ```


#### unique key没有指定not null

```
create table t1(a int unique key);
ERROR 4151 (HY000): Failed to execute ddl, Error code: 12021, Detail Error Messages: CREATE TABLE ERROR: ERROR: the key must default not null
 ```




#### 不支持外键
```
CREATE TABLE users (
 id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
 doc JSON
);
```
```
CREATE TABLE orders (
 id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
 user_id INT NOT NULL,
 doc JSON,
 FOREIGN KEY fk_user_id (user_id) REFERENCES users(id)
);

ERROR 4151 (HY000): Failed to execute ddl, Error code: 12021, Detail Error Messages: CREATE TABLE ERROR: ERROR: no support key type
```


## 2. alter table

ALTER TABLE 语句用于对已有表进行修改，以符合新表结构。ALTER TABLE 语句可用于：
```
ADD，DROP，或 RENAME 索引
ADD，DROP，MODIFY 或 CHANGE 列
```

需要注意的是，在TSpider侧，禁止alter主键

```
MariaDB [tcadmin]> CREATE TABLE IF NOT EXISTS bug47621 (salesperson INT PRIMARY KEY) ENGINE=InnoDB;
Query OK, 0 rows affected (0.027 sec)
MariaDB [tcadmin]> ALTER TABLE bug47621 CHANGE salesperson sales_acct_id INT;
ERROR 4150 (HY000): Failed to execute ddl, Error code: 5006, Detail Error Messages: DETAIL ERROR INFO: 
Spider Node Error:
Host,127.0.0.1#25000;  Error Code,1054; Error Message,Unknown column 'salesperson' in 'partition function'
Host,127.0.0.1#25001;  Error Code,1054; Error Message,Unknown column 'salesperson' in 'partition function'
```



### 2.1 ADD COLUMN
ALTER TABLE.. ADD COLUMN 语句用于在已有表中添加列。
```
MariaDB [tendb_test]> CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT);
MariaDB [tendb_test]> INSERT INTO t1 VALUES (NULL);
MariaDB [tendb_test]> SELECT * FROM t1;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)
```
```
MariaDB [tendb_test]> ALTER TABLE t1 ADD COLUMN c1 INT NOT NULL;
MariaDB [tendb_test]> ALTER TABLE t1 ADD c2 INT NOT NULL AFTER c1;
MariaDB [tendb_test]> SELECT * FROM t1;
+----+----+----+
| id | c1 | c2 |
+----+----+----+
|  1 |  0 |  0 |
+----+----+----+
1 row in set (0.00 sec)
```






### 2.2 DROP COLUMN
DROP COLUMN 语句用于从指定的表中删除列。

```
MariaDB [tendb_test]> CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, col1 INT NOT NULL, col2 INT NOT NULL);
MariaDB [tendb_test]> INSERT INTO t1 (col1,col2) VALUES (1,1),(2,2),(3,3),(4,4),(5,5);
MariaDB [tendb_test]> SELECT * FROM t1;
+----+------+------+
| id | col1 | col2 |
+----+------+------+
|  1 |    1 |    1 |
|  2 |    2 |    2 |
|  3 |    3 |    3 |
|  4 |    4 |    4 |
|  5 |    5 |    5 |
+----+------+------+
5 rows in set (0.01 sec)
```
```
MariaDB [tendb_test]> ALTER TABLE t1 DROP COLUMN col1, DROP COLUMN col2;
MariaDB [tendb_test]> SELECT * FROM t1;
+----+
| id |
+----+
|  1 |
| 18 |
| 35 |
| 52 |
| 69 |
+----+
```

###  2.3 ADD INDEX
ALTER TABLE.. ADD INDEX 语句用于在已有表中添加一个索引。

```
MariaDB [tendb_test]> CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, c1 INT NOT NULL);
MariaDB [tendb_test]> show create table t1;

CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c1` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`id`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```

```
MariaDB [tendb_test]> ALTER TABLE t1 ADD INDEX (c1);
MariaDB [tendb_test]> show create table t1;

CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c1` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `c1` (`c1`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`id`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```




### 2.4 DROP INDEX
DROP INDEX 语句用于从指定的表中删除索引。


```
MariaDB [tendb_test]> CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, c1 INT NOT NULL);
MariaDB [tendb_test]> show create table t1;

CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c1` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`id`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```

```
MariaDB [tendb_test]> CREATE INDEX c1 ON t1 (c1);
MariaDB [tendb_test]> ALTER TABLE t1 DROP INDEX c1;
MariaDB [tendb_test]> show create table t1;

CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c1` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`id`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```




### 2.5 MODIFY COLUMN
ALTER TABLE .. MODIFY COLUMN 语句用于修改已有表上的列，包括列的数据类型和属性

```
MariaDB [tendb_test]>CREATE TABLE t1 (id int not null primary key AUTO_INCREMENT, col1 INT);
MariaDB [tendb_test]>show create table  t1;

CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `col1` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```

```
MariaDB [tendb_test]>ALTER TABLE t1 MODIFY col1 BIGINT;
MariaDB [tendb_test]>ALTER TABLE t1 MODIFY col1 INT;
MariaDB [tendb_test]>ALTER TABLE t1 MODIFY col1 BLOB;
MariaDB [tendb_test]>ALTER TABLE t1 MODIFY col1 BIGINT, MODIFY id BIGINT NOT NULL;
MariaDB [tendb_test]>show create table  t1;

CREATE TABLE `t1` (
  `id` bigint(20) NOT NULL,
  `col1` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```




## 3. ALTER DATABASE

ALTER DATABASE 用于修改指定或当前数据库的默认字符集和排序规则。ALTER SCHEMA 跟 ALTER DATABASE 操作效果一样。


```
MariaDB [tendb_test]> ALTER DATABASE  tendb_test CHARACTER SET  utf8;
MariaDB [tendb_test]> SHOW CREATE DATABASE  tendb_test;
+----------------+---------------------------------------------------------------------+
| Database       | Create Database                                                     |
+----------------+---------------------------------------------------------------------+
| tendb_test     | CREATE DATABASE `tendb_test` /*!40100 DEFAULT CHARACTER SET utf8 */ |
+----------+---------------------------------------------------------------------------+
1 row in set (0.00 sec)
```






## 4. CREATE TABLE LIKE
CREATE TABLE LIKE 语句用于复制已有表的定义，但不复制任何数据。执行此SQL，TSpider会根据处理相应分片信息。

```
MariaDB [tendb_test]>create table t1(c int primary key);
MariaDB [tendb_test]>show create table t1;

 CREATE TABLE `t1` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```

```
MariaDB [tendb_test]>create table t2 like t1;
MariaDB [tendb_test]>show create table t2;

 CREATE TABLE `t2` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t2", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t2", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t2", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t2", server "SPT3"' ENGINE = SPIDER)
```







#  二. 不开启Tdbctl

当ddl_execute_by_ctl=OFF时，允许使用truncate table,但是ddl语句不会传给后端，需要分别在TSpider和TenDB上执行DDL语句。


## 1. create table 

在TSpider侧建表：

```
 MariaDB [tendb_test]> CREATE TABLE `t1` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```

在TenDB侧建表
```
mysql> create table tendb_test_0.t1(c int primary key);
mysql> create table tendb_test_1.t1(c int primary key);
mysql> create table tendb_test_2.t1(c int primary key);
mysql> create table tendb_test_3.t1(c int primary key);
```

<font color="#dd0000">注意</font>   
TSpider建表时，默认使用主键/唯一键的第一个字段作为shard_key,上面例子中`c`即为shard_key；如果没有主键/唯一键，需要显式指定 shard_key（通过表定义 COMMENT来指定shard_key）
```
MariaDB [tendb_test]> CREATE TABLE t1 (
  `c` int(11) NOT NULL
) ENGINE=SPIDER DEFAULT CHARSET=utf8 COMMENT='shard_key "c"'
 PARTITION BY LIST (crc32(`id`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)

Query OK, 0 rows affected (0.00 sec)
```



##  2. create table like 
在TSpider节点上执行create table like时，忽略每个分区中comment信息。 避免TSpider使用create table like后的误操作。


```
MariaDB [tendb_test]>create table t2 like t1;
MariaDB [tendb_test]>show create table t2;

CREATE TABLE `t2` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`a`) MOD 4)
(PARTITION `pt0` VALUES IN (0) ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) ENGINE = SPIDER)
```






