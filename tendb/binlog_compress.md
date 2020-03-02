# TenDB binlog压缩功能介绍
## 背景

Mysql的binlog可以简单的理解为二进制日志，包括一系列描述数据修改的“event”。binlog有下面两个重要用途：
1.    用于主备同步，master把数据修改写入binlog中，然后将这些包含一系列event的binlog发送给slave，由它执行这些event来对数据做出相同的变更。

2.    用于数据恢复，当一个全量备份创建好了以后，binlog可以用event记录下备份以后执行的操作。可以用这些信息将备份的数据恢复到当前的时间点上。

但是在一个繁忙的DB上，有可能短时间产生大量的binlog，这不仅会占用大量的磁盘空间，也会给IO带来不小的压力。显然binlog是不能关闭的，但是在不影响binlog各项功能的情况下，是否有方案能减少binlog的大小呢？答案当然是可行的，在tendb中。采用zlib实现了binlog压缩功能，该功能比较独立，可以在运行中开启和关闭，适用于statement，row，mixed各种格式。

## 技术实现
为了防止低版本mysqlbinlog解析压缩后的binlog event,出现乱码且不报错的场景，新建了4中事件类型，表示statement模式和row模式下的write和updates事件压缩后的event:
```
  QUERY_COMPRESSED_EVENT
  WRITE_ROWS_COMPRESSED_EVENT
  UPDATE_ROWS_COMPRESSED_EVENT
  UPDATE_ROWS_COMPRESSED_EVENT
```

### 压缩细节
那么该压缩event的哪个部分呢？

Mysql定义的binlog的格式为：
```
+=====================================+
| event  | timestamp         0 : 4    |
| header +----------------------------+
|        | type_code         4 : 1    |
|        +----------------------------+
|        | server_id         5 : 4    |
|        +----------------------------+
|        | event_length      9 : 4    |
|        +----------------------------+
|        | next_position    13 : 4    |
|        +----------------------------+
|        | flags            17 : 2    |
|        +----------------------------+
|        | extra_headers    19 : x-19 |
+=====================================+
| event  | fixed part        x : y    |
| data   +----------------------------+
|        | variable part              |
+=====================================+
```
其中event_header部分长度固定，且读写频繁，没有必要压缩。
对于query_event事件来说它的data部分为：

```
+=======================================+
| fixed  | thread id            0 : 4   |
| part   +------------------------------+
|        | execute time         4 : 4   |
|        +------------------------------+
|        | database name len    8 : 1   |
|        +------------------------------+
|        | error code           9 : 2   |
|        +------------------------------+
|        | variable block len   11 : 2  |
+=======================================+
|Variable|      status variables        |
| part   +------------------------------+
|        |       database name          |
|        +------------------------------+
|        |       SQL statement          |
+=======================================+
```
考虑到读写频率以及压缩收益，只对SQL statement这个字段进行压缩，它就是完整的sql语句。

对于row_log_event，它的格式则比较复杂，而且write事件和update事件格式还是不一样的，最重要的是，这个格式5.5和5.6之间也有变化，因此对于这类事件采用的是压缩整个rows部分的数据，只有这样才能保证从5.5升级到5.6时无需改动代码，因为这样整体压缩是不需要感知内部格式变化的。

### 压缩格式
压缩后的格式为：

![format](../pic/compress_format.png)

 首字节标记：第一个bit，0表示未压缩（对应就无解压后长度），1表示压缩；第2，3位表示算法类型（现阶段版本只有zlib算法)，第6，7，8位表示有几个字节来存储压缩长度。

 解压后长度：表示数据在压缩前或解压“压缩的内容”的长度，1-4字节的内容分别表示长度上限为2^8-1、2^16-1、2^24-1,2^32-1。该信息也用于解压后的内存分配。

 压缩的内容：就是压缩后的数据。

 ### 解压细节
  对于解压来说，有两种选择：

  1.在IO线程中解压，解压后写入relaylog。

  2.拉取binlog以后直接写入relaylog，然后在执行的时候解压。
  
  目前默认的是第1种方案，因为样做的好处是分担sql线程的压力，减少relaylog的积压。
  但是如果set global relay_log_uncompress=OFF,则启用方案2，在sql线程解压，为什么需要方案2呢？

  如果io线程拉取后就解压，可能导致slave磁盘空间预估不足。
  例如master压缩的binlog是100G，重建热备时就会认为磁盘剩余100G就够了，其实解压了可能有几百G。

## 如何使用
`set global log_bin_compress=ON `

表示开启Binlog压缩功能


`set global log_bin_compress_min_len=256 `

在开启压缩功能的前提下，binlog压缩字段长度大于256的event，符合压缩条件。


`relay_log_uncompress = ON`

表示在IO线程中解压，解压后写入relaylog。

`relay_log_uncompress = OFF`

拉取binlog以后直接写入relaylog，然后在执行的时候解压。