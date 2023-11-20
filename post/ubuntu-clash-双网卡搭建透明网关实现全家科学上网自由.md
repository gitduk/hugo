---
title: ubuntu + clash + 双网卡搭建透明网关实现全家科学上网自由
date: 2023-08-10 14:49:26
tags:
- clash
---







## 前情提要

手机科学上网需要打开 clash 软件，手动开启代理，非常不爽，我想搭建一个无感科学上网环境，连上 WIFI 就能科学上网。



## 硬件配置

- 台式主机，连接网线 enp3s0
- 一个 USB 无线网卡 wlan0



## 软件环境

- Ubuntu 22.04
- clash
- nftables



## 配置网卡

配置思路：enp3s0 连接有线网，wlan0 开启热点，供手机连接。

使用 nmcli 命令，在 wlan0 上创建热点。

```bash
nmcli dev wifi hotspot ifname wlan0 con-name hotspot ssid [WIFI 名称] password '12345678'
```

开启热点

```bash
nmcli con up hotspot
```

查看设备上的网络连接

```bash
nmcli dev
```

一切正常的话，可以看到以下输出：

![image-20230810153406140](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230810153406140.png)

手机用密码 `12345678` 连接创建的 WIFI，此时可以正常上网，但是不可共享台式机的 clash 代理。

## 开启 Ubuntu ip 转发

把文件 `/etc/sysctl.conf` 中 `net.ipv4.ip_forward=1` 的注释删除

![image-20230810151615005](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230810151615005.png)

加载并应用配置文件中的参数

```bash
sudo sysctl -p
```



## 配置 clash 重定向端口

安装 clash: https://github.com/Dreamacro/clash

clash 配置文件加上下面配置：

```conf
redir-port: 7892
```



## 配置 nftables 转发流量

安装 nftables

```bash
sudo apt install -y nftables
```



nftables 配置文件: <u>/etc/nftables.conf</u>

```conf
#!/usr/sbin/nft -f

flush ruleset

define private_list = {
    0.0.0.0/8,
    10.0.0.0/8,
    127.0.0.0/8,
    169.254.0.0/16,
    172.16.0.0/12,
    192.168.0.0/16,
    224.0.0.0/4,
    240.0.0.0/4
}

table inet filter {
    chain input {
        type filter hook input priority 0;
    }
    chain forward {
        type filter hook forward priority 0;
    }
    chain output {
        type filter hook output priority 0;
    }
}

table ip nat {
    chain proxy {
        ip daddr $private_list return
        ip protocol tcp redirect to :7892
    }
    chain prerouting {
        type nat hook prerouting priority 0; policy accept;
        jump proxy
    }
}
```

重启 nftables 以重载配置

```bash
sudo systemctl restart nftables.service
```



这些配置无误后，手机连上 WIFI，直接就可科学上网。
