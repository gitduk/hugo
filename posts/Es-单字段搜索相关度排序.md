+++
title = 'Es 单字段搜索相关度排序'
date = 2024-03-22T16:55:31+08:00
draft = false
tags = [ "ES",  "BM25" , "tf", "idf" ]
+++

## 前因

使用 Es 搜索，发现搜索结果关键词命中数 3 的文档排在关键词命中数 20 的前面，想让 Es 搜索结果安装关键词命中数排序。

## Es 默认相关性算法

Es 5.0 之前，默认的相关性算分采用的是 TF-IDF，而之后则默认采用 BM25

用 Es 的 explain 查询可以看到详细的算分过程：
```json
GET /your_index/_search

{
  "query": {
    "match_phrase": {"content": "婚姻纠纷"}
  },
  "fields": ["title"],
  "_source": false,
  "explain": true
}
```

输出：

```json
      {
        ...
        "_explanation": {
          "value": 25.720493,
          "description": "weight(content:婚姻纠纷 in 2534301) [PerFieldSimilarity], result of:",
          "details": [
            {
              "value": 25.720493,
              "description": "score(freq=2.0), computed as boost * idf * tf from:",
              "details": [
                {
                  "value": 2.2,
                  "description": "boost",
                  "details": []
                },
                {
                  "value": 13.949452,
                  "description": "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5)) from:",
                  "details": [
                    {
                      "value": 1,
                      "description": "n, number of documents containing term",
                      "details": []
                    },
                    {
                      "value": 1714988,
                      "description": "N, total number of documents with field",
                      "details": []
                    }
                  ]
                },
                {
                  "value": 0.838107,
                  "description": "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl)) from:",
                  "details": [
                    {
                      "value": 2,
                      "description": "freq, occurrences of term within document",
                      "details": []
                    },
                    {
                      "value": 1.2,
                      "description": "k1, term saturation parameter",
                      "details": []
                    },
                    {
                      "value": 0.75,
                      "description": "b, length normalization parameter",
                      "details": []
                    },
                    {
                      "value": 80,
                      "description": "dl, length of field (approximate)",
                      "details": []
                    },
                    {
                      "value": 834.0069,
                      "description": "avgdl, average length of field",
                      "details": []
                    }
                  ]
                }
              ]
            }
          ]
        }
      },
```


文档分数 = boost \* idf \* tf
idf = log(1 + (N - n + 0.5) / (n + 0.5))
tf = freq / (freq + k1 \* (1 - b + b \* dl / avgdl))

| 变量    | 解释                                                                                                                                                                                    |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| boost | 权重分数，这个可以在查询语句中传入                                                                                                                                                                     |
| idf   | 逆文档率 = log(1 + (N - n + 0.5) / (n + 0.5))<br><br>N - 总的文档数<br>n - 出现“联合”一次的文档数。                                                                                                       |
| tf    | 词频 = freq / (freq + k1 \* (1 - b + b \* dl / avgdl)) <br><br>freq - 搜索词在该文档出现的次数<br>k1 - 控制词频影响的重要程度，默认值 1.2<br>b - 控制对大长文本字段的惩罚程度，默认值 0.75<br>dl - 该文档中检索字段的长度<br>avgdl - 检索字段的平均长度。 |
## BM25 调参

调整方式，只能在创建索引的时候调整，无法动态调整。
创建自定义的 `custom_similarity`，参数 b 的值设置为 0，k1 保持默认的 1.2，当 b = 0 时，tf 公式就变成了 freq / (freq + k1)，tf 分数完全忽略文本长度的影响。

```json
PUT /your_index

{
  "settings": {
    "similarity": {
      "custom_similarity": {
        "type": "BM25",
        "b": "0",
        "k1": "1.2"
      }
    }
  },
  "mappings": {
    "properties": {
      "id": {
        "type": "integer",
        "index": true
      },
      "title": {
        "type": "text",
        "index": true,
        "similarity": "custom_similarity"
      }   
    }
  }
}
```

但是 idf 中 的 N 和 n 取决于搜索词和数据，无法修改，只能调 tf 的整参数 b 和 k1，无法完全消除文本长度对相关性的影响。

## 自定义 similarity 算法

建立索引，自定义 similarity：
```json
PUT /index
{
  "settings": {
    "number_of_shards": 1,
    "similarity": {
      "scripted_tfidf": {
        "type": "scripted",
        "script": {
          "source": "double tf = Math.sqrt(doc.freq); double idf = Math.log((field.docCount+1.0)/(term.docFreq+1.0)) + 1.0; double norm = 1/Math.sqrt(doc.length); return query.boost * tf * idf * norm;"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "field": {
        "type": "text",
        "similarity": "scripted_tfidf"
      }
    }
  }
}
```


导入一些数据：
```json
PUT /index/_doc/1
{
  "field": "foo bar foo"
}

PUT /index/_doc/2
{
  "field": "bar baz"
}

POST /index/_refresh
```


搜索测试：
```json
GET /index/_search?explain=true
{
  "query": {
    "query_string": {
      "query": "foo^1.7",
      "default_field": "field"
    }
  }
}
```

搜索结果：
```json
{
  "took": 12,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
        "value": 1,
        "relation": "eq"
    },
    "max_score": 1.9508477,
    "hits": [
      {
        "_shard": "[index][0]",
        "_node": "OzrdjxNtQGaqs4DmioFw9A",
        "_index": "index",
        "_id": "1",
        "_score": 1.9508477,
        "_source": {
          "field": "foo bar foo"
        },
        "_explanation": {
          "value": 1.9508477,
          "description": "weight(field:foo in 0) [PerFieldSimilarity], result of:",
          "details": [
            {
              "value": 1.9508477,
              "description": "score from ScriptedSimilarity(weightScript=[null], script=[Script{type=inline, lang='painless', idOrCode='double tf = Math.sqrt(doc.freq); double idf = Math.log((field.docCount+1.0)/(term.docFreq+1.0)) + 1.0; double norm = 1/Math.sqrt(doc.length); return query.boost * tf * idf * norm;', options={}, params={}}]) computed from:",
              "details": [
                {
                  "value": 1.0,
                  "description": "weight",
                  "details": []
                },
                {
                  "value": 1.7,
                  "description": "query.boost",
                  "details": []
                },
                {
                  "value": 2,
                  "description": "field.docCount",
                  "details": []
                },
                {
                  "value": 4,
                  "description": "field.sumDocFreq",
                  "details": []
                },
                {
                  "value": 5,
                  "description": "field.sumTotalTermFreq",
                  "details": []
                },
                {
                  "value": 1,
                  "description": "term.docFreq",
                  "details": []
                },
                {
                  "value": 2,
                  "description": "term.totalTermFreq",
                  "details": []
                },
                {
                  "value": 2.0,
                  "description": "doc.freq",
                  "details": []
                },
                {
                  "value": 3,
                  "description": "doc.length",
                  "details": []
                }
              ]
            }
          ]
        }
      }
    ]
  }
}
```

可以使用的值已经在上面的 description 里面列举出来。

## 参考文档
- https://blog.51cto.com/u_15812686/6453674
- https://www.elastic.co/guide/en/elasticsearch/reference/8.5/index-modules-similarity.html#scripted_similarity
