---
title: 导入任何Linux系统容器镜像到wsl
p: others/wsl_import_container_image.md
date: 2024-02-29 10:20:41
tags: [Others,windows,wsl]
categories: [Others,windows,wsl]
---

# 导入任何Linux系统容器镜像到wsl

除了Microsoft Store中含有的Linux子系统镜像、Microsoft官方提供的Linux子系统镜像<https://learn.microsoft.com/en-us/windows/wsl/install-manual>以及他人制作的Linux子系统镜像可以创建为Wsl子系统外，WSL还支持将任意Linux容器镜像导入为Wsl子系统，这对于想要拥有某个特定发行版、特定版本的子系统的人说是一个福音。

以Ubuntu24.04为例，由于Ubuntu24.04刚发行不久，此时Microsoft官方还没有提供该发行版，但DockerHub上已经存在Ubuntu24.04的容器镜像。按照下面的步骤，可以完成Ubuntu2404子系统的创建。

前提条件：

- 已经拥有一个docker(或者podman)并已启动
- windows上已经安装好wsl

步骤1：通过docker拉取ubuntu2404容器镜像，并将其导出为tar归档文件(ubuntu2404.tar)

```shell
sudo docker pull ubuntu:24.04
sudo docker run ubuntu:24.04
container_id=$(sudo docker container ls -a | grep -i 'ubuntu:24.04' | awk '{print $1}')
sudo docker export $container_id > ubuntu2404.tar
```

步骤2：将导出的ubuntu2404.tar文件复制到windows上，假设存放路径为`V:\wsl\ubuntu2404.tar`  

步骤3：Windows上打开PowerShell，将ubuntu2404.tar文件导入到wsl

```shell
# 假设想要将导入后创建的Ubuntu2404子系统保存在目录V:\wsl\Ubuntu24-04_v1中
# 并且该子系统的名称为Ubuntu2404
wsl --import Ubuntu2404 V:\wsl\Ubuntu24-04_v1 V:\wsl\ubuntu2404.tar
```

现在，Ubuntu2404子系统已经创建好了(可通过`wsl -l -v`命令查看)，已经可以进入该子系统并设置该子系统，比如设置该子系统的默认登录用户。

```shell
# 登录子系统
wsl -d Ubuntu2404

# 进入子系统后，创建新用户(用户名longshuai，uid/gid为1200)并修改密码，稍后设置为该子系统的默认登录用户
# 顺便把root用户的密码也修改
root@DESKTOP:$ useradd -u 1200 -s /bin/bash -m -d /home/longshuai longshuai
root@DESKTOP:$ passwd
root@DESKTOP:$ passwd longshuai
# 如果通过下面的方式设置默认的登录用户无效，可使用工具LxRunOffline来设置
root@DESKTOP:$ echo -e "[user]\ndefault=longshuai" >> /etc/wsl.conf
root@DESKTOP:$ exit
```

