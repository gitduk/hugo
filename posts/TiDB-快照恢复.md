+++
title = "TiDB 快照恢复"
date = "2023-07-05"
tags = ["TiDB"]

+++



TiDB 使用周期性运行的 GC（Garbage Collection，垃圾回收）来进行清理，默认情况下每 10 分钟一次。每次 GC 时，TiDB 会计算一个称为 safe point 的时间戳（默认为上次运行 GC 的时间减去 10 分钟），接下来 TiDB 会在保证在 safe point 之后的快照都能够读取到正确数据的前提下，删除更早的过期数据。




## 前情提要

公司小萌新误操作，把数据表清空。



## 修改 GC 的生存时间

tidb 默认 GC 生存时间比较短，一旦删库的时间点超过这个时间就直接完犊子。

登录 tidb 数据库，查看 GC 生存时间：

```sql
select * from mysql.tidb where variable_name = 'tikv_gc_life_time';
```

![image-20230705175301357](/img/image-20230705175301357.png)

60 分钟，我们有 1.5 亿条数据，不太够。

设置 GC 生存时间为 72 小时：

```sql
update mysql.tidb set VARIABLE_VALUE="72h" where VARIABLE_NAME="tikv_gc_life_time";
```



## 设置快照时间点

设置 tidb_snapshot 变量为删库前的某个时间点：

```sql
set @@tidb_snapshot="2023-05-30 13:10:00";
```

设置完成之后，应该就能看到被删除的数据了，接下来需要导出导入数据。



## 使用 tiup dumpling 导出被删除的数据

```bash
tiup dumpling -u root -P [port] -p [password] -h 127.0.0.1 -o /path/to/export/backup -r 200000 --filter "[DataBase].[Table]" --snapshot "2023-05-30 13:10:00"
```

数据完全导出后，可恢复 tidb_snapshot 为 `""`，并删除空表。

```sql
set @@tidb_snapshot="";
drop table [table];
```



## 使用 tidb-lightning 导入数据

安装 tidb-lightning

```bash
tiup install tidb-lightning
```



创建 tidb-lightning 配置文件 <u>tidb-lightning.toml</u> ：

```toml
[lightning]
# 日志
level = "info"
file = "log/tidb-lightning.log"

[tikv-importer]
# 选择使用的导入模式
backend = "tidb"
# 设置排序的键值对的临时存放地址，目标路径需要是一个空目录
sorted-kv-dir = "/tmp/sorted-kv-dir"

[mydumper]
# 源数据目录。
data-source-dir = "/path/to/export/backup"

# 配置通配符规则，默认规则会过滤 mysql、sys、INFORMATION_SCHEMA、PERFORMANCE_SCHEMA、METRICS_SCHEMA、INSPECTION_SCHEMA 系统数据库下的所有表
# 若不配置该项，导入系统表时会出现“找不到 schema”的异常
filter = ['*.*', '!mysql.*', '!sys.*', '!INFORMATION_SCHEMA.*', '!PERFORMANCE_SCHEMA.*', '!METRICS_SCHEMA.*', '!INSPECTION_SCHEMA.*']

[tidb]
# 目标集群的信息
host = "127.0.0.1"
port = [port]
user = "root"
password = "xxxxx"
# 表架构信息在从 TiDB 的 "状态端口" 获取。
status-port = 15080
# 集群 pd 的地址，可用 tiup cluster display [your cluster] 查看
pd-addr = "127.0.0.1:2879"
```

导入数据到 TIDB 数据库：

```bash
tiup tidb-lightning -config tidb-lightning.toml
```

