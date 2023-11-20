---
title: rust & sqlserver
date: 2023-10-13 16:44:30
tags:
- Rust
- sqlserver
---



## Rust 操作 sqlserver 数据库代码示例



**下载所需要的依赖**

```rust
cargo add tiberius
cargo add tokio
cargo add tokio_util --features=compat
cargo add futures
```



Rust Code:

```rust
use futures::TryStreamExt;
use tiberius::{AuthMethod, Client, Config, Query, QueryItem};
use tokio::net::TcpStream;
use tokio_util::compat::{Compat, TokioAsyncWriteCompatExt};

struct Conn {
    host: String,
    port: u16,
    user: String,
    password: String,
}

#[derive(Debug)]
struct Data {
    id: Option<i32>,
    title: Option<String>,
}

async fn client(conn: Conn) -> Client<Compat<tokio::net::TcpStream>> {
    let mut config = Config::new();
    config.host(conn.host);
    config.port(conn.port);
    config.authentication(AuthMethod::sql_server(conn.user, conn.password));
    config.trust_cert();

    let tcp = TcpStream::connect(config.get_addr()).await.unwrap();
    tcp.set_nodelay(true).unwrap();

    Client::connect(config, tcp.compat_write()).await.unwrap()
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut conn = client(Conn {
        host: "xxx.xxx.xx.xx".to_string(),
        port: 3306 as u16,
        user: "user".to_string(),
        password: "passwd".to_string(),
    })
    .await;
    println!("Successfully connected to server.");

    let sql = "select top 10 id, title from DB.dbo.table";
    let mut select = Query::new(sql);

    let mut res = select.query(&mut conn).await?;

    while let Some(row) = res.try_next().await? {
        match row {
            QueryItem::Metadata(meta) => {
                println!("{:?}", meta);
            }
            QueryItem::Row(row) => {
                println!();
                let data = Data {
                    id: row.get(0),
                    title: row.get(1).map(|f: &str| f.to_string()),
                };
                println!("{:?}", data);
            }
        }
    }

    Ok(())
}
```

