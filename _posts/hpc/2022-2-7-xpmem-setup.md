---
layout: post
title: XPMEM安装与部署
date: 2022-2-7
categories: hpc
---

# xpmem 安装与部署

## 编译与安装

参考 MVACH2 中对 xpmem 安装过程，首先从 github 下载 xpmem 源代码

```bash
git clone https://gitlab.com/hjelmn/xpmem.git
```

在安装过程中，指定 xpmem 安装目录为 /opt/xpmem。此时对应的安装命令为 

```bash
cd xpmem
./autogen.sh
./configure --prefix=/opt/xpmem
make -j8 install
```

安装完成后，需要在系统中加载 xpmem 模块。以安装目录 /opt/xpmem 为例，通过 insmod 安装对应模块

```bash
insmod /opt/xpmem/lib/modules/4.18.0-193.el8.x86_64/kernel/xpmem/xpmem.ko
chmod 666 /dev/xpmem
```

加载完成后，可以通过 lsmod 命令检查

```bash
lsmod | grep xpmem
xpmem                  45056  0
```

## 参考

1. [Configuring UCX with XPMEM](https://docs.mellanox.com/display/HPCXv27/Unified+Communication+-+X+Framework+Library#UnifiedCommunicationXFrameworkLibrary-ConfiguringUCXwithXPMEM)
2. [Installation of MVAPICH2, MVAPICH2-X and MVAPICH2-GDR with Spack](http://mvapich.cse.ohio-state.edu/userguide/userguide_spack/#_install_xpmem)
