---
title: ES 数据迁移
date: 2023-08-24 10:36:43
tags:
- ES
---


> 老板有一个需求，把旧索引里面的数据全部迁移到新索引，ES 内置的 reindex 可以快速进行数据迁移。



## 迁移方案

由于数据量太大，一次性 reindex 可能会出问题。

于是写了一个脚本分批迁移。



## 分批 reindex 脚本

```bash
SOURCE_INDEX="old index"
TARGET_INDEX="new index"
START=$1
END=$2
BATCH_SIZE=10000

# Elasticsearch 集群的 SSL 配置
# CERT_FILE="/home/[USER]/elasticsearch/config/certs/http_ca.crt"

# 执行 reindex 操作
for ((i = $START; i < $END; i++)); do
  from=$((i * BATCH_SIZE))
  to=$((from + BATCH_SIZE))

  printf "Processing batch $((i + 1)) of $END ...\n"

  # 发起 reindex 请求
  curl -s -X POST "https://127.0.0.1:9200/_reindex?refresh=false" -H 'Content-Type: application/json' --cacert $CERT_FILE -u "elastic:[PASSWORD]" -d "
  {
    \"source\": {
      \"index\": \"$SOURCE_INDEX\",
      \"size\": $BATCH_SIZE,
      \"query\": {
        \"range\": {
          \"id\": {
            \"gte\": $from,
            \"lt\": $to
          }
        }
      }
    },
    \"dest\": {
      \"index\": \"$TARGET_INDEX\"
    }
  }"
  echo
done

echo "Reindex completed."
```

