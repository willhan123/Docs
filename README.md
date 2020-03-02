# TenDB Cluster
TenDB Cluster是腾讯DBA团队提供的MySQL分布式关系型数据库解决方案，主要包括三个核心组件：TSpider，TenDB 、Tdbctl。   
TSpider是TenDB Cluster的接入层，能够水平扩展；TSpider [github地址]()  
TenDB是TenDB Cluster的存储层，是一个定制的MySQL分支； TenDB [github地址]()  
Tdbctl是集群的中控节点，提供集群路由管理、集群变更、集群监控等能力。Tdbctl [github地址]()  
TenDB Cluster Operator则提供在主流云基础设施（Kubernetes）上部署管理TenDB Cluster集群的能力。TenDB Cluster Operator [github地址]()  

## TenDB Cluster简介
TenDB Cluster是腾讯游戏DBA团队提供的MySQL分布式关系型数据库解决方案，主要特点包括：透明分库分表、高可用的MySQL集群服务，透明及在线的扩容及缩容；使得开发者可以仅专注于业务逻辑的开发及运营，无需编写数据分片逻辑，在海量用户并发情况下，也无须关心DB存储层的负载压力。
## 快速体验
> docker-compose使用方式
## 使用文档
具体见[参考手册](SUMMARY.md)

## 联系我们
> 邮箱  
> issue地址