集群的中控节点是一个MySQL分支，在兼容MySQL的参数的基础上新增了一些参数。
其主要参数以及配置可以参照mysql官方手册，本文将主要介绍新增的参数。





`TC_ADMIN`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: FALSE   

VARIABLE_COMMENT: If set to TRUE, as config center to process query

使用这个参数后，中控节点才会进行转发，不然就是个普通mysql
```




`TC_FORCE_EXECUTE`
```
VARIABLE_SCOPE: SESSION|GLOBAL

DEFAULT_VALUE: ON

VARIABLE_COMMENT: If set to TRUE, go on running spider query if remote failed

中控节点分发时， 如果spider执行出错，是否继续分发到remote上去
```



`TC_CHECK_REPAIR_ROUTING`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: ON

VARIABLE_COMMENT: If set to TRUE, check and repair routing between tdbctl and spiders
默认开启，表示定期会监查和修复中控节点和spider的路由信息
```





`TC_CHECK_REPAIR_ROUTING_INTERVAL`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: 300

VARIABLE_COMMENT: The interval time of checking and reparing routing betwwen tdbctl and spiders
监查和修复中控节点和spider的路由信息的时间间隔，单位为秒
```

`TC_SPIDER_WRAPPER_PREFIX`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: SPIDER

VARIABLE_COMMENT: prefix of server name for SPIDER wrapper

spider的名称前缀
```


`TC_MYSQL_WRAPPER_PREFIX`
```
VARIABLE_SCOPE: GLOBAL

DEFAULT_VALUE: SPT

VARIABLE_COMMENT: prefix of server name for MYSQL wrapper

后端mysql的名称前缀
```

