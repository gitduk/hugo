+++
title = "redhat 安装显卡驱动"
date = "2023-10-13"
tags = ["redhat", "nvidia"]
+++

公司购买了一张 4090 显卡，需要安装到服务器上。
服务器 OS 版本：redhat9

#### 安装前的配置工作


重启到文本模式

```bash
sudo systemctl set-default multi-user.target
reboot
```


查看显卡是否被识别

```bash
lspci | egrep 'VGA|3D'
```

禁用 nouveau
```bash
echo "
blacklist nouveau
blacklist lbm-nouveau
alias nouveau off
alias lbm-nouveau off
options nouveau modeset=0
" | sudo tee /etc/modprobe.d/blacklist-nvidia-nouveau.conf
sudo dracut --force
```

#### 开始安装

更新储存库缓存

```bash
sudo dnf makecache
```

启用 official RHEL 9 CodeReady Builder package 储存库

```bash
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(uname -i)-rpms
```

安装 epel-release

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf makecache
```

安装编译工具和依赖库

```bash
sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) gcc make dkms acpid libglvnd-glx libglvnd-opengl libglvnd-devel pkgconfig
```

添加 NVIDIA 官方的储存库

```bash
sudo dnf config-manager --add-repo http://developer.download.nvidia.com/compute/cuda/repos/rhel9/$(uname -i)/cuda-rhel9.repo
sudo dnf makecache
```

安装最新的显卡驱动

```bash
sudo dnf module install nvidia-driver:latest-dkms
```

验证安装

```bash
nvidia-smi
```


安装完成后，需重启系统。

#### 参考链接

- https://linuxhint.com/install-nvidia-drivers-rhel-9/

