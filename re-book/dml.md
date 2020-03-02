

开发者在TSpider节点上使用的DML会直接路由给后端tendb节点,本文将以test_tspider为例




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

SELECT 语句用于从后端 TenDB 读取数据。


### 指定分区键
select * from test_spider where id = 1 and level > 3; 
update test_spider set level = 10 where id = 2;
根据分区规则，语法分析识别分区键，将语句路由到指定后端。

### 不带分区键
select * from test_spider where level > 3;
select * from test_spider where age = 5;
update test_spider set level = 3 where level = 2;
所有后端执行，结果合并返回。


### order by
select * from test_spider where level > 3 order by age desc;
所有后端执行
select * from test_spider where level > 3, TSpider负责排序。


### count/min/max/sum 
select count(*) from test_spider where age > 10 group by level;
所有后端执行 
select count(*), level from test_spider where age > 10 group by level.
TSpider负责将相同level结果相加后返回。


### join
select t1.* from t1, t2 where t1.c1 = t2.c1 and t2.c2 = 'a' and t1.c2 = 'b';
连接在mysql优化器中有不同的执行计划。



#### MySQL 兼容性

##### 随机访问
```
新增全局参数SPIDER_RONE_SHARD_SWITCH，默认开启，表示随机访问某个分区， 
比如select spider_rone_shard from tb limit 1。 随机从tb中某一个分片获取结果。
```

##### 并行功能

```
新增参数spider_bgs_mode，默认不开启，控制是否打开并行功能。tspider集群在接受应用层的SQL请求后，将判定SQL需要路由到哪些具体的remote实例执行，然后并行在各个remote实例执行，最后统一汇总结果。limit x,y、多表关联、非跨分区行为、事务等场景的，不使用并行功能。
```

## DML
INSERT DELETE  REPLACE 语句与 MySQL 完全兼容

### MySQL 兼容性

#### 并行功能

```
新增参数spider_bgs_dml，默认不开启，当spider_bgs_mode=ON时，用来控制insert  update  delete是否是并行执行的。
```

#### shard_key

```
当spider_query_one_shard指定为true时,控制update/delete/update必须带等值的shard_key作为条件。对于两表的join，如果其中一个表指定shard_key，在spider_query_one_shard为true亦可支持。

      当新增global参数spider_transaction_one_shard指定为true时生效,控制事务中多个query必须帖路由到同一个shard中。
       另外增加 config_table语法，指定某表为config_table，则跳过spider_query_one_shard的限制，但不能跳过spider_transaction_one_shard的限制。config_table用法如下。
    CREATE TABLE `t6` (
        `id` int(11) NOT NULL,
         PRIMARY KEY (`id`)
    ) ENGINE=InnoDB  COMMENT='shard_key "id", config_table "true"'

      需要注意的是，前面两个参数spider_query_one_shard、spider_transaction_one_shard仅在非super权限下生效。
```


## 建议
TSpider有哪些限制？
TSpider对SQL等值查询/更新支持得很好，接入节点等值查询路由到后端单个存储节点，通过分布式实现CPU、磁盘、内存的扩展；TSpider也支持范围查询或跨分区查询及更新，但效率会差些，跨表查询的效率最差；对于偶尔使用跨分区、跨表查询的业务，我们还是推荐使用TSpider；

建议：
1.不频繁有对主键的更新（频繁使用会影响性能，下同）
2.不频繁使用join操作，若使用join要求业务通过straight_join指定驱动表
3.不频繁使用聚集函数（avg，group by, having）