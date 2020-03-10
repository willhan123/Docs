# TenDBCluster手工部署
本章节描述如何在一个单机上部署TenDBCluster集群
下文将会部署一个2个TSpider节点，4个TenDB节点，1个Tdbctl节点的集群

## 部署TenDB
部署4个后端节点，端口号依次为20000~20003
下文以20000端口为例说明，其他3个实例更改相应端口号即可

### 创建配置文件
> touch /home/mysql/my.cnf.20000
```sql
[client]
port=20000
socket=/home/mysql/mysqldata/20000/mysql.sock
[mysqld]
default-storage-engine=innodb
skip-name-resolve
datadir=/home/mysql/mysqldata/20000/data
character-set-server=utf8
innodb_data_home_dir=/home/mysql/mysqldata/20000/innodb/data
innodb_file_per_table=1
innodb_flush_log_at_trx_commit=0
innodb_lock_wait_timeout=50
innodb_log_buffer_size=2048M
innodb_log_file_size=256M
innodb_log_files_in_group=4
innodb_log_group_home_dir=/home/mysql/mysqldata/20000/innodb/log
slow_query_log_file=/home/mysql/mysqllog/20000/slow-query.log
max_binlog_size=256M
port=20000
relay-log=/home/mysql/mysqldata/20000/relay-log/relay-log.bin
replicate-ignore-db=mysql
replicate-wild-ignore-table=mysql.%
server_id=241
skip-external-locking
socket=/home/mysql/mysqldata/20000/mysql.sock
tmpdir=/home/mysql/mysqldata/20000/tmp
wait_timeout=86400
[mysql]
default-character-set=utf8
port=20000
socket=/home/mysql/mysqldata/20000/mysql.sock                                          
```
### 创建目录
```bash
#创建TenDB初始化时需要的目录
mkdir /home/mysql/mysqldata/20000/data
mkdir -p /home/mysql/mysqldata/20000/innodb/data
mkdir -p /home/mysql/mysqldata/20000/innodb/log
mkdir -p /home/mysql/mysqldata/20000/relay-log
mkdir -p /home/mysql/mysqldata/20000/tmp
```
### 安装TenDB
下载TenDB介质包到/usr/local
```bash
cd /usr/local
#解压介质
tar xzvf  mysql-5.7.20-linux-x86_64-tmysql-3.1.5-gcs.tar.gz
ln -s mysql-5.7.20-linux-x86_64-tmysql-3.1.5-gcs mysql
chown -R mysql mysql-5.7.20-linux-x86_64-tmysql-3.1.5-gcs
# 初始化mysql
cd /usr/local/mysql && ./bin/mysqld --defaults-file=/home/mysql/my.cnf.20000 --initialize-insecure --user=mysql
# 启动mysql
./bin/mysqld_safe --defaults-file=/home/mysql/my.cnf.20000 --user=mysql &
```
### 部署另外3个实例
重复步骤1,2,3部署其他20001~20003实例，部署前需要替换my.cnf文件中的端口号等

## 部署TSpider
部署2个接入层节点TSpider，端口号依次为25000, 25001
下文以25000端口为例说明，25001实例更改相应端口号即可

### 创建配置文件
```bash
touch /home/mysql/my.cnf.25000
```
配置文件内容如下
```sql
[client]
port=25000
socket=/home/mysql/mysqldata/25000/mysql.sock
[mysqld]
ddl_execute_by_ctl=on
default-storage-engine=innodb
skip-name-resolve
performance_schema=OFF
log_slow_admin_statements=ON
alter_query_log=ON
query_response_time_stats=ON
innodb_flush_method=O_DIRECT
slow_query_log=1
core-file
datadir=/home/mysql/mysqldata/25000/data
character-set-server=utf8mb4
innodb_buffer_pool_size=120M
innodb_data_file_path=ibdata1:1G:autoextend
innodb_data_home_dir=/home/mysql/mysqldata/25000/innodb/data
innodb_file_per_table=1
innodb_flush_log_at_trx_commit=0
innodb_log_files_in_group=4
innodb_log_group_home_dir=/home/mysql/mysqldata/25000/innodb/log
log_bin_trust_function_creators=1
slow_query_log_file=/home/mysql/mysqldata/25000/slow-query.log
port=25000
replicate-wild-ignore-table=mysql.%
skip-external-locking
skip-symbolic-links
socket=/home/mysql/mysqldata/25000/mysql.sock
spider_bgs_mode=1
spider_get_sts_or_crd=1
spider_index_hint_pushdown=1
tmpdir=/home/mysql/mysqldata/25000/tmp
sql_mode=''
spider_auto_increment_mode_switch=1
spider_auto_increment_mode_value=1
spider_auto_increment_step=37
[mysql]
default-character-set=utf8mb4
no-auto-rehash
port=25000
socket=/home/mysql/mysqldata/25000/mysql.sock                                        
```
#### 重要参数说明
- ddl_execute_by_ctl  
集群支持DDL变更下，此参数必须在TSpider节点开启，确保在此节点执行的DDL语句会转发给Tdbctl节点，由tdbctl节点同步DDL语句到集群的各个TSpider节点和TenDB节点


