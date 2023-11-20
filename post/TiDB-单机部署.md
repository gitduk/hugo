---
title: TiDB 单机部署
date: 2023-10-23 15:37:51
tags:
- TiDB
---





## 机器配置

- OS: Ubuntu 23.04

- CPU: 20 核
- Mem: 32 G



## 安装 tiup 组件

```bash
# 安装 tiup 命令
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh

# 升级 cluster 组件
tiup update --self && tiup update cluster

# 验证 tiup 版本信息
tiup --binary cluster
```



## 准备集群拓扑文件

<u>topo.yaml</u>

```yaml
# # Global variables are applied to all deployments and used as the default value of
# # the deployments if a specific deployment value is missing.
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/home/tidb/tidb-deploy"
  data_dir: "/home/tidb/tidb-data"

# # Monitored variables are applied to all the machines.
monitored:
  node_exporter_port: 9100
  blackbox_exporter_port: 9115

server_configs:
  tidb:
    instance.tidb_slow_log_threshold: 300
    performance.txn-entry-size-limit: 62914560      # 60M
    performance.txn-total-size-limit: 1073741824    # 1G

  tikv:
    storage.block-cache.shared: true
    readpool.storage.use-unified-pool: true
    readpool.coprocessor.use-unified-pool: true
    readpool.unified.max-thread-count: 10
    raftstore.raft-entry-max-siz: "10MB"

  pd:
    replication.enable-placement-rules: true
    replication.location-labels: ["host"]

  tiflash:
    logger.level: "info"

pd_servers:
  - host: 192.168.70.45

tidb_servers:
  - host: 192.168.70.45

tikv_servers:
  - host: 192.168.70.45
    port: 20160
    status_port: 20180
    config:
      server.labels: { host: "logic-host-1" }

  - host: 192.168.70.45
    port: 20161
    status_port: 20181
    config:
      server.labels: { host: "logic-host-2" }

  - host: 192.168.70.45
    port: 20162
    status_port: 20182
    config:
      server.labels: { host: "logic-host-3" }

tiflash_servers:
  - host: 192.168.70.45

monitoring_servers:
  - host: 192.168.70.45

grafana_servers:
  - host: 192.168.70.45

```



## 配置系统环境

```bash
# 关闭系统 SWAP
echo "vm.swappiness = 0" | sudo tee -a /etc/sysctl.conf
sudo swapoff -a

# 禁用 THP
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
sudo sysctl -p

# 配置网络参数
echo "fs.file-max = 1000000" | sudo tee -a /etc/sysctl.conf
echo "net.core.somaxconn = 32768" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_tw_recycle = 0" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_syncookies = 0" | sudo tee -a /etc/sysctl.conf
echo "vm.overcommit_memory = 1" |sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 配置 tidb 用户的打开文件句柄限制
cat << EOF | sudo tee -a /etc/security/limits.conf
tidb           soft    nofile          1000000
tidb           hard    nofile          1000000
tidb           soft    stack           32768
tidb           hard    stack           32768
EOF

# 将 CPU 频率调频模式从 powersave 设置为 performance
echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 禁用 SELinux
sudo sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# mount disk with noatime
sudo mount -o remount,noatime /

```



## 部署集群

**检查集群存在的潜在风险**

```bash
tiup cluster check ./topo.yaml --user root -p
```



**自动修复集群存在的潜在风险**

```bash
tiup cluster check ./topo.yaml --apply --user root -p
```



**部署 TiDB 集群**

```bash
tiup cluster deploy tidb v7.1.1 ./topo.yaml --user root -p
```



**初始化 TiDB 集群**

```bash
tiup cluster start tidb --init
```

```output
Started cluster `tidb` successfully
The root password of TiDB database has been changed.
The new password is: 'j7yz516-bRm*8XY+@9'.
Copy and record it to somewhere safe, it is only displayed once, and will not be stored.
The generated password can NOT be get and shown again.
```



**修改密码**

```bash
mysql -h 127.0.0.1 -P 4000 -u root -p'j7yz516-bRm*8XY+@9'
```

```mysql
SET PASSWORD FOR 'root'@'%' = 'xxxxxx';
```

