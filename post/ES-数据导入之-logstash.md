---
title: ES 数据导入之 logstash
date: 2023-07-14 17:36:30
tags:
- ES
- Logstash
---



> [Logstash](https://www.elastic.co/cn/logstash/) 是免费且开放的服务器端数据处理管道，能够从多个来源采集数据，转换数据，然后将数据发送到您最喜欢的“存储库”中。



## 前情提要

新建了一个 ES 索引，需要从 TIDB 导入一些数据到 ES 索引中。



- ES 索引名： index_test
- tidb 地址： 127.0.0.1:4000
- es 地址： 127.0.0.1:9200, 127.0.0.1:9201, 127.0.0.1:9202



## 下载 mysql-connector-java-8.0.21.jar 包

下载链接： https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.21/mysql-connector-java-8.0.21.jar



## 创建 Logstash configuration 文件

<u>tidb-to-es.tml</u>

```
input {
    jdbc {
      jdbc_driver_library => '/path/to/mysql-connector-java-8.0.21.jar'
      jdbc_driver_class => 'com.mysql.cj.jdbc.Driver'
        jdbc_connection_string => 'jdbc:mysql://127.0.0.1:4000/LawDB?useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true&autoReconnect=true'
        jdbc_user => 'root'
        jdbc_password => '[password]'
        jdbc_validate_connection => true
        # jdbc_paging_enabled => 'true'
        # jdbc_page_size => '100'
        jdbc_default_timezone => 'Asia/Shanghai'
      statement => 'select id,title,content from [your table] where id <= 10000'
    }
}

filter {
  mutate {
    remove_field => ['@version', '@timestamp']
  }
}

output {
    stdout {
      codec => rubydebug
    }
    elasticsearch {
      hosts => [ 'http://127.0.0.1:9200', 'http://127.0.0.1:9201', 'http://127.0.0.1:9202' ]
      index => 'index_test'
      action => 'create'
      document_id => '%{id}'
      cacert => '/home/[USER]/elasticsearch-8.5.3/config/certs/http_ca.crt'
      user => "elastic"
      password => "[elastic password]"
    }
}
```



## 开始导入数据

```bash
logstash -f /path/to/tidb-to-es.conf
```



## 分页处理

当数据量很大的时候，Logstash 可自动分页导入数据

```conf
jdbc_paging_enabled => 'true'
jdbc_page_size => '100'
```

但是 ES 分页语句效率极其低下，于是自己写了个脚本分页

<u>paging.sh</u>

```bash
#!/usr/bin/env zsh

table_name="[your table]"
index_name="[your index]"
limit=10000

for i in {$1..$2}; do
  [[ -e "./stop" ]] && echo "$0 $i $2" >> "./stop" && exit 0
  offset=$(( $i * $limit ))
  text="input {
    jdbc {
      jdbc_driver_library => '/path/to/mysql-connector-java-8.0.21.jar'
      jdbc_driver_class => 'com.mysql.cj.jdbc.Driver'
      jdbc_connection_string => 'jdbc:mysql://127.0.0.1:4000/LawDB?useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true&autoReconnect=true'
      jdbc_user => 'root'
      jdbc_password => '[passwd]'
      jdbc_validate_connection => true
      # jdbc_paging_enabled => 'true'
      # jdbc_page_size => '10000'
      jdbc_default_timezone => 'Asia/Shanghai'
      statement => 'select id,section_head,section_party,section_litigation,section_truth,section_reason,section_result from $table_name where id > $offset order by id limit $limit'
    }
  }

  filter {
    mutate {
      remove_field => ['@version', '@timestamp']
    }
  }
  
  output {
      stdout {
        codec => rubydebug
      }
  	elasticsearch {
        hosts => [ 'http://127.0.0.1:9200', 'http://127.0.0.1:9201', 'http://127.0.0.1:9202' ]
        index => '$index_name'
        action => 'create'
        document_id => '%{id}'
  	}
  }
  "
  /path/to/logstash -w 10 --path.data "/tmp/logstash-$1-$2" -e "$text"
done
```

导入 100W 数据

```bash
paging.sh 0 100
```

想暂停 logstash，只需在脚本目录建立一个 stop 空文件

```bash
touch stop
```

