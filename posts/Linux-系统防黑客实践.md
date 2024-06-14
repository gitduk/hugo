+++
title = 'Linux 系统防黑客实践'
date = 2024-04-01T11:44:02+08:00
draft = false
+++



> 服务器系统为 Redhat 9
>
> 本地机器为 Ubuntu 23.04

## 修改默认的 ssh 端口

修改配置文件 */etc/ssh/sshd_config* 的 `Port` 配置

```txt
Port 2222
```



## ssh 用密钥登录

**在服务器端配置**

修改配置文件 */etc/ssh/sshd_config* 配置

```
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys
```

重启 `sshd` 服务

```bash
sudo systemctl restart sshd
```



**在本地机器配置**

运行命令生成私钥和公钥

```bash
ssh-keygen -t rsa
```

一直按回车，完成后 ~/.ssh 目录会生成 `id_rsa` 和 `id_rsa.pub` 两个文件。



Copy 公钥到服务器

```bash
# 需要输入用户 user 的密码
ssh-copy-id -p 2222 -i ~/.ssh/id_rsa.pub user@host
```



登录测试，如果无须输入密码即可登录服务器，配置成功。

```bash
ssh -p 2222 user@host
```



如果还需要输入密码，可按照下面几个办法进行排查



**客户端排查**

```bash
# 查看 ssh 登录时-p 2222 user@host的日志
ssh -vvv -p 2222 user@host
```

看看有什么可疑的日志



**服务器端排查**

```bash
# 查看系统安全日志
sudo /usr/bin/cat /var/log/secure

# 查看 sshd 服务日志
sudo systemctl status sshd
```



如果日志里面出现类似下面的日志

```log
Authentication refused: bad ownership or modes for directory /home/user
Authentication refused: bad ownership or modes for directory /home/user/.ssh
```



可用下面的命令解决

```bash
chmod go-w /home/user
chmod 700 /home/user/.ssh
chmod 600 /home/user/.ssh/authorized_keys
```



## fail2ban 防止暴力破解

安装

```bash
sudo apt install fail2ban
```



配置

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo vim /etc/fail2ban/jail.local
```



*/etc/fail2ban/jail.local*

```txt 
[sshd]
# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode   = normal
enabled = true
port    = 22
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
```

- `logpath = %(sshd_log)s` 指定 SSH 服务的日志文件路径,这里使用了变量,真实路径在其他地方定义。

- `backend = %(sshd_backend)s` 同上,指定了该 jail 使用的后端,但具体内容在别处定义为变量。



重启服务应用配置

```bash
sudo systemctl restart fail2ban
```



查看状态

```bash
# 列出所有被封禁的 IP 及其原因
sudo fail2ban-client status
sudo fail2ban-client status sshd

# 查看 fail2ban 日志文件
sudo tail -n 100 /var/log/fail2ban.log
```

