---
title: Rust 操作 mysql 数据库
date: 2023-10-17 16:35:57
tags:
- Rust
- mysql
---



## 安装所需要的包

```bash
cargo add mysql
```



## 连接数据库

``` rust
use mysql::{prelude::Queryable, *};
use std::fs;

fn my_conn(url: &str) -> Result<PooledConn, mysql::Error> {
    let pool = Pool::new(url)?;
    let conn = pool.get_conn()?;
    Ok(conn)
}

fn main() {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = my_conn(url).unwrap();
}
```



## 建表

```rust
use mysql::{prelude::Queryable, *};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    conn.query_drop(r"
		CREATE TABLE table (
          `id` int not null auto_increment comment 'id',
  		  `title` char(255) default '' comment 'title',
  		  PRIMARY KEY(id)
		) comment 'test table';
        ")?;

    Ok(())
}
```





## 插入数据

**一条一条插入（10000条数据/46s）**

```rust
use mysql::{prelude::Queryable, *};

fn csv_data() -> String {
    let path = "lines.txt";
    std::fs::read_to_string(path).unwrap()
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    let data = csv_data();

    for line in data.lines() {
        let sql = format!("INSERT INTO table (title) VALUES ('{line}')");
        let res = conn.query_drop(sql);

        match res {
            Ok(_) => println!("insert: {line}"),
            Err(e) => println!("{e}"),
        }
    }

    Ok(())
}
```

**批量插入（10000条数据/45s）**

```rust
use mysql::{prelude::Queryable, *};

fn csv_data() -> String {
    let path = "lines.txt";
    std::fs::read_to_string(path).unwrap()
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    let data = csv_data();

    conn.exec_batch(
        r"INSERT INTO table (title) VALUES (:title)",
        data.lines().map(|title| {
            params! {
                "title" => title
            }
        }),
    )?;

    Ok(())
}
```



**transaction 分批插入（10000条数据/0.04s）**

```rust
use mysql::{prelude::Queryable, *};

fn csv_data() -> String {
    let path = "lines.txt";
    std::fs::read_to_string(path).unwrap()
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let data = csv_data();

    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    let mut trans = conn
        .start_transaction(TxOpts::default())
        .expect("error trans");

    let mut batch_params = Vec::new();
    let batch_size = 1000;

    for (i, line) in data.lines().enumerate() {
        batch_params.push(params! {
            "title" => line,
        });

        if batch_params.len() >= batch_size {
            match trans.exec_batch(
                r"INSERT INTO table (title) VALUES (:title)",
                batch_params.clone(),
            ) {
                Ok(_) => {}
                Err(err) => {
                    println!("Failed to insert batch starting at line {}: {:?}", i, err);
                }
            }
            batch_params.clear();
            trans.commit().expect("error commit");
            trans = conn
                .start_transaction(TxOpts::default())
                .expect("error trans");
        }
    }
    if !batch_params.is_empty() {
        match trans.exec_batch(
            r"INSERT INTO table (title) VALUES (:title)",
            batch_params.clone(),
        ) {
            Ok(_) => {}
            Err(err) => {
                println!("Failed to insert batch starting: {:?}", err);
            }
        }
        batch_params.clear();
        trans.commit().expect("error commit");
    }

    Ok(())
}
```



## 查询数据

**检查数据是否存在，而不关心具体的值**

```rust
use mysql::{prelude::Queryable, *};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://root:mysql@127.0.0.1:3306/LawDB";
    let mut conn = Pool::new(url)?.get_conn()?;

    match conn
        .query_first("select title from table where id=1314")
        .map(|row| row.map(|title: String| title))?
    {
        Some(_) => println!("YES"),
        None => println!("NO"),
    }

    Ok(())
}
```



**查询一条数据**

```rust
use mysql::{prelude::Queryable, *};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    let data: (i32, String) = match conn
        .query_first("select id,title from table where id=1")
        .map(|row| row.map(|(id, title): (i32, String)| (id, title)))?
    {
        Some((id, title)) => (id, title),
        None => (-1, "".to_string()),
    };

    println!("result: {:?}", data);

    Ok(())
}
```



**查询多条数据: query_map**

```rust
use mysql::{prelude::Queryable, *};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    let datas = conn.query_map(
        "select id,title from table limit 10;",
        |(id, title): (i32, String)| (id, title),
    )?;

    for d in datas {
        println!("{}: {}", d.0, d.1);
    }

    Ok(())
}
```



**查询多条数据: query_iter**

```rust
use mysql::{prelude::Queryable, *};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    conn.query_iter("select id,title from table limit 10;")?
        .for_each(|row| {
            let r: (i32, String) = from_row(row.unwrap());
            println!("{}: {}", r.0, r.1);
        });

    Ok(())
}
```

```rust
use mysql::{prelude::Queryable, *};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    conn.query_iter("select id,title from table limit 10;")?
        .for_each(|row| {
            let (id, title): (i32, String) = from_row(row.unwrap());
            println!("{}: {}", id, title);
        });

    Ok(())
}
```



## 删除数据

```rust
use mysql::{prelude::Queryable, *};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "mysql://user:password@127.0.0.1:3306/DB";
    let mut conn = Pool::new(url)?.get_conn()?;

    conn.query_drop(r"delete from table")?;

    Ok(())
}
```