### 创建目录
```
mkdir -p /home/mysql/mysqldata/25000/data
mkdir -p /home/mysql/mysqldata/25000/innodb/data
mkdir -p /home/mysql/mysqldata/25000/innodb/log
mkdir -p /home/mysql/mysqldata/25000/tmp
```

### 安装TSpider
下载TSpider介质包到/usr/local
```bash
cd /usr/local
#解压软链介质
tar xzvf  mariadb-10.3.7-linux-x86_64-tspider-3.4.5-gcs.tar.gz
ln -s mariadb-10.3.7-linux-x86_64-tspider-3.4.5-gcs tspider
chown -R tspider mariadb-10.3.7-linux-x86_64-tspider-3.4.5-gcs
# 初始化TSpider
cd /usr/local/tspider && ./scripts/mysql_install_db --defaults-file=/home/mysql/my.cnf.25000
# 启动TSpider
./bin/mysqld_safe --defaults-file=/home/mysql/my.cnf.25000 --user=mysql &
```
### 部署其他节点
重复步骤1,2,3部署其他25000实例，部署前需要替换my.cnf文件中的端口号，同时修改spider_auto_increment_mode_value=2

## 部署Tdbctl
部署1个中控节点Tdbctl，端口号为26000
### 创建配置文件
```bash
touch /home/mysql/my.cnf.26000
```
配置内容如下
```sql
[client]
port=26000
socket=/home/mysql/mysqldata/26000/mysql.sock
[mysqld]
datadir=/home/mysql/mysqldata/26000/data
socket=/home/mysql/mysqldata/26000/mysql.sock
port=26000
[mysql]
port=26000
socket=/home/mysql/mysqldata/26000/mysql.sock                                       
```
### 创建目录
```bash
mkdir -p /home/mysql/mysqldata/26000/data
```
### 安装Tdbctl
下载Tdbctl介质包到/usr/local

```bash
cd /usr/local
#解压软链介质
tar xzvf  tdbctl-1.4-linux-x86_64.tar.gz
ln -s tdbctl-1.4-linux-x86_64 tdbctl
chown -R tspider mariadb-10.3.7-linux-x86_64-tspider-3.4.5-gcs
# 初始化Tdbctl
cd /usr/local/tdbctl && ./bin/mysqld --defaults-file=/home/mysql/my.cnf.26000 --initialize-insecure --user=mysql
# 启动Tdbctl
./bin/mysqld_safe --defaults-file=/home/mysql/my.cnf.26000 --user=mysql &
```

## 配置集群
示例权限仅供参考，实际权限控制需结合应用访问安全考虑

### 配置TenDB权限
分别连接TenDB 20000~20003授权，下面以20000端口为例，其他实例替换相应端口操作即可
> mysql -uroot --socket=/home/mysql/mysqldata/20000/mysql.sock
```sql
create user mysql@'127.0.0.1' identified by 'mysql';
grant all privileges on *.* to mysql@'127.0.0.1';
```
### 配置Tdbctl权限
>mysql -uroot --socket=/home/mysql/mysqldata/26000/mysql.sock
```sql
create user mysql@'127.0.0.1' identified by 'mysql';
grant all privileges on *.* to mysql@'127.0.0.1';
```
### 配置TSpider权限
分别连接TSpider 25000, 25001授权
下面以25000端口为例,25001替换相应端口即可
>mysql -uroot --socket=/home/mysql/mysqldata/25000/mysql.sock
```sql
create user mysql@'127.0.0.1' identified by 'mysql';
grant all privileges on *.* to mysql@'127.0.0.1';
```

