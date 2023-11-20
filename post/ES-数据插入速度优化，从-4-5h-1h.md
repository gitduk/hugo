---
title: ES 数据插入速度优化，从 4.5h -> 1h
date: 2023-08-04 09:19:10
tags:
- ES
- async
---



> 写了个脚本把数据库里面的 content 字段转换为向量插入到 ES， 100w 数据需要 4.5 小时，太慢了！！！



## 一、慢速度脚本（4.5h）

只用 aiomysql 对数据库进行异步请求，ES 更新则是一条一更新。

```python
import os
import asyncio
import aiomysql
import requests
from text2vec import SentenceModel
from requests.auth import HTTPBasicAuth

# tidb
host = "127.0.0.1"
port = 3306
database = "[database]"
user = "root"
password = "[password]"
batch_size = 10000

# es
es_url = "http://127.0.0.1:9200"
# es_user = "[elastic]"
# es_user = "[password]"

# load model
model = SentenceModel("shibing624/text2vec-base-chinese")


async def query_mysql(start, end, cursor):
    sql_query = f"select id,content from [table] where id > {start} and id <= {end}"
    await cursor.execute(sql_query)
    return await cursor.fetchall()


async def update_es(data):
    _id = data.get("id")
    url = f"{es_url}/[INDEX]/_doc/{_id}"
    # 如果 ES 开启了安全验证
    # response = requests.put(url, json=data，auth=HTTPBasicAuth(es_username, es_password))
    response = requests.put(url, json=data)

    # 检查响应状态码
    if response.status_code in [200, 201]:
        msg = f"[{_id}] 插入成功"
    else:
        msg = f"[{id}] 插入失败: {response.text}"
        os.system("echo '{}' >> ./error.log".format(msg))
    print(msg)

    
async def process_batch(start, end, pool):
    async with pool.acquire() as conn:
        async with conn.cursor() as cursor:
            rows = await query_mysql(start, end, cursor)
            ids = [row[0] for row in rows]
            contents = [row[1] for row in rows]
            vector = model.encode(contents, show_progress_bar=True)
            for _id, vec in zip(ids, vector.tolist()):
                data = {
                    "id": _id,
                    "content_vector": vec
                }
                await update_es(data)

                
async def main(batches):
    pool = await aiomysql.create_pool(host=host, port=port, user=user, password=password, db=database)
    tasks = []
    for i in range(batches):
        start = i * batch_size
        end = (i + 1) * batch_size
        tasks.append(process_batch(start, end, pool))

    await asyncio.gather(*tasks)
    pool.close()
    await pool.wait_closed()

    
if __name__ == "__main__":
    asyncio.run(main(100))
```



运行截图(4h 30m 21s)：
![image-20230804092603804](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230804092603804.png)



## 二、初步优化（1h）

安装 elasticsearch[async] 模块

```bash
pip3 install 'elasticsearch[async]'
```

使用 es bulk 批量更新数据，加速数据插入。

```python
import os
import asyncio
import aiomysql
from text2vec import SentenceModel
from elasticsearch import AsyncElasticsearch
from elasticsearch.helpers import async_bulk

host = '127.0.0.1'
port = 3306
database = '[database]'
user = 'root'
password = '[password]'
batch_size = 10000

# load model
model = SentenceModel('shibing624/text2vec-base-chinese')

# es client
# 如果加了安全验证机制：
# es = AsyncElasticsearch(["http://127.0.0.1:9200"], basic_auth=("elastic", "[password]"))
es = AsyncElasticsearch(["http://127.0.0.1:9200"])


def build_bulk_data(ids, vectors):
  	for _id, vec in zip(ids, vectors):
    	data = {
      		"_index": index,
      		"_id": _id,
      		"_source": {
        		"id": _id, 
        		"content_vector": vec
            }
        }
    yield data


async def query_mysql(start, end, cursor):
    sql_query = f"select id,content from [table] where id > {start} and id <= {end}"
    await cursor.execute(sql_query)
    return await cursor.fetchall()


async def process_batch(start, end, pool):
    async with pool.acquire() as conn:
        async with conn.cursor() as cursor:
            # query mysql
            rows = await query_mysql(start, end, cursor)
            ids = [row[0] for row in rows]
            contents = [row[1] for row in rows]

            # encode content
            vectors = model.encode(contents, show_progress_bar=True)

            # build bulk data
            # 内存不足的情况下，可以用生成器代替列表:
            # bulk_data = build_bulk_data(ids, vectors)
            bulk_data = [{"_index": index, "_id": _id, "_source": {"id": _id, "content_vector": vec}} for _id, vec in zip(ids, vectors)]
            try:
                resp = await async_bulk(es, bulk_data, chunk_size=500, max_retries=3, request_timeout=60*3)
            except Exception as e:
                print(f"[{start}-{end}] Error: {e}")
            else:
                print(f"[{start}-{end}] SUCCESS: {resp[0]}")


async def main(batches, offset=0):
    # mysql pool
    pool = await aiomysql.create_pool(host=host, port=port, user=user, password=password, db=database)

    # create task list
    tasks = []
    for i in range(offset, offset + batches):
        start = i * batch_size
        end = (i + 1) * batch_size
        tasks.append(process_batch(start, end, pool))

    await asyncio.gather(*tasks)
    pool.close()
    await pool.wait_closed()


if __name__ == "__main__":
    asyncio.run(main(100, offset=100))
```

