<a id="jump"></a>

# Difference with MySQL single instance
A TenDB cluster is made up of multiple proxy layer TSpiders, and storage layer TenDB instances, which contain distributed data by sharding rules. 
Although TSpider leverages MySQL's spider storage engine, and is able to support most of the standard MySQL commands, there are several unsupported commands. Some features may differ to single instance MySQL.
The differences are summarized in the below table:
|    | MySQL   | Tendb Cluster   |
|:----|:----|:----|
| Multiple unique keys   | Supported   | Unsupported   |
| Foreign key constraint   | Supported   | Unsupported   |
| Change table shard key | Supported   | Unsupported  |
| CREATE TABLE tblName AS SELECT| Supported   | Unsupported |
| CREATE TEMPORARY TABLE   | Supported      | Unsupported   |
| XA transaction | Supported    | Unsupported |
| SAVEPOINT | Supported   | Unsupported |
| Auto-increment column | Supported   | Supported   (TSpider only guarantees uniqueness of auto-increment columns, and no guarantee that they are ordered and increment globally)
| Stored Procedure/Trigger/Function/View   | Supported   | Supported (But not suggested)   |
| KILL THREADS ALL/ALL SAFE| Unsupported | Supported |
| KILL thread_id SAFE | Unsupported | Supported |

More details are discussed as below:
<a id="jump1"></a>  
## Unsupported features
Multiple unique keys
Foreign key constraint
Change table shard key
Add/delete primary key
CREATE TABLE tblName AS SELECT statement
CREATE TEMPORARY TABLE
XA Transactions
SAVEPOINT
## Features differs to MySQL
<a id="jump21"></a>
### **DDL**
Differences between TSpider and MySQL are discussed in CREATE TABLE AND ALTER TABLE statements, respectively as below
#### CREATE TABLE
Create an innodb table with below SQL statement:
```
CREATE TABLE `t1` (
  `c1` int(11) NOT NULL,
  `c2` int(11) DEFAULT NULL,
  PRIMARY KEY (`c1`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```
Above SQL will be sent to central admin node Tdbctl, then distributed by Tdbctl to proxy layer TSPider nodes and storage layer TenDB nodes. After user got creation success confirmation, issue show create table will get below output:
```
mysql> show create table t1\G
********************* 1. row**********************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `c1` int(11) NOT NULL,
  `c2` int(11) DEFAULT NULL,
  PRIMARY KEY (`c1`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8mb4
 PARTITION BY LIST (crc32(`c1`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "tendb_test_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) COMMENT = 'database "tendb_test_1", table "t1", server "SPT1"' ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) COMMENT = 'database "tendb_test_2", table "t1", server "SPT2"' ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) COMMENT = 'database "tendb_test_3", table "t1", server "SPT3"' ENGINE = SPIDER)
1 row in set (0.00 sec)
```
Unlike single instance MySQL, show create table yields a table definition partitioned by primary key, and the storage engine is spider.
Users are able to access the t1 table and perform CRUD operations in TSpider like in InnoDB.
After creation, database tendb_test_0, tendb_test_1, tendb_test_2, and tendb_test_3 will be created on TenDB nodes  respectively. And each database has a t1 table.
```
mysql> show create table tendb_test_0.t1\G
********************* 1. row *********************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `c1` int(11) NOT NULL,
  `c2` int(11) DEFAULT NULL,
  PRIMARY KEY (`c1`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)
```
TenDB cluster exposed appropriate inner details to user, which may help user to understand TSpider better thus to utilize it better.

