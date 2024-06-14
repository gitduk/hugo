+++
title = "TiDB 单机部署"
date = "2023-10-23"
tags = ["TiDB"]
+++


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
# 拓扑文件配置: https://docs.pingcap.com/zh/tidb/stable/tiup-cluster-topology-reference
# tidb 配置详情: https://docs.pingcap.com/zh/tidb/stable/tidb-configuration-file
# tikv 配置详情: https://docs.pingcap.com/zh/tidb/stable/tikv-configuration-file
# pd 配置详情: https://docs.pingcap.com/zh/tidb/stable/pd-configuration-file

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
    instance.tidb_slow_log_threshold: 1000

    # TiDB 单行数据的大小限制, 默认值：6291456
    # 最大值不超过 125829120（表示 120MB）
    performance.txn-entry-size-limit: 125829120

    # TiDB 单个事务大小限制, 默认值：104857600
    # 最大值不超过 1099511627776（表示 1TB）
    performance.txn-total-size-limit: 1099511627776

  pd:
    replication.enable-placement-rules: true
    replication.max-replicas: 1
    replication.location-labels: ["host"]

    # 同时进行 leader 调度的任务个数, 默认值：4
    # 如果集群的 TiKV 节点有足够资源, 可设置为 CPU 核数的 1/4 左右
    schedule.leader-schedule-limit: 4

    # 同时进行 Region 调度的任务个数, 默认值：2048
    schedule.region-schedule-limit: 2048

    # 同时进行 replica 调度的任务个数, 默认值：64
    schedule.replica-schedule-limit: 64

  tikv:
    # 建议为cpu核数的1/4
    server.grpc-concurrency: 8

    # MEM_TOTAL * 0.5 / TiKV 实例数量
    storage.block-cache.capacity: "3G"

    # readpool
    readpool.storage.use-unified-pool: true
    readpool.coprocessor.use-unified-pool: true

    # 处理高优先级读请求的线程池线程数量
    # 当 8 ≤ cpu num ≤ 16 时，默认值为 cpu_num * 0.5；
    # 当 cpu num 小于 8 时，默认值为 4；
    # 当 cpu num 大于 16 时，默认值为 8
    readpool.storage.high-concurrency: 8

    # cores * 0.8 / TiKV 数量
    readpool.unified.max-thread-count: 13

    # 单个日志最大大小，硬限制, 默认值：8MB
    raftstore.raft-entry-max-size: "128MB"

    # 磁盘总容量 / TiKV 实例数量
    raftstore.capacity: "80GB"

    # duration	副本允许的最长未响应时间, 默认值: "10m"
    raftstore.max-peer-down-duration: "1m"

    # 单个 log 文件最大大小，超过设定的参数值后，系统自动切分成多个文件, 默认值：300，单位 M
    log.file.max-size: 300

    # 保留 log 文件的最长天数, 默认值：0
    log.file.max-days: 7

tidb_servers:
  - host: 127.0.0.1
    port: 4000
    status_port: 10080
    numa_node: "0"

  - host: 127.0.0.1
    port: 4001
    status_port: 10081
    numa_node: "0"

  - host: 127.0.0.1
    port: 4002
    status_port: 10082
    numa_node: "0"

pd_servers:
  - host: 127.0.0.1
    client_port: 2379
    peer_port: 2380

  - host: 127.0.0.1
    client_port: 2381
    peer_port: 2382

  - host: 127.0.0.1
    client_port: 2383
    peer_port: 2384

tikv_servers:
  - host: 127.0.0.1
    port: 20160
    status_port: 20180
    numa_node: "1"
    config:
      server.labels: { host: "tikv-01" }

  - host: 127.0.0.1
    port: 20161
    status_port: 20181
    numa_node: "1"
    config:
      server.labels: { host: "tikv-02" }

  - host: 127.0.0.1
    port: 20162
    status_port: 20182
    numa_node: "1"
    config:
      server.labels: { host: "tikv-03" }

tiflash_servers:
  - host: 127.0.0.1

monitoring_servers:
  - host: 127.0.0.1
    port: 9090

grafana_servers:
  - host: 127.0.0.1
    port: 3000


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

**手动修复集群存在的潜在风险**

```txt
127.0.0.1  disk          Fail    mount point /home/tidb does not have 'nodelalloc' option set
```


临时修复（重启失效）：
```bash
sudo umount /home/tidb
sudo mount -o nodelalloc /dev/sdb1 /home/tidb
```


永久修复，编辑 `/etc/fstab` 文件，在设备类型后面添加 `nodelalloc` 选项（重启生效）：
```
UUID=your-uuid /home/tidb    ext4,nodelalloc    defaults        0 0
```

验证：
```bash
mount | grep "/home/tidb"

/dev/sdb1 on /home/tidb type ext4 (rw,relatime,nodelalloc,stripe=256)
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