### 配置集群
连接Tdbctl节点，配置mysql.servers路由表
>mysql -umysql -pmysql -h127.0.0.1 -P26000
```sql
#插入路由信息
insert into mysql.servers values('SPT0','127.0.0.1','','mysql','mysql',20000,'','mysql','');
insert into mysql.servers values('SPT1','127.0.0.1','','mysql','mysql',20001,'','mysql','');
insert into mysql.servers values('SPT2','127.0.0.1','','mysql','mysql',20002,'','mysql','');
insert into mysql.servers values('SPT3','127.0.0.1','','mysql','mysql',20003,'','mysql','');
insert into mysql.servers values('SPIDER0','127.0.0.1','','mysql','mysql',25000,'','SPIDER','');
insert into mysql.servers values('SPIDER1','127.0.0.1','','mysql','mysql',25001,'','SPIDER','');
insert into mysql.servers values('TDBCTL0','127.0.0.1','','mysql','mysql',26000,'','TDBCTL','');
#刷新路由，同步路由到TSpider节点
tdbctl flush routing;
```

### 集群验证
#### 连接集群
>mysql -umysql -pmysql -h127.0.0.1 -P25000

#### 创建库表
配置完成后，可以像操作单机MySQL一样操作集群，具体集群语法限制、扩展参见语法文档
```sql
create database test1;
use test1;
# 创建表需要指定唯一键，否则会报错
create table t1(a int primary key, b int);
# 查看分片情况
show create table t1;
#TSpider节点表结构如下，
#在原表结构基础上，多了分片信息，可以看到库名在各个TenDB分片是不同的，以_0~N结尾
CREATE TABLE `t1` (
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`a`)
) ENGINE=SPIDER DEFAULT CHARSET=utf8mb4
 PARTITION BY LIST (crc32(`a`) MOD 4)
(PARTITION `pt0` VALUES IN (0) COMMENT = 'database "test1_0", table "t1", server "SPT0"' ENGINE = SPIDER,
 PARTITION `pt1` VALUES IN (1) COMMENT = 'database "test1_1", table "t1", server "SPT1"' ENGINE = SPIDER,
 PARTITION `pt2` VALUES IN (2) COMMENT = 'database "test1_2", table "t1", server "SPT2"' ENGINE = SPIDER,
 PARTITION `pt3` VALUES IN (3) COMMENT = 'database "test1_3", table "t1", server "SPT3"' ENGINE = SPIDER)
```
连接任意一个TenDB节点查看表结构
> mysql -umysql -pmysql -h127.0.0.1 -P20000
```sql
#在TenDB节点，库名会跟TSpider多一个后缀_N,
use test1_0;
show create table t1;
#一个正常的InnoDB表，字段定义与TSpider相同
CREATE TABLE `t1` (
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

#### DDL & DML 操作
通过连接TSpider节点操作整个集群，sql语法与单机MySQL大致相同
>mysql -umysql -pmysql -h127.0.0.1 -P25000
```sql
use test1;
#crc32(a)%4,插入如下数据，确保每个分片都存在一条
insert into t1 values(1,2);
insert into t1 values(2,3);
insert into t1 values(4,5);
insert into t1 values(7,8);
select * from t1;
+---+------+
| a | b    |
+---+------+
| 4 |    5 |
| 2 |    3 |
| 7 |    8 |
| 1 |    2 |
+---+------+
```
连接20000端口的TenDB查看
>mysql -umysql -pmysql -h127.0.0.1 -P20000
```sql
use test1_0;
select * from t1;
#存在一条数据crc(4)%4为0
+---+------+
| a | b    |
+---+------+
| 4 |    5 |
+---+------+
```
连接Tspider25000端口清理库
>mysql -umysql -pmysql -h127.0.0.1 -P25000
```sql
drop database test1;
```
