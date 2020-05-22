本文档基于社区Kube-Prometheus，具体参考(https://github.com/coreos/kube-prometheus)。因为部署的环境稍有不同，因此做了相应改动，本文档主要讲解改动部分。

# jasonnet配置
kube-prometheus编译所需全部用jasonnet配置，社区工程中jsonnet/kube-prometheus下有相应文件参考。
本文添加了thanos支持：
``` 
(import 'kube-prometheus/kube-prometheus-thanos-sidecar.libsonnet')
    ……
    thanos+:: {
        objectStorageConfig: {
          key: 'thanos.yaml',  // How the file inside the secret is called
          name: 'thanos-objectstorage',  // This is the name of your Kubernetes secret with the config
        },
    }
```

# 编译
```
make
```
编译完成后在manifest目录下得到所需的yaml文件

# 修改yaml文件
## prometheus-clusterRole.yaml 
增加权限
```
resources:
  - nodes/metrics
  - pods
  verbs:
  - get
  - list
  - watch
- nonResourceURLs:
  - /metrics
  - /metrics-hr
  - /metrics-mr
  verbs:
  - get
```
## node-exporter-daemonset.yaml 
node-exporter 不监控磁盘
```
- args:
        - --web.listen-address=127.0.0.1:9100
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
#        - --path.rootfs=/host/root
        - --log.level=debug
        - --collector.filesystem.ignored-mount-points=.*
        - --collector.filesystem.ignored-fs-types=.*
#        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+|data/docker/.+|data/kubernetes-volume/.+|host/root/data/kubelet/.+|data/kubelet/.+)($|/)
#        - --collector.filesystem.ignored-fs-types=^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs|tmpfs|vfat|fuse.lxcfs|rootfs|lxcfs)$

#        - mountPath: /host/root
#          mountPropagation: HostToContainer
#          name: root
#          readOnly: true
……
#      - hostPath:
#          path: /
#        name: root
```
## 导入manifests下的yaml文件
```
kubectl create -f manifests/setup
kubectl create -f manifests/
```

# 编辑thanos.yaml，创建secret
```
kubectl create secret generic thanos-objectstorage -n $namespace --from-file=./thanos/thanos.yaml
```

# 创建PodMonitor
PodMonitor根据pod label匹配对应Pod，获取数据，同时导入pod性能数据采集规则。
```
kubectl create -f dbmonitor/tendb-podmonitor.yaml
kubectl create -f dbmonitor/os-rule.yaml
```
自此完成kube-prometheus的部署，可以通过Prometheus的NodeIP访问Prometheus和Grafana
* Prometheus：$nodeip:30900
* Grafana：$nodeip:30902
* AlertManager: $nodeip:30903

# Grafana 视图
Grafana使用jsonfile导入对应DB的Grafana模板，构建视图。

# Trouble Shooting
在部署kube-Prometheus可能会遇到以下问题：
1. Prometheus不能匹配到node
部署时报错：
```
Warning  FailedScheduling  106s (x33 over 42m)  default-scheduler 0/70 nodes are available: 70 node(s) didn't match node selector.
```
kube-prometheus有node selector匹配node，需要标签为“kubernetes.io/os: linux”的节点，如果不能匹配，则需要修改manifests下的yaml文件，增加或者修改node selector。

2. prometheus-Operator不断重启，CrashLoopBackOff状态
PodMonitor缺少port或者targetPort说明
```
podMetricsEndpoints:
  - interval: 20s
    #port: mysql-port
    scheme: http
    targetPort: "annotation_prometheus.io" # can by number or string
```

3. PodMonitor不能匹配到DB集群的pod
prometheus依赖annotation匹配对应的pod，从而获取性能数据，如下可以给对应的pod打上annotation
```
kubectl annotate pods $podname "prometheus.io/scrape"=true -n $namespace
kubectl annotate pods $podname "prometheus.io/port"=9121 -n $namespace
```


