---
title: TiDB 多物理机集群部署
date: 2023-07-10 15:50:41
tags:
- TiDB
---



## 前情提要

买了三台服务器，内网 IP 如下：

- 192.168.70.50
- 192.168.70.51
- 192.168.70.52

OS: RedHat

想搭建一个 TIDB 集群，只要有一台机器存活就能正常对外提供服务。



## 服务器环境准备

关闭系统 SWAP

```bash
echo "vm.swappiness = 0" | sudo tee -a /etc/sysctl.conf
sudo swapoff -a
```

禁用 THP

```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
sudo sysctl -p
```

配置网络参数

```bash
echo "fs.file-max = 1000000" | sudo tee -a /etc/sysctl.conf
echo "net.core.somaxconn = 32768" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_tw_recycle = 0" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_syncookies = 0" | sudo tee -a /etc/sysctl.conf
echo "vm.overcommit_memory = 1" |sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

配置 tidb 用户的打开文件句柄限制

```bash
cat << EOF | sudo tee -a /etc/security/limits.conf
tidb           soft    nofile          1000000
tidb           hard    nofile          1000000
tidb           soft    stack           32768
tidb           hard    stack           32768
EOF
```

将 CPU 频率调频模式从 powersave 设置为 performance

```bash
echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

关闭防火墙

```bash
# redhat
sudo systemctl stop firewalld.service
```

禁用 SELinux

```bash
sudo sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

mount disk with noatime

```bash
sudo mount -o remount,noatime /
```



## 部署集群

编写集群拓扑文件: <u>topology.yaml</u>

```yaml
# # Global variables are applied to all deployments and used as the default value of
# # the deployments if a specific deployment value is missing.
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/SSD/tidb-deploy"
  data_dir: "/SSD/tidb-data"

monitored:
  node_exporter_port: 9800
  blackbox_exporter_port: 9815

server_configs:
  tidb:
    log.slow-threshold: 300
    performance.txn-entry-size-limit: 8388608    # 8M
  tikv:
    storage.block-cache.shared: true
    readpool.storage.use-unified-pool: true
    readpool.coprocessor.use-unified-pool: true
    readpool.unified.max-thread-count: 80
    raftstore.raft-entry-max-siz: "8MB"
  pd:
    replication.location-labels: ["host"]
    replication.max-replicas: 3
    schedule.leader-schedule-limit: 4
    schedule.region-schedule-limit: 2048
    schedule.replica-schedule-limit: 64

pd_servers:
  - host: 192.168.70.48
    client_port: 2079
    peer_port: 2080
	  # deploy_dir: "/home/tidb/tidb-deploy/pd-2079"
    # data_dir: "/home/tidb/tidb-data/pd-2079"
    # log_dir: "/home/tidb/tidb-deploy/pd-2079/log"
  - host: 192.168.70.50
    client_port: 2079
    peer_port: 2080
  - host: 192.168.70.51
    client_port: 2079
    peer_port: 2080

tidb_servers:
  - host: 192.168.70.48
    port: 4011
    status_port: 17080
    deploy_dir: "/home/tidb/tidb-deploy/tidb-4011"
    log_dir: "/home/tidb/tidb-deploy/tidb-4011/log"
    numa_node: "0"
  - host: 192.168.70.48
    port: 4012
    status_port: 17081
    numa_node: "1"
  - host: 192.168.70.50
    port: 4011
    status_port: 17080
    deploy_dir: "/home/tidb/tidb-deploy/tidb-4011"
    log_dir: "/home/tidb/tidb-deploy/tidb-4011/log"
    numa_node: "0"
  - host: 192.168.70.50
    port: 4012
    status_port: 17081
    numa_node: "1"
  - host: 192.168.70.51
    port: 4011
    status_port: 17080
    deploy_dir: "/home/tidb/tidb-deploy/tidb-4011"
    log_dir: "/home/tidb/tidb-deploy/tidb-4011/log"
    numa_node: "0"
  - host: 192.168.70.51
    port: 4012
    status_port: 17081
    numa_node: "1"

