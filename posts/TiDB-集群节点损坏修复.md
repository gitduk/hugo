+++
title = "TiDB 集群节点损坏修复"
date = "2023-07-04"
tags = ["TiDB"]

+++



tidb 某个节点的数据目录被删除，导致节点损坏。




## 删除损坏节点

```bash
# 损坏节点 IP 为：192.168.70.51 端口为：27161
tiup cluster scale-in tidb-cluster --node 192.168.70.51:27161
```



## 添加新的节点

<u>scale-out.yaml</u> 文件内容：

```
tikv_servers:
  - host: 192.168.70.51
    port: 27161
    status_port: 27181
    numa_node: "1"
    config:
      server.labels: { host: "tikv3" }
      readpool.unified.max-thread-count: 25
      storage.block-cache.capacity: "30GiB"
      raftstore.capacity: "2048GiB"
```

添加节点：

```bash
# 集群名称为：tidb-cluster
tiup cluster scale-out tidb-cluster ./scale-out.yaml --user root -p
```



## 验证集群状态

```bash
tiup cluster display tidb-cluster
```

