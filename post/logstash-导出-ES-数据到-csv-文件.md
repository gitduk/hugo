---
title: logstash 导出 ES 数据到 csv 文件
date: 2023-08-15 17:51:57
tags:
- logstash
---





> 有一个需求，把 ES 索引里面的某个字段导入 csv 文件里面。



## logstash 配置文件

```conf
input {
  elasticsearch {
    hosts => [ 'http://127.0.0.1:9200' ]
    index => '[index]'
    user => "[user]"
    password => "[password]"
    query => '{
      "query": {
        "range": {
          "id": {
            "gte":0,
            "lte":100
          }
        }
      },
      "_source": ["content"]
    }'
  }
}

output {
  csv {
    fields => ["content"]
    path => "content.csv"
  }
  stdout {
    codec => rubydebug
  }
}

```



运行 logstash

```bash
/path/to/logstash -f "/path/to/config/file"
```

