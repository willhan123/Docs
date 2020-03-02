TSpider跟普通分区表一样支持范围/hash/列表分区，TSpider扩展了partition子句的comment字段，可指定分区后端db的地址(server标识)及库表名。server字段会读取表mysql.servers中的后端db ip地址和port等信息。

当ddl_execute_by_ctl=ON的时候，开发者在TSpider节点上使用的DDL会路由给TDBCTL，由TDBCTL对集群中的TSpider+TenDB节点进行统一变更处理。performance_schema,information_schema,mysql,test,db_infobase， 默认在这些库下在ddl操作，不会转发到后端tendb。

当ddl_execute_by_ctl=OFF的时候,中控节点不会对DDL进行转发，需要分别在TSpider和后端tendb上执行ddl操作。


# ddl_execute_by_ctl=ON

当ddl_execute_by_ctl=ON时，建表语句和普通mysql没有区别。

##create table

```
create table t1(c int primary key);
```

```
show create table t1;
```

```
Create Table: CREATE TABLE `t1` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```


### MySQL 兼容性
不支持的DDL语句

#### 不支持空间类型 GEOMETRY
```
CREATE TABLE IF NOT EXISTS t1(f1 INT PRIMARY KEY, f3 LINESTRING NOT NULL)ENGINE=InnoDB ;
create table if not exists t1(a int not null primary key, b geometry not null, d int ) engine=innodb;
```

#### 不支持临时表
```
create temporary table t1 (c int primary key, a char(1) ) engine=innodb;
```


#### CREATE TABLE ... SELECT

```
mysql> create table if not exists t2(a int not null, b int, primary key(a)) engine=innodb;
mysql> create table if not exists t4(a int not null, b int, primary key(a)) engine=innodb select * from t2; 
ERROR 4148 (HY000): SQL TYPE: CRREATE TABLE ... SELECT ,can not be supported when set ddl_execute_by_ctl
```

#### blob列作为主键
```
CREATE TABLE bug58912 (a BLOB, b TEXT, PRIMARY KEY(a(1))) ENGINE=InnoDB;
```

#### too more unique key
```
 create table t1 (id int unsigned not null auto_increment, code tinyint unsigned not null, name char(20) not null, primary key (id), key (code), unique (name)) engine=innodb;
```

#### no key
```
create table t1 (a int) engine=innodb;
```


A PRIMARY KEY must include all columns in the table's partitioning function
```
CREATE TABLE IF NOT EXISTS bar (a int, b int, c char(10), PRIMARY KEY (c(3)), KEY b (b) ) engine=myisam;
Host,127.0.0.1#25000;  Error Code,1503; Error Message,A PRIMARY KEY must include all columns in the table's partitioning function
```

#### 找不到key
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
CREATE TABLE orders (
 id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
 user_id INT NOT NULL,
 doc JSON,
 FOREIGN KEY fk_user_id (user_id) REFERENCES users(id)
);

ERROR 4151 (HY000): Failed to execute ddl, Error code: 12021, Detail Error Messages: CREATE TABLE ERROR: ERROR: no support key type
```


### alter table
#### ADD COLUMN
ALTER TABLE.. ADD COLUMN 语句用于在已有表中添加列。
```
CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT);
INSERT INTO t1 VALUES (NULL);
SELECT * FROM t1;

+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)


ALTER TABLE t1 ADD COLUMN c1 INT NOT NULL;
ALTER TABLE t1 ADD c2 INT NOT NULL AFTER c1;
SELECT * FROM t1;

+----+----+----+
| id | c1 | c2 |
+----+----+----+
|  1 |  0 |  0 |
+----+----+----+
1 row in set (0.00 sec)
```

##### MySQL 兼容性
不支持将新添加的列设为 PRIMARY KEY。
不支持将新添加的列设为 AUTO_INCREMENT。




#### DROP COLUMN
DROP COLUMN 语句用于从指定的表中删除列。

```
CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, col1 INT NOT NULL, col2 INT NOT NULL);

INSERT INTO t1 (col1,col2) VALUES (1,1),(2,2),(3,3),(4,4),(5,5);
SELECT * FROM t1;
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
ALTER TABLE t1 DROP COLUMN col1, DROP COLUMN col2;
SELECT * FROM t1;

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

####  ADD INDEX
ALTER TABLE.. ADD INDEX 语句用于在已有表中添加一个索引。

