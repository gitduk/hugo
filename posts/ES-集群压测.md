+++
title = "ES 集群压测"
date = "2023-11-21"
tags = ["ES", "esrally"]

+++



新进了一台高性能服务器，准备把旧的 ES 集群迁移到新服务器，需要对比两个 ES 集群的性能。



## 一些信息

旧服务器 ES 地址:

- 192.168.0.1:9200

新服务器 ES 地址:

- 192.168.0.2:9200



## 测试工具

[esrally](https://github.com/elastic/rally?spm=a2c6h.12873639.article-detail.10.56e43a96p8hfqD) 是 elastic 官方开源的一款基于 python3 实现的针对 es 的压测工具。



#### 查询压测

> 自定义查询压测: 对当前索引进行自定义 dsl 查询压测



自定义 track:

```bash
# 创建 track 文件夹
mkdir track_1 && cd track_1

# 获取索引的 map 信息
curl -u "elastic:passwd" --cacert /path/to/es/http_ca.crt "https://192.168.0.1:9200/your_index/_mapping?pretty=true" > your_index.json

# 创建 track.json 文件
touch track.json
```

<u>track.json</u>

```json
{% import "rally.helpers" as rally with context %}
{
  "version": 2,
  "description": "Tracker-generated track for test",
  "indices": [
    {
      "name": "your_index",
      "body": "your_index.json"
    }
  ],
  "schedule": [
    {
      "operation": {
        "name": "query-dsl",
        "operation-type": "search",
        "body": {
          "query": {
            "match_phrase": {
              "content": "搜索字符串"
            }
          }
        }
      },
      "warmup-time-period": 10,
      "time-period": 120,
      "target-throughput": 1000,
      "clients": 100
    }
  ]
}
```

- `warmup-time-period` 预热时间 10 s
- `time-period` 测试时间 120 s
- `target-throughput` 目标吞吐量 1000
- `clients` 客户端数量 100



对 ES 集群进行测试

```bash
esrally race --track-path=/path/to/track_1 --pipeline=benchmark-only --target-hosts=192.168.0.1:9200 --client-options="use_ssl:true,verify_certs:true,ca_certs:'/path/to/old_es/http_ca.crt',basic_auth_user:'elastic',basic_auth_password:'passwd'" --kill-running-processes --report-file=track_1_old.csv
```

对新 ES 集群进行测试

```bash
esrally race --track-path=/path/to/track_1 --pipeline=benchmark-only --target-hosts=192.168.0.2:9200 --client-options="use_ssl:true,verify_certs:true,ca_certs:'/path/to/new_es/http_ca.crt',basic_auth_user:'elastic',basic_auth_password:'passwd'" --kill-running-processes --report-file=track_1_new.csv
```

对比测试结果

```bash
esrally compare --baseline=9c42f589-df61-45f2-902c-9dd742877c43 --contender=a51179ee-91f6-444d-87af-a2cb60a46159
```

- 其中 `9c42f589-df61-45f2-902c-9dd742877c43` 和 `a51179ee-91f6-444d-87af-a2cb60a46159` 分别是两次测试的 Race ID.
- `esrally list races` 命令可查看 Race ID.



## 参考链接

- https://cloud.tencent.com/developer/article/1973178

