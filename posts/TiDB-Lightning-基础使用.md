+++
title = "TiDB-Lightning 基础使用"
date = "2023-11-23"
tags = ["TiDB", "Lightning"]

+++



前提是已经用 [Dumpling](https://docs.pingcap.com/tidb/stable/dumpling-overview) 将数据从上游数据库导出。




## 创建配置文件 tidb-lightning.toml

<u>tidb-lightning.toml</u>

```toml
[lightning]
# 日志
level = "info"
file = "log/tidb-lightning.log"

[tikv-importer]
# 选择使用的导入模式
backend = "tidb"
# 设置排序的键值对的临时存放地址，目标路径需要是一个空目录
sorted-kv-dir = "/tmp/data-sorted-kv-dir"
# 忽略重复数据
on-duplicate = "ignore"

[mydumper]
# 源数据目录。
data-source-dir = "/path/to/exported/data"

# 配置通配符规则，默认规则会过滤 mysql、sys、INFORMATION_SCHEMA、PERFORMANCE_SCHEMA、METRICS_SCHEMA、INSPECTION_SCHEMA 系统数据库下的所有表
# 若不配置该项，导入系统表时会出现“找不到 schema”的异常
filter = ['*.*', '!mysql.*', '!sys.*', '!INFORMATION_SCHEMA.*', '!PERFORMANCE_SCHEMA.*', '!METRICS_SCHEMA.*', '!INSPECTION_SCHEMA.*']

[tidb]
# 目标集群的信息
host = "127.0.0.1"
port = 4000
user = "root"
password = "passwd"
# 表架构信息在从 TiDB 的 "状态端口" 获取。
status-port = 15080
# 集群 pd 的地址，可用 tiup cluster display [your cluster] 查看
pd-addr = "127.0.0.1:2379"
```



## 导入数据

```bash
tiup tidb-lightning  -config tidb-lightning.toml
```



## 参考链接

- https://docs.pingcap.com/tidb/stable/tidb-lightning-configuration

