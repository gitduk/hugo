---
title: 文本转向量并插入到 ES 中
date: 2023-07-03 17:52:05
tags:
- ES
---



## ES 中创建索引

```
PUT vector-index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text"
      },
      "content_vector": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "l2_norm"
      }
    }
  }
}
```



## 导入一些数据

使用 logstash 工具从 mysql 数据库导入数据

```
input {
    jdbc {
      jdbc_driver_library => 'path/to/mysql-connector-java-8.0.21.jar'
      jdbc_driver_class => 'com.mysql.cj.jdbc.Driver'
        jdbc_connection_string => 'jdbc:mysql://[IP ADDR]:[POST]/[DATABASE]?useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true&autoReconnect=true'
        jdbc_user => 'root'
		jdbc_password => 'password'
        jdbc_validate_connection => true
        # jdbc_paging_enabled => 'true'
        # jdbc_page_size => '100'
        jdbc_default_timezone => 'Asia/Shanghai'
      statement => 'select id,title from [your table] limit 10000'
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
      hosts => [ 'https://[ES IP ADDR]:[ES POST]' ]
      index => '[INDEX NAME]'
      action => 'create'
      document_id => '%{id}'
      cacert => '/path/to/http_ca.crt'
      user => "elastic"
      password => "[elastic password]"
	}
}

```



## 文本转向量并插入 ES 

python 脚本

```python
import os
import pymysql
import pandas as pd
from rich.progress import Progress, TextColumn, BarColumn, TimeElapsedColumn, TimeRemainingColumn, MofNCompleteColumn
from text2vec import SentenceModel
import requests
from requests.auth import HTTPBasicAuth

es_url = "https://[ES IPADDR]:[ES POST]"
es_username = "elastic"  # 替换为实际的用户名
es_password = "[password]"  # 替换为实际的密码

conn = pymysql.connect(
    host="[MYSQL HOST]", user="[user]", password="[password]", database="[DATABASE]", port=[your post],
    cursorclass=pymysql.cursors.DictCursor
)
cursor = conn.cursor()

# load model
model = SentenceModel('shibing624/text2vec-base-chinese')

# insert func
def to_es(_id, vector):
    data = {
        "doc": {
            "content_vector": vector.tolist()
        }
    }
    url = f"{es_url}/precedent_vector/_update/{_id}"
    # crt 证书生成 pem 证书：openssl x509 -in http_ca.crt -out http_ca.pem
    response = requests.post(url, json=data, auth=HTTPBasicAuth(es_username, es_password), verify='http_ca.pem')
    # 检查响应状态码
    if response.status_code != 200:
        msg = "{}:{}".format(_id, response.text)
        os.system("echo '{}' >> error.log".format(msg))
        print(msg)

with Progress(TextColumn("[progress.description]{task.description}"),
              MofNCompleteColumn(),
              BarColumn(),
              TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
              TimeRemainingColumn(),
              TimeElapsedColumn()) as progress:
    start = 0
    end = 100
    batch_size = 100
    batch = progress.add_task(description="Batch", total=end - start)
    task = progress.add_task(description="Text Embedding", total=3)
    insert = progress.add_task(description="Insert To ES", total=batch_size)

    for i in range(start, end):
        start = i * batch_size

        # step one
        progress.update(task, advance=1, description="Query Data")
        sql = f"select id,section_head,section_party,section_litigation,section_truth,section_reason,section_result from precedent where id > {start} order by id limit {batch_size}"
        os.system("echo '{}' > ./query.sql".format(sql))
        cursor.execute(sql)
        results = cursor.fetchall()

        # step two
        progress.update(task, advance=1, description="Create DataFrame")
        df = pd.DataFrame(results)
        df["content"] = df["section_head"].str.cat(
            [df["section_party"], df["section_litigation"], df["section_truth"], df["section_reason"],
             df["section_result"]], sep="\n")

        # step three
        progress.update(task, advance=1, description=f"Text Embedding")
        vectors = model.encode(df["content"].tolist(), show_progress_bar=True)

        # insert to es
        for _id, vector in zip(df.id.tolist(), vectors):
            progress.update(insert, advance=1)
            to_es(_id, vector)
        progress.reset(insert)

        progress.update(batch, advance=1)
        progress.reset(task)
    progress.reset(batch)
```



**数据量不大，并且希望可以实时看到插入进度，可以用上面的代码，追求 ES 更新速度，请移步: [ES 数据插入速度优化，从 4.5h -> 40min](https://blog.wukaige.com/ES-%E6%95%B0%E6%8D%AE%E6%8F%92%E5%85%A5%E9%80%9F%E5%BA%A6%E4%BC%98%E5%8C%96%EF%BC%8C%E4%BB%8E-4-5h-40min/)**
