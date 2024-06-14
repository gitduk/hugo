+++
title = "frp + nginx 实现不同子域名访问同一主机不同服务"
date = "2023-07-04"
tags = ["frp", "nginx"]

+++



## 前置条件

- 运行在内网的多个服务
- 一台带有公网 IP 的云服务器（如果是国内服务器，域名需要备案）
- 一个域名




## 域名解析


到 cloudflare 解析配置你的域名解析

![image-20230629164727654](/img/image-20230629164727654.png)



## Frps, Nginx 服务配置

frps 部署 - https://github.com/fatedier/frp

Nginx 安装 - `sudo apt install nginx`



frps 配置文件

```t
[common]
bind_port = 7000
bind_addr = 0.0.0.0
vhost_http_port = 7080
token = [custom token]

dashboard_port = 7050
dashboard_user = admin
dashboard_pwd = changeme

subdomain_host = wukaige.com
```



nginx 配置 <u>/etc/nginx/conf.d/frp.conf</u>

```nginx
server {
        listen 80;
        server_name alist.wukaige.com;
        location / {
                proxy_pass http://127.0.0.1:7080;
                proxy_set_header   Host             $host;
                proxy_set_header   X-Real-IP        $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}
```



重启 frps 和 nginx 服务

```bash
sudo systemctl restart nginx.service

# 以服务方式安装的 frps
sudo systemctl restart frps.service
```



## Frpc 配置

frpc 配置文件

```text
[common]
server_addr = [ip to frps server]
server_port = 7000
token = [custom token]

[alist]
type = http
local_port = 5244
subdomain = alist
```



重启 frpc 服务

```
# 以服务方式安装的 frpc
sudo systemctl restart frpc.service
```

没有例外的话，访问 `alist.wukaige.com` 即可访问本地 alist 服务。
有例外的话，看看服务器和本地防火墙，是否开放需要开放的端口。
