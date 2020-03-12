# 大数据场景应用

基于TenDB Cluster透明分库分表、动态扩缩容的特点， TenDB Cluster也很适合大数据量的场景。本文提供一些大数据量集群的运营经验。

## 应用特点
1. 数据量大
2. 写多读少
3. 点查询为主
4. 基本没有复杂计算

## 集群规模

TenDB Cluster集群如果想调整分片数，需要对全部数据进行重导，大数据量下这个运营成本很高。因此提前规划好集群分片数尤为重要，通常可以按需求的1.5倍到2倍来评估分片数。 为了便于存储实例扩缩，要选择公约数较多的数作为初始分片数，比如48、64、72分片。

## 配置
spider_sql_slow_log    
spider_bgs_mode    
spider_net_read_timeout    
spider_net_write_timeout   
spider bgs dml   

## 表结构

## 存储实例维护

### 数据定期清理

后端使用分区表

### 列压缩
列压缩，其它引擎

### 硬链，防抖动

### 备份
物理备份 
可选： 堆表（无主键）

有时间字段作为shard key

后端也使用分区表

适合场景，写入量大，写多读少；主要是基于shard key的读取。
流水日志场景（数据定期清理）

##

流水日志场景，写入量巨大，写多读少， 数据定期清理；
可选： 堆表（无主键）

有时间字段作为shard key

后端也使用分区表

适合场景，写入量大，写多读少；主要是基于shard key的读取。
流水日志场景（数据定期清理）

## 配置建议
spider_sql_slow_log    
spider_bgs_mode    
spider_net_read_timeout    
spider_net_write_timeout   
spider bgs dml   
集群部署： 凭估好分片数   
合并写入  
尽量使用堆表  
数据定期清理   

## 数据定期清理

后端使用分区表

## 列压缩

## 硬链，防抖动