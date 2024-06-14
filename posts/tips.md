+++
title = 'Tips'
date = 2024-05-09T15:11:12+08:00
draft = true

+++



```bash
sudo firewall-cmd --permanent --zone=public --add-port=5001/tcp
sudo firewall-cmd --permanent --zone=public --add-port=5001/tcp
```

```bash
curl -X DELETE "http://localhost:9200/index_name"
```

```bash
while read -r row; do
  echo $row
done < $file
```

```bash
sudo usermod -aG group user
```

```bash
sudo chmod 700 /home/nest/.ssh
sudo chmod 600 /home/nest/.ssh/authorized_keys
sudo chmod go-w /home/nest/
```

```bash
# 修改用户默认登录 SHELL
sudo usermod -s /usr/bin/zsh user
```

