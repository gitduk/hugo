+++
title = "logstash 拼接字段"
date = "2023-07-19"
tags = ["logstash"]

+++



ES 数据的 content 字段是 html 格式的内容，现在想用纯文本内容代替 html。




## 方案一

 用 ES reindex API 的 script 功能去除 html 标签。

```bash
#!/bin/bash

SOURCE_INDEX="[source index]"
TARGET_INDEX="[target index]"
START=$1
END=$2
BATCH_SIZE=10000

# Elasticsearch 集群的 SSL 配置
# CERT_FILE="/path/to/your/es/http_ca.crt"

# 执行 reindex 操作
for ((i = $START; i < $END; i++)); do
  from=$((i * BATCH_SIZE))
  to=$((from + BATCH_SIZE))

  printf "Processing batch $((i + 1)) of $END ...\n"

  # 发起 reindex 请求
  # curl -s -X POST "https://127.0.0.1:9555/_reindex?refresh=false" -H 'Content-Type: application/json' --cacert $CERT_FILE -u "elastic:[passwd]" -d "
  curl -s -X POST "http://127.0.0.1:9220/_reindex?refresh=false" -H 'Content-Type: application/json' -d "
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
    },
    \"script\": { \"source\": \"ctx._source.content = /<[^>]+>/ .matcher(ctx._source.content).replaceAll('')\" }
  }"
  echo
done

echo "Reindex completed."
```

发现去掉了很多原文内容，舍弃。



## 方案二

从原始数据库查询 content 字段的各个部分（text 版本的 content 被分成了 section_1, section_2, section_3）并更新 ES 的 content 字段。

logstash 配置文件： <u>update-content.conf</u>

```conf
input {
    jdbc {
      jdbc_driver_library => '/path/to/mysql-connector-java-8.0.21.jar'
      jdbc_driver_class => 'com.mysql.cj.jdbc.Driver'
      jdbc_connection_string => 'jdbc:mysql://127.0.0.1:4011/LawDB?useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true&autoReconnect=true'
      jdbc_user => 'root'
      jdbc_password => '[passwd]'
      jdbc_validate_connection => true
      jdbc_paging_enabled => 'true'
      jdbc_page_size => '1000'
      jdbc_default_timezone => 'Asia/Shanghai'
      statement => 'select id,section_1,section_2,section_3 from [your table]'
    }
}

filter {
  mutate {
    add_field => {
      "content" => "%{section_1} %{section_2} %{section_3}"
    }
  }
  mutate {
    remove_field => ["@version", "@timestamp", "section_1", "section_2", "section_3"]
  }
}

output {
    stdout {
      codec => rubydebug
    }
    elasticsearch {
      hosts => [ 'http://127.0.0.1:9220' ]
      index => '[your index]'
      action => 'update'
      document_id => '%{id}'
    }
}

```

运行 logstash

```bash
logstash -w 10 --path.data "/tmp/logstash-data" -f /path/to/update-content.conf
```

