---
title: linux 双网卡做 bond0
date: 2023-07-11 17:55:28
tags:
- linux
---



> 为了提供网络的高可用性，我们可能需要将多块网卡绑定成一块虚拟网卡对外提供服务，这样即使其中的一块物理网卡出现故障，也不会导致连接中断。



## 前情提要

服务器配置了两张网卡，现在需要做 bond0。

网关: 192.168.0.1
网卡一: eno8303 (192.168.0.2)
网卡二: eno8403 (192.168.0.3)



## 创建 bond0

```bash
sudo nmcli con add type bond con-name bond0 ifname bond0 mode 802.3ad miimon 100 updelay 200 downdelay 200
```



## 绑定网卡到 bond0

```bash
sudo nmcli con add type ethernet con-name bond0-slave1 ifname eno8303 master bond0
sudo nmcli con add type ethernet con-name bond0-slave2 ifname eno8403 master bond0
```



## 修改 bond0 配置

```bash
sudo nmcli con mod bond0 ipv4.addresses "192.168.0.100/24"
sudo nmcli con mod bond0 ipv4.gateway "192.168.0.1"
sudo nmcli con mod bond0 ipv4.dns "223.5.5.5"
sudo nmcli con mod bond0 ipv4.method manual
sudo nmcli con mod bond0 connection.autoconnect yes
sudo nmcli con up bond0
sudo nmcli con up bond0-slave1
sudo nmcli con up bond0-slave2

# 删除 bond0
# sudo nmcli con delete bond0
```



## 修改系统 route

```bash
sudo ip route add default via 192.168.0.1 dev bond0 proto static metric 300
```