```
CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, c1 INT NOT NULL);
ALTER TABLE t1 ADD INDEX (c1);
show create table t1;



CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c1` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `c1` (`c1`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`id`) MOD 1)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER)
```


##### MySQL 兼容性

无法向表中添加 PRIMARY KEY

#### DROP INDEX
DROP INDEX 语句用于从指定的表中删除索引。


```
CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, c1 INT NOT NULL);
CREATE INDEX c1 ON t1 (c1);
ALTER TABLE t1 DROP INDEX c1;
show create table t1;

CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c1` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`id`) MOD 1)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER)
```

##### MySQL 兼容性
不支持删除 PRIMARY KEY


#### MODIFY COLUMN
ALTER TABLE .. MODIFY COLUMN 语句用于修改已有表上的列，包括列的数据类型和属性

```
CREATE TABLE t1 (id int not null primary key AUTO_INCREMENT, col1 INT);
show create table  t1;



CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `col1` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test_3", table "t1", server "SPT3"' ENGINE = SPIDER)


ALTER TABLE t1 MODIFY col1 BIGINT;
ALTER TABLE t1 MODIFY col1 INT;
ALTER TABLE t1 MODIFY col1 BLOB;
ALTER TABLE t1 MODIFY col1 BIGINT, MODIFY id BIGINT NOT NULL;
show create table  t1;



CREATE TABLE `t1` (
  `id` bigint(20) NOT NULL,
  `col1` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```




### ALTER DATABASE

ALTER DATABASE 用于修改指定或当前数据库的默认字符集和排序规则。ALTER SCHEMA 跟 ALTER DATABASE 操作效果一样。


```
MariaDB [alex]> ALTER DATABASE  test CHARACTER SET  utf8;

MariaDB [alex]>  SHOW CREATE DATABASE  test;
+----------+---------------------------------------------------------------+
| Database | Create Database                                               |
+----------+---------------------------------------------------------------+
| test     | CREATE DATABASE `test` /*!40100 DEFAULT CHARACTER SET utf8 */ |
+----------+---------------------------------------------------------------+
1 row in set (0.00 sec)
```



## ALTER TABLE

ALTER TABLE 语句用于对已有表进行修改，以符合新表结构。ALTER TABLE 语句可用于：
```
ADD，DROP，或 RENAME 索引
ADD，DROP，MODIFY 或 CHANGE 列
```

### MySQL 兼容性
禁止alter主键

```
MariaDB [tcadmin]> CREATE TABLE IF NOT EXISTS bug47621 (salesperson INT PRIMARY KEY) ENGINE=InnoDB;
Query OK, 0 rows affected (0.027 sec)
MariaDB [tcadmin]> ALTER TABLE bug47621 CHANGE salesperson sales_acct_id INT;
ERROR 4150 (HY000): Failed to execute ddl, Error code: 5006, Detail Error Messages: DETAIL ERROR INFO: 
Spider Node Error:
Host,127.0.0.1#25000;  Error Code,1054; Error Message,Unknown column 'salesperson' in 'partition function'
Host,127.0.0.1#25001;  Error Code,1054; Error Message,Unknown column 'salesperson' in 'partition function'
```


## CREATE TABLE LIKE
CREATE TABLE LIKE 语句用于复制已有表的定义，但不复制任何数据。执行此SQL，Tspider会根据处理相应分片信息。

```
create table t1(c int primary key);
show create table t1;

 CREATE TABLE `t1` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test_3", table "t1", server "SPT3"' ENGINE = SPIDER)

create table t2 like t1;
show create table t2;


 CREATE TABLE `t2` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test_0", table "t2", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test_1", table "t2", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test_2", table "t2", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test_3", table "t2", server "SPT3"' ENGINE = SPIDER)

```







#  ddl_execute_by_ctl=OFF

当ddl_execute_by_ctl=OFF时，允许使用truncate table,但是ddl语句不会传给后端，需要分别在spider和tendb上执行DDL语句。


## create table 

在spider侧建表：

···
 CREATE TABLE `t1` (
  `c` int(11) NOT NULL,
  PRIMARY KEY (`c`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`c`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
···

在tendb侧建表
```
create table test_0.t1(c int primary key);
```

##  create table like 


当ddl_execute_by_ctl=OFF，需要注意的sql

create table like 

在tspider节点上执行create table like时，忽略每个分区中comment信息。 避免tspider使用create table like后的误操作。


```
create table t2 like t1;

show create table t2;

CREATE TABLE `t2` (
  `a` int(11) NOT NULL,
  PRIMARY KEY (`a`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8
 PARTITION BY LIST (crc32(`a`) MOD 1)
(PARTITION `pt0` VALUES IN (0) ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) ENGINE = SPIDER)
```






