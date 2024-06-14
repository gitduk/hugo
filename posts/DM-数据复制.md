+++
title = "DM 数据复制"
date = "2023-07-10"
tags = [ "TiDB", "DM"]
categories = ["DB"]
+++

[TiDB Data Migration](https://docs.pingcap.com/zh/tidb/stable/dm-overview) (DM) 是一款便捷的数据迁移工具，支持从与 MySQL 协议兼容的数据库（MySQL、MariaDB、Aurora MySQL）到 TiDB 的全量数据迁移和增量数据同步。使用 DM 工具有利于简化数据迁移过程，降低数据迁移运维成本。


## 前情提要

新建立了一个 tidb 多节点集群，现在需要从 tidb 单节点集群迁移数据到新集群。

## 部署 DM 集群

安装 dmctl

```bash
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
tiup install dm dmctl
```

生成 DM 集群最小拓扑文件

```bash
tiup dm template
```

复制输出的配置信息，修改 IP 地址后保存为 <u>topology.yaml</u> 文件，使用 TiUP 部署 DM 集群

```bash
tiup dm deploy dm-test 6.0.0 topology.yaml -p
```

## 准备数据源

创建数据源文件 <u>mysql-01.yaml</u>

```yaml
# 数据源 ID，在数据迁移任务配置和 dmctl 命令行中引用该 source-id 可以关联到对应的数据源
source-id: "mysql-01"

from:
  host: "127.0.0.1"
  port: 3306
  user: "root"
  password: "fCxfQ9XKCezSzuCD0Wf5dUD+LsKegSg=" 	 # 使用 tiup dmctl --encrypt "123456" 加密。
```

使用如下命令将数据源增加至 DM 集群

```bash
# tiup dm display dm-main -R dm-master 命令可查看 --master-addr 地址
tiup dmctl --master-addr http://127.0.0.1:8761 operate-source create mysql-01.yaml
```



## 创建数据同步任务

创建任务的配置文件 <u>mysql-to-tidb.yaml</u>

```bash
## ********* 任务信息配置 *********
name: mysql-to-tidb
shard-mode: ""                       # 默认值为 "" 即无需协调。如果为分库分表合并任务，请设置为悲观协调模式 "pessimistic"。在深入了解乐观协调模式的原理和使用限制后，也可以设置为乐观协调模式 "optimistic"
task-mode: "full"                    # 任务模式，可设为 "full" - "只进行全量数据迁移"、"incremental" - "Binlog 实时同步"、"all" - "全量 + Binlog 迁移"
# timezone: "UTC"                    # 指定数据迁移任务时 SQL Session 使用的时区。DM 默认使用目标库的全局时区配置进行数据迁移，并且自动确保同步数据的正确性。使用自定义时区依然可以确保整个流程的正确性，但一般不需要手动指定。

## ******** 数据源配置 **********
mysql-instances:
  - source-id: "mysql-01"           # 从数据源 mysql-01 迁移数据
    block-allow-list:  "bw-rule-1"
    filter-rules: ["filter-rule-1"]
    route-rules: ["route-rule-1"]

    mydumper-config-name: "global"   # mydumpers 配置的名称
    loader-config-name: "global"     # loaders 配置的名称
    syncer-config-name: "global"     # syncers 配置的名称

## ******** 目标 TiDB 配置 **********
target-database:       # 目标 TiDB 配置
  host: "127.0.0.1"
  port: 3306
  user: "root"
  # 如果密码不为空，则推荐使用经过 dmctl 加密的密文
  password: "fCxfQ9XKCezSzuCD0Wf5dUD+LsKegSg=" 

## ******** 功能配置 **********
mydumpers:                           # dump 处理单元的运行配置参数
  global:                            # 配置名称
    threads: 4                       # dump 处理单元从上游数据库实例导出数据和 check-task 访问上游的线程数量，默认值为 4
    chunk-filesize: 64               # dump 处理单元生成的数据文件大小，默认值为 64，单位为 MB
    extra-args: "--consistency none" # dump 处理单元的其他参数，不需要在 extra-args 中配置 table-list，DM 会自动生成

loaders:                             # load 处理单元的运行配置参数
  global:                            # 配置名称
    pool-size: 16                    # load 处理单元并发执行 dump 处理单元的 SQL 文件的线程数量，默认值为 16，当有多个实例同时向 TiDB 迁移数据时可根据负载情况适当调小该值

    # 保存上游全量导出数据的目录。该配置项的默认值为 "./dumped_data"。
    # 支持配置为本地文件系统路径，也支持配置为 Amazon S3 路径，如: s3://dm_bucket/dumped_data?endpoint=s3-website.us-east-2.amazonaws.com&access_key=s3accesskey&secret_access_key=s3secretkey&force_path_style=true
    dir: "./dumped_data"

    # 全量阶段数据导入的模式。可以设置为如下几种模式：
    # - "sql"(默认)。使用 [TiDB Lightning](/tidb-lightning/tidb-lightning-overview.md) TiDB-backend 进行导入。
    # - "loader"。使用 Loader 导入。此模式仅作为兼容模式保留，目前用于支持 TiDB Lightning 尚未包含的功能，预计会在后续的版本废弃。
    import-mode: "sql"
    # 全量导入阶段针对冲突数据的解决方式：
    # - "replace"（默认值）。仅支持 import-mode 为 "sql"，表示用最新数据替代已有数据。
    # - "ignore"。仅支持 import-mode 为 "sql"，保留已有数据，忽略新数据。
    # - "error"。仅支持 import-mode 为 "loader"。插入重复数据时报错并停止同步任务。
    on-duplicate: "replace"

syncers:                             # sync 处理单元的运行配置参数
  global:                            # 配置名称
    worker-count: 16                 # 应用已传输到本地的 binlog 的并发线程数量，默认值为 16。调整此参数不会影响上游拉取日志的并发，但会对下游产生显著压力。
    batch: 100                       # sync 迁移到下游数据库的一个事务批次 SQL 语句数，默认值为 100，建议一般不超过 500。
    enable-ansi-quotes: true         # 若 `session` 中设置 `sql-mode: "ANSI_QUOTES"`，则需开启此项

    # 设置为 true，则将来自上游的 `INSERT` 改写为 `REPLACE`，将 `UPDATE` 改写为 `DELETE` 与 `REPLACE`，保证在表结构中存在主键或唯一索引的条件下迁移数据时可以重复导入 DML。
    safe-mode: false
    # 自动安全模式的持续时间
    # 如不设置或者设置为 ""，则默认为 `checkpoint-flush-interval`（默认为 30s）的两倍，即 60s。
    # 如设置为 "0s"，则在 DM 自动进入安全模式的时候报错。
    # 如设置为正常值，例如 "1m30s"，则在该任务异常暂停、记录 `safemode_exit_point` 失败、或是 DM 进程异常退出时，把安全模式持续时间调整为 1 分 30 秒。详情可见[自动开启安全模式](https://docs.pingcap.com/zh/tidb/stable/dm-safe-mode#自动开启)。
    safe-mode-duration: "60s"
    # 设置为 true，DM 会在不增加延迟的情况下，尽可能地将上游对同一条数据的多次操作压缩成一次操作。
    # 如 INSERT INTO tb(a,b) VALUES(1,1); UPDATE tb SET b=11 WHERE a=1; 会被压缩成 INSERT INTO tb(a,b) VALUES(1,11); 其中 a 为主键
    # 如 UPDATE tb SET b=1 WHERE a=1; UPDATE tb(a,b) SET b=2 WHERE a=1; 会被压缩成 UPDATE tb(a,b) SET b=2 WHERE a=1; 其中 a 为主键
    # 如 DELETE FROM tb WHERE a=1; INSERT INTO tb(a,b) VALUES(1,1); 会被压缩成 REPLACE INTO tb(a,b) VALUES(1,1); 其中 a 为主键
    compact: false
    # 设置为 true，DM 会尽可能地将多条同类型的语句合并到一条语句中，生成一条带多行数据的 SQL 语句。
    # 如 INSERT INTO tb(a,b) VALUES(1,1); INSERT INTO tb(a,b) VALUES(2,2); 会变成 INSERT INTO tb(a,b) VALUES(1,1),(2,2);
    # 如 UPDATE tb SET b=11 WHERE a=1; UPDATE tb(a,b) set b=22 WHERE a=2; 会变成 INSERT INTO tb(a,b) VALUES(1,11),(2,22) ON DUPLICATE KEY UPDATE a=VALUES(a), b=VALUES(b); 其中 a 为主键
    # 如 DELETE FROM tb WHERE a=1; DELETE FROM tb WHERE a=2 会变成 DELETE FROM tb WHERE (a) IN (1),(2)；其中 a 为主键
    multiple-rows: false

block-allow-list:
  # 规则名称
  bw-rule-1:
    # 迁移哪些库，支持通配符 "*" 和 "?"，do-dbs 和 ignore-dbs 只需要配置一个，如果两者同时配置只有 do-dbs 会生效
    do-dbs: ["DATABASE"]         
    # ignore-dbs: ["mysql", "account"] # 忽略哪些库，支持通配符 "*" 和 "?"
    do-tables:                                  # 迁移哪些表，do-tables 和 ignore-tables 只需要配置一个，如果两者同时配置只有 do-tables 会生效
    - db-name: "DATABASE"
      tbl-name: "YOUR TABLE"

filters:                                        # 定义过滤数据源特定操作的规则，可以定义多个规则
  filter-rule-1:                                # 规则名称
    schema-pattern: "DATABASE"                     # 匹配数据源的库名，支持通配符 "*" 和 "?"
    table-pattern: "YOUR TABLE"                  # 匹配数据源的表名，支持通配符 "*" 和 "?"
    events: ["truncate table", "drop table"]    # 匹配上 schema-pattern 和 table-pattern 的库或者表的操作类型
    action: Ignore                              # 迁移（Do）还是忽略(Ignore)

routes:                                         # 定义数据源表迁移到目标 TiDB 表的路由规则，可以定义多个规则
  route-rule-1:                                 # 规则名称
    schema-pattern: "DATABASE"                     # 匹配数据源的库名，支持通配符 "*" 和 "?"
    table-pattern: "YOUR TABLE"                  # 匹配数据源的表名，支持通配符 "*" 和 "?"
    target-schema: "DATABASE"                      # 目标 TiDB 库名
    target-table: "YOUR TABLE"                   # 目标 TiDB 表名
```

启动任务

```bash
tiup dmctl --master-addr http://127.0.0.1:8761 start-task mysql-to-tidb.yaml
```

查询任务

```bash
tiup dmctl --master-addr http://127.0.0.1:8761 query-status
```

