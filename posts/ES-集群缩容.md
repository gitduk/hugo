+++
title = "ES 集群缩容"
date = "2023-11-24"
tags = ["ES"]

+++



想通过增加 ES 节点达到提高 ES 吞吐量的目的，但是发现效果并不好。


Es 版本： 8.5.3

在 kibana 的 Dev Tools > Console 里运行：

```bash
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.exclude._name": "node-04"
  }
}
```

```
#! [xpack.monitoring.collection.enabled] setting was deprecated in Elasticsearch and will be removed in a future release.
{
  "acknowledged": true,
  "persistent": {},
  "transient": {
    "cluster": {
      "routing": {
        "allocation": {
          "exclude": {
            "_name": "node-04"
          }
        }
      }
    }
  }
}
```



在 Stack Monitoring > Elasticsearch overview  的 Shard Activity 面板下查看节点状态，等待分片迁移完毕即可安全移除 ES 节点。