tikv_servers:
  - host: 192.168.70.48
    port: 27160
    status_port: 27180
    deploy_dir: "/home/tidb/tidb-deploy/tikv-27160"
    data_dir: "/home/tidb/tidb-data/tikv-27160"
    log_dir: "/home/tidb/tidb-deploy/tikv-27160/log"
    numa_node: "0"
    config:
      server.labels: { host: "tikv1" }
      readpool.unified.max-thread-count: 25
      storage.block-cache.capacity: "50GiB"
      raftstore.capacity: "2048GiB"
  - host: 192.168.70.48
    port: 27161
    status_port: 27181
    numa_node: "1"
    config:
      server.labels: { host: "tikv1" }
      readpool.unified.max-thread-count: 25
      storage.block-cache.capacity: "50GiB"
      raftstore.capacity: "2048GiB"

  - host: 192.168.70.50
    port: 27160
    status_port: 27180
    deploy_dir: "/home/tidb/tidb-deploy/tikv-27160"
    data_dir: "/home/tidb/tidb-data/tikv-27160"
    log_dir: "/home/tidb/tidb-deploy/tikv-27160/log"
    numa_node: "0"
    config:
      server.labels: { host: "tikv2" }
      readpool.unified.max-thread-count: 25
      storage.block-cache.capacity: "30GiB"
      raftstore.capacity: "2048GiB"
  - host: 192.168.70.50
    port: 27161
    status_port: 27181
    numa_node: "1"
    config:
      server.labels: { host: "tikv2" }
      readpool.unified.max-thread-count: 25
      storage.block-cache.capacity: "30GiB"
      raftstore.capacity: "2048GiB"

  - host: 192.168.70.51
    port: 27160
    status_port: 27180
    deploy_dir: "/home/tidb/tidb-deploy/tikv-27160"
    data_dir: "/home/tidb/tidb-data/tikv-27160"
    log_dir: "/home/tidb/tidb-deploy/tikv-27160/log"
    numa_node: "0"
    config:
      server.labels: { host: "tikv3" }
      readpool.unified.max-thread-count: 25
      storage.block-cache.capacity: "30GiB"
      raftstore.capacity: "2048GiB"
  - host: 192.168.70.51
    port: 27161
    status_port: 27181
    numa_node: "1"
    config:
      server.labels: { host: "tikv3" }
      readpool.unified.max-thread-count: 25
      storage.block-cache.capacity: "30GiB"
      raftstore.capacity: "2048GiB"

cdc_servers:
  - host: 192.168.70.50
    port: 8500
    gc-ttl: 88400
    #ticdc_cluster_id: "cluster0"
  - host: 192.168.70.51
    port: 8500
    gc-ttl: 88400
    #ticdc_cluster_id: "cluster0"

monitoring_servers:
  - host: 192.168.70.48
    ssh_port: 22
    port: 9790
    ng_port: 14020
    # deploy_dir: "/tidb-deploy/prometheus-8249"
    # data_dir: "/tidb-data/prometheus-8249"
    # log_dir: "/tidb-deploy/prometheus-8249/log"

grafana_servers:
  - host: 192.168.70.48
    port: 3700
    # deploy_dir: /tidb-deploy/grafana-3000

alertmanager_servers:
  - host: 192.168.70.48
    ssh_port: 22
    web_port: 9793
    cluster_port: 9794
    # deploy_dir: "/tidb-deploy/alertmanager-9093"
    # data_dir: "/tidb-data/alertmanager-9093"
    # log_dir: "/tidb-deploy/alertmanager-9093/log"
```

检查并修复集群配置

```bash
tiup cluster deploy tidb-cluster v6.5.2 ./topology.yaml --user root -p
```

部署集群

```bash
tiup cluster deploy tidb-cluster v6.5.2 ./topology.yaml --user root -p
```

检查集群状态

```bash
tiup cluster display tidb-cluster
```

开启必要端口并初始化集群

```bash
# redhat
# tiup cluster display tidb-cluster 查看需要打开的端口
# 可用 GPT 读取集群配置文件，写脚本开启所有端口
sudo firewall-cmd --add-port=port/tcp --permanent
sudo firewall-cmd --reload

tiup cluster start tidb-cluster --init
#Started cluster `tidb-cluster` successfully
#The root password of TiDB database has been changed.
#The new password is: 'passwd'.
#Copy and record it to somewhere safe, it is only displayed once, and will not be stored.
```

连接 TIDB 数据库并修改密码

```bash
mysql -h 127.0.0.1 -P 4011 -u root -p'passwd'

> SET PASSWORD FOR 'root'@'%' = 'xxx';
```

