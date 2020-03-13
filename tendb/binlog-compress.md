# TenDB binlog压缩
为了在不影响binlog各项功能的情况下，减少binlog的大小。在TenDB中,采用zlib实现了binlog压缩功能，该功能比较独立，可以在运行中开启和关闭，适用于statement，row，mixed各种格式。


## 用法
### 压缩

`set global log_bin_compress=ON `  
表示开启Binlog压缩功能

`set global log_bin_compress_min_len=256 `  
在开启压缩功能的前提下，binlog压缩字段长度大于256的event，符合压缩条件。

>启用binlog压缩功能，可以达到7~9的压缩比。在消耗较多CPU的同时，减少了IO。

### 解压

`set global relay_log_uncompress = ON`  
表示在IO线程中解压，解压后写入relaylog。

`set global relay_log_uncompress = OFF`  
拉取binlog以后直接写入relaylog，然后在执行的时候解压。

>目前默认relay_log_uncompress = ON，因为样做的好处是分担sql线程的压力，减少relay log的积压。
如果slave机器磁盘空间不足，可以set global relay_log_uncompress=OFF。


## 兼容性

新增了4中事件类型，表示statement模式和row模式下的write和updates事件压缩后的event:
```
  QUERY_COMPRESSED_EVENT
  WRITE_ROWS_COMPRESSED_EVENT
  UPDATE_ROWS_COMPRESSED_EVENT
  UPDATE_ROWS_COMPRESSED_EVENT 
```
所以推荐使用TenDB的mysqlbinlog工具进行binlog操作。


