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
表示binlog在IO线程中解压。

`set global relay_log_uncompress = OFF`  
表示binlog在sql线程中解压 。

>如果slave机器，relay-log空间够，可以set global relay_log_uncompress=ON(默认配置)，在IO线程中解压。  
如果slave机器，relay-log空间不够，可以set global relay_log_uncompress=OFF，在sql线程中解压，但是这样可能会导致sql线程变慢。  
不过目前slave都有并行同步能力，一般不对sql线程速度造成较大影响。


## 兼容性

新增了4中事件类型，表示statement模式和row模式下的write和updates事件压缩后的event:
```
  QUERY_COMPRESSED_EVENT
  WRITE_ROWS_COMPRESSED_EVENT
  UPDATE_ROWS_COMPRESSED_EVENT
  UPDATE_ROWS_COMPRESSED_EVENT 
```
所以推荐使用TenDB的mysqlbinlog工具进行binlog操作。