运行截图(1h 3m 9s)：

![image-20230804112915854](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230804112915854.png)

## 三、内存优化

优化目的：减少内存使用

优化策略：

- 使用 async_streaming_bulk 代替 async_bulk
- 使用 asyncio.as_completed 代替 asyncio.gather



async_streaming_bulk 和 async_bulk 的区别：

- async_bulk 会将所有请求收集到一个列表中，然后一次性发送给 Elasticsearch，它会等待所有的请求完成，然后返回结果。

- async_streaming_bulk 则是一边收集请求，一边流式地发送给 Elasticsearch，不需要等待所有的请求收集完成，它会实时返回每个请求的结果。
- async_bulk 适用于需要等待所有请求完成才返回的场景，但是需要消耗更多内存来 hold 住所有请求。
- async_streaming_bulk 适用于对实时返回每个结果更敏感的场景，它消耗的内存更少。
- sync_bulk 的性能会比 async_streaming_bulk 好，因为它是批量发送请求。但是 async_streaming_bulk 更适合处理大量请求，不会因为内存消耗太大而出问题。



asyncio.as_completed 和 asyncio.gather 的区别：

- asyncio.gather 会同时启动所有的异步任务，等待它们全部完成后才返回结果。
- asyncio.as_completed 会按任务完成的顺序返回结果，不会等待所有的任务完成。
- asyncio.gather 如果有一个任务 raise 异常会立即停止其它任务。asyncio.as_completed 不会，会继续返回其它已经完成的任务的结果。
- asyncio.gather 的结果顺序和传入的任务顺序一致。asyncio.as_completed 的结果顺序依赖于任务完成的时间，与传入顺序无关。
- asyncio.gather 更适合需要同时开始执行多个任务，并需要所有任务结果才能继续的场景。asyncio.as_completed 更适合只关心任务完成就使用结果的场景。
- asyncio.gather 的性能和资源消耗可能更高，因为同时启动和跟踪更多任务。
- asyncio.as_completed 可以通过返回完成任务来提早优化或发出决策。



优化后的代码：

```python
import os
import json
import asyncio
import aiomysql
import aiohttp
from text2vec import SentenceModel
from elasticsearch import AsyncElasticsearch
from elasticsearch.helpers import async_streaming_bulk

host = '127.0.0.1'
port = 3306
database = '[DATABASE]'
user = 'root'
password = '[PASSWD]'
batch_size = 10000

# load model
model = SentenceModel('shibing624/text2vec-base-chinese')

# es client
es = AsyncElasticsearch(["http://127.0.0.1:9220"])


def build_bulk_data(ids, vectors):
    for _id, vec in zip(ids, vectors):
        data = {
            "_index": index,
            "_id": _id,
            "_source": {
                "id": _id, 
                "content_vector": vec
            }
        }
        yield data


async def query_mysql(start, end, cursor):
    sql_query = f"select id,content from [table] where id > {start} and id <= {end}"
    await cursor.execute(sql_query)
    return await cursor.fetchall()


async def process_batch(start, end, pool):
    async with pool.acquire() as conn:
        async with conn.cursor() as cursor:
            # query mysql
            rows = await query_mysql(start, end, cursor)
            ids = [row[0] for row in rows]
            contents = [row[1] for row in rows]

            # encode content
            vectors = model.encode(contents, show_progress_bar=True)

            # build bulk data
            # bulk_data = [{"_index": index, "_id": _id, "_source": {"id": _id, "content_vector": vec}} for _id, vec in zip(ids, vectors)]
            bulk_data = build_bulk_data(ids, vectors)
            async for ok, result in async_streaming_bulk(es, bulk_data, chunk_size=5000, max_retries=3, request_timeout=60*3):
                if not ok:
                    os.systemd("echo '{}' >> ./error.log".format(result))
                else:
                    print(result)


async def main(batches, offset=0):

    # mysql pool
    pool = await aiomysql.create_pool(host=host, port=port, user=user, password=password, db=database)

    # task list
    tasks = [process_batch(i * batch_size, (i + 1) * batch_size, pool) for i in range(offset, offset + batches)]

    # wait task completed
    for completed in asyncio.as_completed(tasks):
        await completed

    pool.close()
    await pool.wait_closed()


if __name__ == "__main__":
    asyncio.run(main(100))
```

运行截图(59m 28s)：

![image-20230814172629394](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230814172629394.png)
