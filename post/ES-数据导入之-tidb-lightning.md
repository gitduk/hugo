---
title: ES 数据导入之 tidb lightning
date: 2023-07-17 11:28:03
tags:
- ES
---



> [TiDB Lightning](https://docs.pingcap.com/zh/tidb/stable/tidb-lightning-overview) 是用于从静态文件导入 TB 级数据到 TiDB 集群的工具，常用于 TiDB 集群的初始化数据导入。



## 前情提要

突然发现初始化 TIDB 数据库的时候漏掉了一个表，还好有备份，现在需要从静态备份里面导入漏掉的那个表。



## 编辑 tidb-lightning 配置文件

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
sorted-kv-dir = "/tmp/sorted-kv-dir"

[mydumper]
# 源数据目录。
data-source-dir = "/path/to/data/exported/"

# 配置通配符规则，默认规则会过滤 mysql、sys、INFORMATION_SCHEMA、PERFORMANCE_SCHEMA、METRICS_SCHEMA、INSPECTION_SCHEMA 系统数据库下的所有表
# 若不配置该项，导入系统表时会出现“找不到 schema”的异常
filter = ['*.*', '!mysql.*', '!sys.*', '!INFORMATION_SCHEMA.*', '!PERFORMANCE_SCHEMA.*', '!METRICS_SCHEMA.*', '!INSPECTION_SCHEMA.*']

[mydumper.csv]
# 字段分隔符，支持一个或多个字符，默认值为 ','。如果数据中可能有逗号，建议源文件导出时分隔符使用非常见组合字符例如'|+|'。
separator = '|+|'
# 引用定界符，设置为空表示字符串未加引号。
delimiter = ''
# 行尾定界字符，支持一个或多个字符。设置为空（默认值）表示 "\n"（换行）和 "\r\n" （回车+换行），均表示行尾。
terminator = ""
# CSV 文件是否包含表头。
# 如果为 true，首行将会被跳过，且基于首行映射目标表的列。
header = true
# CSV 是否包含 NULL。
# 如果为 true，CSV 文件的任何列都不能解析为 NULL。
not-null = true
# 如果 `not-null` 为 false（即 CSV 可以包含 NULL），
# 为以下值的字段将会被解析为 NULL。
null = '\N'
# 是否解析字段内的反斜线转义符。
backslash-escape = true
# 是否移除以分隔符结束的行。
trim-last-separator = false

[tidb]
# 目标集群的信息
host = "127.0.0.1"
port = 4011
user = "root"
password = "[your password]"
# 表架构信息在从 TiDB 的“状态端口”获取。
status-port = 15080
# 集群 pd 的地址
pd-addr = "127.0.0.1:2079"
```



## 启动 tidb-lightning

```bash
tiup tidb-lightning -config tidb-lightning.toml
```



导入成功你会看到下面的输出：
![image-20230717113854601](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230717113854601.png)



