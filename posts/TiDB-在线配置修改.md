+++
title = "TiDB 在线配置修改"
date = "2024-03-11"
tags = ["TiDB"]
draft = true
+++


```sql
show config;
show config where type='tidb'
show config where instance in (...)
show config where name like '%log%'
show config where type='tikv' and name='log.level'
set config "127.0.0.1:20180" `split.qps-threshold`=1000
```



## 参考文档

- https://docs.pingcap.com/zh/tidb/v7.6/dynamic-config
