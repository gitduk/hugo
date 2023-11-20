---
title: TiDB 错误排查
date: 2023-10-23 14:27:51
tags:
- TiDB
---



## entry too large, the max entry size is 6291456, the size of data is 7254170

查看 `performance.txn-entry-size-limit` 大小：

```mysql
show config where type='tidb' and name like '%txn-entry-size-limit';
```

```output
+------+--------------------+----------------------------------+---------+
| Type | Instance           | Name                             | Value   |
+------+--------------------+----------------------------------+---------+
| tidb | 192.168.70.46:4000 | performance.txn-entry-size-limit | 6291456 |
+------+--------------------+----------------------------------+---------+
1 row in set (0.01 sec)
```

修改 `performance.txn-entry-size-limit` 大小：

```bash
tiup cluster edit-config <cluster-name>
```

```yaml
server_configs:
  tidb:
    txn-entry-size-limit: 62914560
```

重载 tidb 配置：

```bash
tiup cluster reload <cluster-name> -R tidb
```

检查修改是否生效：

```mysql
show config where type='tidb' and name like '%txn-entry-size-limit';
```



## raft entry is too large, region 157021, entry size 10309778

查看 `raft-entry-max-size` 大小:

```mysql
show config where type='tikv' and name like '%entry-max-size';
```

```
+------+---------------------+-------------------------------+-------+
| Type | Instance            | Name                          | Value |
+------+---------------------+-------------------------------+-------+
| tikv | 192.168.xx.xx:xxxxx | raftstore.raft-entry-max-size | 8MiB  |
+------+---------------------+-------------------------------+-------+
```

修改 `raft-entry-max-size` 大小:

```bash
tiup cluster edit-config <cluster-name>
```

```yml
server_configs:
  tikv:
    raftstore.raft-entry-max-size: 20MB
```

重载 tikv 配置:

```bash
tiup cluster reload <cluster-name> -R tikv
```

检查修改是否生效：

```mysql
show config where type='tikv' and name like '%entry-max-size';
```

