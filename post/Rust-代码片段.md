---
title: Rust 代码片段
date: 2023-09-22 09:37:56
tags:
- Rust
---



## mysql

> https://rustmagazine.github.io/rust_magazine_2021/chapter_3/rust-mysql.html

```rust	
use mysql::{Pool, PooledConn, from_row};

fn my_conn(url: &str) -> Result<PooledConn, mysql::Error> {
    let pool = Pool::new(url)?;
    let conn = pool.get_conn()?;
    Ok(conn)
}

fn main() {
    let url = "mysql://xxx:xxx@127.0.0.1:3306/DB";
    let conn = my_conn(url).unwrap();
    let sql = "select title from table;";
    
    // query map
    let _books = conn.query_map(sql, |title: String| {
        title
    })?;
    
    // query iter
    let sql = "select id,title,date from table2;";
    conn.query_iter(query)?.for_each(|row| {
        let r: (i32, String, String) = from_row(row.unwrap());
        println!("{} {} {}", r.0, r.1, r.2);
    });
}

```



## system command

```rust
fn run_command(cmd: String) -> Result<(), Box<dyn std::error::Error>> {
    let status = Command::new("sh")
        .arg("-c") 
        .arg(&cmd)
        .status()?;

    if !status.success() {
        println!("命令执行失败: {}", cmd);
    }

    Ok(())
}

fn main() {
    let cmd = "echo hello".to_string();
    if let Err(e) = run_command(cmd) {
        eprintln!("执行 cmd 时出现错误: {:?}", e);
    }
}

```





## parse json file

```rust
use serde_json::Value;

fn json_data(path: &str) -> Result<Value, std::io::Error> {
    let data = fs::read_to_string(path)?;
    let json: Value = serde_json::from_str(&data)?;

    Ok(json)
}

fn main() {
    let path = "file.json";
    let _filter_array = json_data(path).unwrap();
}
```



## get env value

```bash
cargo run -- 1 100
```

```rust
fn main() {
    let args: Vec<_> = env::args().collect();
    let start = &args[1].parse::<usize>().unwrap();
    let end = &args[2].parse::<usize>().unwrap();
}
```



## get cpu cores

```rust
let cores = std::thread::available_parallelism().unwrap();
```



## hashmap check keys

```rust
map.contains_key(key)
```



## String to Vec<&str>

```rust
let s = "".to_string()
	+ "line one\n"
	+ "line two\n"
	+ "line three";

let v: Vec<&str> = s.split(',').collect();
```

