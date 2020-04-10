
#Big data scenarios
The ability of transparent sharding and dynamic scalability makes TenDB Cluster ideal for big data scenarios, especially for below:
Huge amount of data (50T+)
Number of reads overwhelms the number of writes
Scattered reads takes a large portion among the overall reads
Complexed calculations take a small portion
Data may need to be rotated periodically
For TenDB Clusters with a large amount of data, It's important to avoid excessive calculation load on TSpider layer and to avoid heavy IO load on TenDB storage layer. Best practices will be advised in this manual for clusters with large data amounts.

### **Sharding number**
All data in old shards must be re-organized to change the number of shards, which could make a heavy workload to the system. So it's crucial to choose a proper sharding number. Usually it's suggested to shard your data with 1.5 times to 2 times of your expected data amount. To make it easier to scale up or scale down the cluster, It's suggested to use a sharding number with more submultiples, such as 48, 64, or 72.

### **Table design and workload**
In a big data cluster, below tips are advised:
By default, the first column of primary key/unique key will be used as a shard key, if no primary key nor unique key is defined, you must designate a shard key explicitly.
When designating a shard key, it's suggested that the distribution of mod(hash(shard key)) should be even. Otherwise a skewed data distribution may lead to hot spots or bottleneck on part of the shards.
Use merged write when you write your data.
Use shard key in your query if possible.
Avoid retrieving large amounts of data directly from Tspider node, which may lead to an OOM issue.
Avoid requests with a heavy calculation workload.
Avoid join query that retrieves large amounts of data.

### **Manage storage instance**
In a big data cluster, bottlenecks usually lie on the storage layer. Below practices are suggested:
Compress your data. Big data compress feature of TenDB is suggested. Or you can use TokuDB or other engines with high compress ratio.
Use physical backup to backup your data.
If old data need to be deleted periodically, It's suggested to use partitioned tables in your storage instance.
When dropping a partition, you can use the hard link trick to avoid influence on your IO performance.
For non-SSD disks, queries that generate scatted IO should be avoided if possible, such as (二级索引反查) or write to a heap table.
