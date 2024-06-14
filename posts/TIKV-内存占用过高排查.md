+++
title = "TIKV 内存占用过高排查"
date = "2023-07-05"
tags = ["TiDB"]

+++



tidb 集群（tidb-cluster）的 tikv 节点，占用内存高达 30/50 G，内存不够用。




## 排查

登录 grafana 监控面板，发现 tidb-cluster-TiKV-Details 下面的 Block cache size 面板疑似有点问题。

![image-20230705173007288](/img/image-20230705173007288.png)



遂去查看 tidb-cluster 集群的配置文件

```bash
tiup cluster edit-config tidb-cluster
```

问题在于 tikv 的 `storage.block-cache.capacity` 配置：

```yaml
storage.block-cache.capacity: 50GiB
```



## 解决

把 tikv 的 `storage.block-cache.capacity` 配置修改成合适的值，保存配置，重启 tidb 集群即可。
