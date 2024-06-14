+++
title = "Ubuntu 显卡配置"
date = "2023-06-29"
tags = ["linux", "nvidia"]

+++



## 系统环境

系统: Ubuntu 22.04
内核: 5.19.0-32-generic
显卡: AD102 [GeForce RTX 4090]

 

## 前期准备

```
# 查看显卡设备
lshw -numeric -C display

# 查看显卡
lsmod | grep -E "nvidia|nouveau"

# 添加显卡 ppa
sudo add-apt-repository ppa:graphics-drivers
sudo apt update

# 禁用 nouveau
echo "
blacklist nouveau
blacklist lbm-nouve
alias nouveau off
alias lbm-nouveau off

options nouveau modeset=0

# for sway
options nvidia-drm modeset=1

" | sudo tee /etc/modprobe.d/blacklist-nvidia-nouveau.conf
sudo update-initramfs -u

# 禁用 nvidia, enable nouveau
# echo "
# blacklist nvidia
# options nvidia modeset=0
# " | sudo tee /etc/modprobe.d/blacklist-nvidia-nouveau.conf
# sudo update-initramfs -u

# 查看推荐的驱动版本, 到官网下载对应的显卡驱动 <https://www.nvidia.com/Download/index.aspx?lang=en-us>
ubuntu-drivers devices

# 文本模式
sudo systemctl set-default multi-user.target
reboot
```



## 重启进入命令行模式

```bash
# 安装显卡驱动
sudo dpkg -i xxx.deb

# 图形模式
sudo systemctl set-default multi-user.target
reboot

# 查看安装
nvidia-smi
```



显卡型号查询地址:
http://pci-ids.ucw.cz/mods/PC/10de/2684



## CUDA 安装

先查看显卡驱动支持的 CUDA 版本
https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html

再进入官网下载安装
https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_local



## CUDNN 安装

解压[下载](https://developer.nvidia.com/rdp/cudnn-archive)的 CUDNN 文件: cudnn-linux-x86_64-8.8.0.121_cuda11-archive.tar.xz

```bash
cd cudnn-linux-x86_64-8.8.0.121_cuda11-archive/
sudo cp include/cudnn*.h /usr/local/cuda/include
sudo cp lib64/libcudnn* /usr/local/cuda/lib64

sudo chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*
```



## 参考链接

- https://cloud.tencent.com/developer/article/2000757
- https://blog.csdn.net/Zhou_Dao/article/details/103703289
- https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html
- https://towardsdatascience.com/installing-multiple-cuda-cudnn-versions-in-ubuntu-fcb6aa5194e2
