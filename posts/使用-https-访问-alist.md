+++
title = "使用 https 访问 alist"
date = "2023-07-17"
tags = ["nginx", "alist"]

+++



通过 http 访问 alist，地址栏那个感叹号看起来有点丑。




## 申请 Cloudflare 证书

[看这里](https://www.google.com.hk/search?q=%E7%94%B3%E8%AF%B7+Cloudflare+%E8%AF%81%E4%B9%A6&oq=%E7%94%B3%E8%AF%B7+Cloudflare+%E8%AF%81%E4%B9%A6&aqs=chrome..69i57.630j0j7&sourceid=chrome&ie=UTF-8)

把 Origin Certificate 保存为： your-domain.pem

把 Private Key 保存为： your-domain.key



## 配置 nginx 转发规则

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name alist.wukaige.com;

    ssl_certificate /path/to/your-domain.pem;
    ssl_certificate_key /path/to/your-domain.key;

    if ($scheme != "https") {
        return 301 https://$server_name$request_uri;
    }

    client_max_body_size 10000m;

    location / {
        proxy_pass http://127.0.0.1:7080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```



## 重启 nginx 服务

```bash
sudo systemctl restart nginx.service
```



