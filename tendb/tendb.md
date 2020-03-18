# TenDB简介
TenDB作为TenDB Cluster集群的存储节点，承担着数据存储的重要作用。  
TenDB是基于社区Percona版本高度定制的单据MySQL，目前支持MySQL 5.5，5.6，5.7，在完全兼容源生MySQL的基础上，开发集成了一些实用的功能特性，无论是集群应用还是单机使用，TenDB都是不错的选择。  
相比Percona已有的特性，TenDB支持binlog压缩，binlog限速，在线加字段，blob压缩，XA事务优化等，这些特性很多都是应用在实际使用时会遇到的痛点，下文将会着重介绍这些重要的特性。