#### ALTER TABLE
As demonstrated above, An InnoDB table created in TenDB cluster will be transformed into a partitioned table using spider engine. As a partitioned table, certain rules should be followed, such as partition key must be part of a unique index. Therefore below SQL that adds unique key will not be supported:
```
mysql> alter table t1 add unique(c2);
ERROR 1503 (HY000): A UNIQUE INDEX must include all columns in the table's partitioning function
```
However, if the new unique key contains the partition key, the alter table statement can be accepted.
```
mysql> alter table t1 add unique(c1,c2);
Query OK, 0 rows affected (0.01 sec)
```
The consideration behind this constraint is that partitioned tables can not make sure that the uniqueness still holds after merging data from other partitions. For the same reason, Each partitioned table corresponds to one TenDB storage instance, and TenDB cluster can not guarantee data uniqueness after merging data from instances neither.
<a id="jump22"></a>
### **Storage engine**
Tables created in TenDB cluster with InnoDB/MyISAM/etc. will be transformed into spider engine unanimously, by TSpider proxy layer. Whereas the TenDB instance who stores the actual data will use the storage engine specified by users.
<a id="jump23"></a>
### **Auto increment column**
TenDB cluster guarantees only the uniqueness of the auto-increment column, and no guarantee for continuity and incremental are made.
In TenDB cluster, a table will be distributed in multiple TenDB storage instances, and multiple TSpider nodes will read and write this table equally and simultaneously. Differing from single instance MySQL, an auto-increment column needs to be maintained by multiple TenDB instances together.
To achieve the same behavior with that in single instance MySQL, All tables in TenDB cluster must be locked when TSpider nodes update auto-increment column, or use third-party resource as a critical resource to ensure the uniqueness. Both solutions will introduce noticeable performance overhead, therefore it is not suitable for production environments.
The implementation strategy of auto-increment in TenDB cluster is, to let each TSpider node to maintain its own auto-increment column, and to make sure that the increment generated in each TSpider node is different to that generated in other nodes. Due to this autonomy, it can not be guaranteed that the auto-increment column is continuous and incremental in cluster aspect, however, this approach has low cost in maintaining auto-increment column and high efficiency during updating them.
There are 3 parameters in each TSpider related to auto-increment column:
> spider_auto_increment_mode_switch: wether to enable auto-increment column feature on TSpider
> spider_auto_increment_step: step size of TSpider auto-increment
> spider_auto_increment_mode_value: the remainder of auto-increment column value modulo of spider_auto_increment_step

With auto-increment column enabled, given spider_auto_increment_step size of 17, and spider_auto_increment_mode_value=3, then the auto-increment sequence generated on that TSpider node will be 3, 3+17, 3+17+17, ...
> According to above, with the same spider_auto_increment_mode_switch setting, TSpider nodes within a cluster should have the same spider_auto_increment_step and differet spider_auto_increment_mode_value.

<a id="jump24"></a>

### **Query plan**
Proxy layer of TenDB cluster focuses on how to forward users requests to backend storage efficiently, instead of how the SQL will be executed on the storage node. Therefore when reviewing a query plan on TSpider with explain, you may find even a query with a primary key can request a full table scan, in below example, where t1 table have a primary key c1, and explain results as followed:

```
mysql> explain select * from t1 where c1=1;
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra                             |
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------+
|    1 | SIMPLE      | t1    | ALL  | NULL          | NULL | NULL    | NULL |    2 | Using where with pushed condition |
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------+
```

<a id="jump25"></a>

### **Transaction**
TenDB cluster does not support XA transaction, nor savepoint feature.
for ordinary transactions, by default Tendb cluster supports only single shard transaction to gain better performance. Only with parameter spider_internal_xa enabled, TSpider will transform single instance transaction to XA protocol to coordinate storage nodes. To enable distributed transactions, parameter spider_trans_rollback should be used along with spider_internal_xa. See below example:
When an application initiates below transaction:

```
begin; 
insert into t1(c1, c2) values(1, 2);
update t2 set c2=3 where c1=2;
commit;
```
TSpider will transform above transaction to interact TenDB with XA protocol, SQLs behind the scenes(table t1 has two shards):
```
TenDB1: xa start xid;
TenDB1: insert into rm1.t1(c1,c2) values(1,2); 
TenDB2: xa start xid;
TenDB2: update rm2.t2 set c2=3 where c1=2; 
TenDB1: xa end xid; xa prepare xid; 
TenDB2: xa end xid; xa prepare xid; 
TenDB1: xa commit xid one phase with logs; 
TenDB2: xa commit xid;
```
After the application initiating a transaction, for the first request(insert statement above), TSpider will send xa start to backend, then execute insert on one of the storage nodes, based on the routing rule. All succeeding requests(such as update in this example) will be distributed to certain TenDB node. When application commit or rollback, TSpider use xa end or xa prepare to pre-commit transaction and then commit it using xa commit.
<a id="jump251"></a>

#### **xa commit xid one phase with logs**
When committing above transaction, the statement executed in TenDB1 is xa commit xid one phase with logs. This is an extended feature to XA implemented by TenDB. Its purpose is to commit XA transaction and append xid to the original mysql.xa_commit_log.

The reason why the xid needs to be recorded in log is that software or hardware failure may be encountered during XA transaction execution, in which situation the xa prepare transaction needs a xa_commit_log as a start point to resume. Hanging transactions with xa_commit_log recorded should be committed, while hanging transactions without xa_commit_log should be rolled back.

<a id="jump252"></a>

#### **spider_trans_rollback parameter**

Transaction atomicity is defined as below:
> Multiple operations on a database within one transaction should be an indivisible atomic unit, The transaction can be committed only if all the operations succeed. Any failure among the operations will cause the entire operation set to be rolled back, to restore the database to the previous status.

However, in MySQL innodb transactions, the success SQL statements can be committed even with failure SQL statement present.
```
mysql> select * from t1;
+----+------+
| c1 | c2   |
+----+------+
|  1 | a    |
|  2 | b    |
|  3 | c    |
+----+------+
3 rows in set (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t1 values(4,'d');
Query OK, 1 row affected (0.00 sec)

mysql> update t1 set c2='aaa' where c3=1;
ERROR 1054 (42S22): Unknown column 'c3' in 'where clause'
mysql> commit;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t1;
+----+------+
| c1 | c2   |
+----+------+
|  1 | a    |
|  2 | b    |
|  3 | c    |
|  4 | d    |
+----+------+
4 rows in set (0.00 sec)
```
When a transaction is executed in TenDB cluster, several reasons may lead to update failure, one of the reasons is that the update statement executed successfully in some of the shards while failed in others.

According to innodb's transaction protocol, the following SQL can be committed or rolled back normally, which may cause a SQL(such as the update statement above) to be partially executed on the cluster.

If transaction atomicity to be enforced strictly, a transaction should be rolled back immediately without proceeding on any failure SQL.

with parameter *spider_trans_rollback* switched on, transaction atomicity will be enforced, SQL statements in one transaction will be executed all, or none.

<a id="jump253"></a>

#### **Transaction results returned**

TenDB cluster defines 3 types of transaction results, respectively, **Success**, **Fail**, **Timeout**.

Success indicates that all operations in a transaction are executed successfully.
Fail indicates that the entire transaction failed and the database is untouched.
Timeout indicates the transaction may be either in a state of Success or Fail, which needs to be checked by the user.

<a id="jump254"></a>

#### **Handling hanging tranction**

TenDB introduced an extended command, xa recover with time, to list transactions in prepared state and how long it has been in this state.
TenDB deem a transaction that in prepare state for more than 30 seconds as a hanging transaction, which should be handled by either commit or rollback.
In each Tdbctl there will be a background thread to look for transactions in prepared state for more than 30 seconds. Once a hanging transaction is detected, its xid will be scanned in xa_commit_log in all TenDB nodes, if found, XA commit xid will be executed, otherwise XA ROLLBACK xid will be executed. The flow chart of hanging transaction handling is as below:

![pic](../pic/xarecover.png)
<a id="jump26"></a>

### **Stored procedure and trigger**
It's not suggested to use complex stored procedures or triggers.
In scenarios where strong data consistency is required, it's suggested to use distributed transactions. Otherwise only local transactions will be supported.
