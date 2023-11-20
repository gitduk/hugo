---
title: TiDB 扩容
date: 2023-11-17 18:07:51
tags:
- TiDB
---

## 编写扩容配置文件

<u>scale-out.yml</u>

```conf
tidb_servers:
  - host: 127.0.0.1
    ssh_port: 22
    port: 4001
    status_port: 10081
    numa_node: "1" # suggest numa node bindings.   
```

## 扩容命令

检查配置文件

```bash
tiup cluster check tidb scale-out.yml --cluster --user root -p
```

自动修复配置文件

```bash
tiup cluster check tidb scale-out.yml --cluster --apply --user root -p
```

扩容

```bash
tiup cluster scale-out tidb scale-out.yml -u root -p
```

