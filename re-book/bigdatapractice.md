# 大数据场景应用
适合场景，写入量大，写多读少；主要是基于shard key的读取。


建议配置参数，
spider_sql_slow_log; 
spider_bgs_mode;
spider_net_read_timeout
spider_net_write_timeout
spider bgs dml   
集群部署： 凭估好分片数   
合并写入  
尽量使用堆表  
数据定期清理

### 数据定期清理
后端使用分区表

### 硬链，防抖动