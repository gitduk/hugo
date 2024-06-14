+++
title = 'ES 索引备份'
date = 2024-03-12T16:40:30+08:00
draft = false
tags = [ "ES", "elasticsearch-dump" ]

+++


## 前情提要

服务器空间不够，要买几个硬盘重组 raid0，需要把 ES 数据提前导出。

工具：[elasticsearch-dump](https://github.com/elasticsearch-dump/elasticsearch-dump)


## 安装

```bash
npm install elasticdump -g
```

## 导出

假设有 index1-6 6 个索引需要导出，写了一个脚本一次性导出全部索引，并记录导出的 log。

```bash
#!/usr/bin/env zsh

indexs=(
  index_1
  index_2
  index_3
  index_4
  index_5
  index_6
)

for index in "${indexs[@]}"; do
	echo "dumping $index ..." | tee -a dump.log

	elasticdump \
		--input=http://elastic:passwd@127.0.0.1:9200/${index} \
		--output=/home/you/es/${index}_mapping.json \
		--type=mapping

	elasticdump \
		--input=http://elastic:passwd@127.0.0.1:9200/${index} \
		--output=/home/you/es/${index}.json \
		--type=data \
		--limit=10000 \
		--maxSockets=20 \
		--noRefresh | tee -a ${index}.log

	[[ $? -eq 0 ]] && echo "dump $index success" | tee -a dump.log || echo "dump $index failed" | tee -a dump.log

done
```

elasticdump 参数解释：

- `--limit=10000` 限制每批次导出的文档数量为 10000
- `--maxSockets=20` 设置最大并发连接数为 20
- `--noRefresh` 禁止在导出过程中执行 Elasticsearch 索引刷新操作



**这里有一个大坑，导出 mapping 并不会导出索引的设置，建议手动备份建索引语句，恢复备份时手动创建索引。**



## 导入

部署了一个新 ES 索引并创建好索引后，导入备份文件。

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0 elasticdump \
  --input=./your_index.json \
  --output=https://elastic:password@127.0.0.1:9200/your_index \
  --type=data \
  --limit=10000 \
  --maxSockets=50 \
  --noRefresh
```



`NODE_TLS_REJECT_UNAUTHORIZED=0` - 防止出现 `Error Emitted => self-signed certificate in certificate chain` 错误。

