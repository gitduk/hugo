---
title: TiDB-Lightning 错误排查
date: 2023-11-17 10:10:01
tags:
- TiDB
- Lightning
---



## raft entry is too large



修改 TiDB 配置 

```bash
tiup cluster edit-config tidb
```

```conf
  tikv:
    raftstore.raft-entry-max-siz: 128MB
```

重启 tikv 节点

```bash
tiup cluster reload tidb-cluster -R tikv
```

