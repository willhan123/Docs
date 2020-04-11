#General guidelines
Due to the ability to be compatible with MySQL protocol, making transparent partitions on both database or table level, and online scale up and scale down, TenDB cluster is suitable for most scenarios where MySQL is applicable whereas a shared layout is more adequate to handle large amount of data or requests.

This document summarizes best practices on common tasks when using TenDB cluster, including table creation, SQL query, and instance management.

## **Create database or table**
Below tips are advised when creating database or table on TenDB cluster:
Use the same character set for columns to the table when creating a table on TSpider.
When creating a table on TSpider, by default the first column of primary key/unique key will be used as shard_key; if no primary key nor unique key is present, a shard key should be designated explicitly.
When choosing a shard_key, data should be distributed evenly upon mod(harsh(shard_key)), otherwise, a skewed data layer could lead to hot spots or bottleneck.
Foreign key constraint is not applicable for TSpider tables.
When using auto-increment columns, use bigint as the data type for an auto_increment column, and it's suggested to let TSpider itself to manage the columns. It's important to be aware that TSpider only guarantees uniqueness of auto-increment columns, and no guarantee that they are ordered and increment globally.
If primary key and unique key are present, if the primary key is of auto_increment type, and the unique key is the character column of application business logic, it's suggested to modify auto_increment column to ordinary index.
Enabling compression is suggested for Blob columns.
If data are to be expired and deleted periodically, partitioned tables are suggested to be used in TenDB storage instance.
It's suggested to store hot and cold data separately when designing a table.
It's suggested to store log data and data subject to change in separate storage, Such as different databases or clusters.

## **SQL requests**
Below tips are advised when make requests to TenDB cluster:
Use shard key on the right of == in your query, If not possible, it's suggested to create an index, and to throttle frequency of requests, massive scans that across partitions should be avoided if possible.
Avoid batch deletion.
Avoid retrieving large amounts of data from TSpider node, which may lead to memory usage surge in TSpider node or even an OOM failure.
Avoid aggregation queries(such as group by, etc.) of high frequency.
Use straight_join to designate driving table when using join.
Avoid to retrieve data in order by x limit m, n style.
If data are to be retrieved frequently, such as configuration data, It's suggested to use a cache in the front layer to reduce the workload on database.
If business logic makes random limit n requests, SPIDER_RONE_SHARD switch could be turned on, when enabled, requests like select SPIDER_RONE_SHARD * from t limit n will be routered randomly to one of the backend shards.
## **TSpider instance management**
Below tips are advised when manage TSpider instances:
Setting common parameters, see the next section: Setting parameters for TSpider
When retrieving a large amount of data, or even the entire database, It's suggested to use temporary TSpider nodes to serve the request.

## **Setting parameters for TSpider **
### spider_quick_mode
> This parameter specifies where to cache data retrieved from backends, RemoteDB or local buffer.
> The default value is 1 upon installation. If a database merge task is expected, it's suggested to be set to 0.
> In rare situations, It's set to 3, to cope with OOM killed issue when retrieving a large amount of data. The side effect is temporary storage space of spider may be exhausted when retrieving mass data.

### spider_max_connections
> This parameter is used to specify the max number of connections from TSpider to RemoteDB. According to our experience in production environments, the value of 200 is adequate for most applications. If the value is set to high, too many connections may be made to RemoteDB. 
And spider_max_connections * <TSpider count> should be less than the max_connections set on TenDB storage node.

### spider_bgs_mode

> This parameter specifies whether to enable parallel query in backends. It's suggested to be set to 1, ie., to enable parallel query in backends.

### spider_index_hint_pushdown
> This parameter specifies whether to enable index hint to be pushed to backends, such as force_index. It's suggested to be set to 1.

### spider_get_sts_or_crd
> This parameter specifies whether to allow statements such as show table status, and show index, etc. to be executed on RemoteDB to collect statistics. It's suggested to be set to 1.

### spider_ignore_autocommit
> This parameter specifies whether to allow the set autocommit=0 statement can be executed on RemoteDB.

### spider_rone_shard_switch
> TSpider introduced a new SPIDER_RONE_SHARD option, when enabled, statements such as select SPIDER_RONE_SHARD * from t limit 10 will be distributed to a random backend shard. In scenarios where only a small sample piece of the data is desired, It could be enabled.

For more suggestions on TSpider parameter setting, see [TSpider parameters](./../re-book/tspider-parameter-en.md)

## **存储实力管理**
Below tips are advised when manage TenDB storage instance:
It's suggested to use SSD/NVME disk to avoid I/O bottleneck.
It's suggested to use Binlog of ROW mode.
It's suggested to use a physical backup manner to back your data. It's suggested to switch upload Binlog to remote backup system, in every 5 minutes.
When dropping a partition, use the hard link trick to avoid influence on I/O performance.

## **Sharding number**
All data in old shards must be re-organized to change the number of shards, which could make a heavy workload to the system. So it's crucial to choose a proper sharding number. Usually it's suggested to shard your data with 1.5 times to 2 times of your expected data amount. To make it easier to scale up or scale down the cluster, It's suggested to use a sharding number with more submultiples, such as 12, 26 or 24.

