---
title: Python 异步更新 mysql 数据库 (asyncio + aiomysql)
date: 2023-08-03 15:00:07
tags:
- python
- async
---



> 数据库有一亿数据，需要写一个垃圾脚本更新某个字段，需要 6+ 小时，太慢了！！！



## 一、下载依赖

```python
pip3 install asyncio aiomysql
```



## 二、优化脚本

```python
import os
import re
import asyncio
import aiomysql

# Replace these values with your actual MySQL database credentials
host = "127.0.0.1"
port = 3306
database = "[YOUR DB]"
user = "root"
password = "[PASSWD]"
batch_size = 10000

def parse_money(text):
    text = text.replace("万元", "0000元")

    # 共计 / 合计
    money = re.search("[共合]计(\d+(?:\.\d{1,2})?)元", text)
    if money:
        return int(float(money.group(1)))

    # xxxx 元 (只有一条记录)
    money = re.findall("(\d+(?:\.\d{1,2})?)元", text)
    if len(set(money)) == 1:
        return int(float(money[0]))

    return 0

async def query_mysql(start, end, cursor):
    sql_query = f"SELECT id,text FROM [TABLE] where id > {start} and id <= {end}"
    await cursor.execute(sql_query)
    return await cursor.fetchall()

async def update_mysql(_id, money, cursor):
    try:
        sql_query = f"UPDATE [TABLE] SET money={money} where id={_id}"
        await cursor.execute(sql_query)
    except Exception as e:
        os.system("echo {}:{}: >> error.log".format(_id, money, e))
    else:
        print(sql_query)

async def process_batch(start, end, pool):
    async with pool.acquire() as conn:
        async with conn.cursor() as cursor:
            rows = await query_mysql(start, end, cursor)

            for _id, section_truth in rows:
                text = re.search("诉讼请求.*?事实和理由", section_truth)
                if not text:
                    continue
                money = parse_money(text.group())
                if money != 0:
                    await update_mysql(_id, money, cursor)
                else:
                    print(f"{_id}[{money}]: {text.group()}")
        await conn.commit()

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
    asyncio.run(main(13379))
```

