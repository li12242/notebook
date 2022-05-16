---
layout: post
title: Docker建立Linux开发环境
date: 2022-5-16
categories: tools
---

# 背景

在 Windows 和 Mac 环境中，由于缺少 Linux 环境，无法进行 Linux CPP 项目开发，需要登录远程或使用虚拟机，具有诸多不便。使用 Docker 可以在本地建立一个虚拟的 Linux CPP 开发环境，并且占用资源较少，很好满足需求。

# Docker 镜像

使用 Dockerfile 指定镜像安装过程。由于在本地开发过程中使用 Clion，需要使用 cmake 和 rsync 等工具，设置 Dockerfile 如下：

```
FROM ubuntu:18.04

LABEL maintainer=lilongxiang01@inspur.com

RUN apt-get clean
RUN apt-get update
RUN apt-get install -y ssh \
    build-essential \
    gcc \
    g++ \
    gdb \
    clang \
    cmake \
    rsync \
    tar \
    python \
    openssh-server
RUN apt-get clean
```

使用下列命令建立 Docker 镜像，配置镜像名称为 teye/ubuntu。

```bash
$ docker build -t teye/ubuntu -f Dockerfile .
```

# Docker 容器

使用如下命令启动容器，并挂载到后台。

```bash
docker run -idt --name teye_dev -p 8022:22 --privileged=true teye/ubuntu /sbin/init
```

其中运行 `/sbin/init` 可以保证 systemctl 命令正常运行，否则可能出现如下错误：

```bash
$ systemctl
System has not been booted with systemd as init system (PID 1). Can't operate.
```

启动容器后，使用如下命令进入容器

```bash
docker exec -it teye_dev /bin/bash
```

进入容器后，修改 root 账户密码

```bash
$ passwd
```

并配置启动 sshd 服务，保证能够远程登录

```bash
$ vim /etc/ssh/sshd_config
...
UsePAM no
PermitRootLogin yes
PasswordAuthentication yes

$ systemctl start sshd
```

# 远程登录

在本地命令行，使用如下命令尝试远程登录。由于启动容器时指定了端口为 8022，登录时也需要指定。

```bash
ssh root@localhost -p 8022
The authenticity of host '[localhost]:8022 ([::1]:8022)' can't be established.
ECDSA key fingerprint is SHA256:H4Pqa96Wm+wWk2LGd83LyFV7VujclW94qeEDw+RR5xo.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:8022' (ECDSA) to the list of known hosts.
root@localhost's password:
```

