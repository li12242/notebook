---
layout: post
title: Singularity 容器介绍
date: 2022-9-9
categories: tools
---

## 1. 简介

singularity 是与 docker 类似的容器工具。所谓容器指的是一个标准的软件单元，它打包代码及其所有依赖项，以便应用程序从一个计算环境快速可靠地运行到另一个计算环境。

使用容器的优点包括：

1. 容器简化了软件安装和管理；
2. 可以在一台机器上构建映像并在另一台机器上运行它；
3. 容器的版本控制和冻结使数据具有可重复性。

singularity 容器的优点包括：

* 不需要 sudo 权限即可运行（与 Docker 不同）；
* 在多节点环境中与 HPC 资源管理器很好地互操作；
* 默认情况下，轻松利用集群或服务器上的 GPU、高速网络、并行文件系统；
* 能够将 Docker 镜像转换为 Singularity。

singularity 容器可以从网络下载，储存为 SIF 文件。
```bash
$ singularity pull library://godlovedc/demo/lolcow
INFO:    Downloading library image
 89.24 MiB / 89.24 MiB [==============================================================] 100.00% 60.03 MiB/s 1s
WARNING: Container might not be trusted; run 'singularity verify lolcow_latest.sif' to show who signed it
INFO:    Download complete: lolcow_latest.sif
```

运行时直接使用 SIF 直接运行
```
$ singularity run lolcow_latest.sif
 ___________________________________
< Beware of low-flying butterflies. >
 -----------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

## 2. 安装配置

对于 singularity 安装和配置方法，可以参考官网说明[^1]。

### 2.1. 环境配置

对于支持 `apt-get` 和 `yum` 命令环境，分别使用如下命令安装底层环境。

```bash
$ sudo apt-get update && sudo apt-get install -y \
    build-essential \
    libssl-dev \
    uuid-dev \
    libgpgme11-dev \
    squashfs-tools \
    libseccomp-dev \
    pkg-config
```

```bash
$ sudo yum update -y && \
    sudo yum groupinstall -y 'Development Tools' && \
    sudo yum install -y \
    openssl-devel \
    libuuid-devel \
    libseccomp-devel \
    wget \
    squashfs-tools
```

[^1]: https://docs.sylabs.io/guides/3.0/user-guide/installation.html

### 2.2. go 语言配置

下载 go 语言安装包，并将其解压

```bash
$ export VERSION=1.11 OS=linux ARCH=amd64 && \
    wget https://dl.google.com/go/go$VERSION.$OS-$ARCH.tar.gz && \
    sudo tar -C /usr/local -xzvf go$VERSION.$OS-$ARCH.tar.gz && \
    rm go$VERSION.$OS-$ARCH.tar.gz
```

设置 go 语言相关环境变量

```bash
$ echo 'export GOPATH=${HOME}/go' >> ~/.bashrc && \
    echo 'export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin' >> ~/.bashrc && \
    source ~/.bashrc
```

### 2.3. singularity 编译安装

在 github 官网下载最新版 singularity 源代码。

```bash
export VERSION=3.0.3 && # adjust this as necessary \
    mkdir -p $GOPATH/src/github.com/sylabs && \
    cd $GOPATH/src/github.com/sylabs && \
    wget https://github.com/sylabs/singularity/releases/download/v${VERSION}/singularity-${VERSION}.tar.gz && \
    tar -xzf singularity-${VERSION}.tar.gz && \
    cd ./singularity
```

采用如下命令对 singularity 源代码进行编译安装

```bash
$ ./mconfig && \
    make -C ./builddir && \
    sudo make -C ./builddir install
```

当需要将软件安装到特定目录时，configure 过程增加 `--prefix` 参数。

```bash
$ ./mconfig --prefix=/opt/singularity
```

## 3. 容器构建

容器构建使用 `build` 命令，可以从 Container 和 Docker Hub 等外部资源下载和组装现有容器。
构建容器主要方法有多种：

* 以library://开头的 URI，用于从 Container Library 构建；
* 以docker://开头的 URI，用于从 Docker Hub 构建；
* URI 以shub://开头，从 Singularity Hub 构建；
* 本地计算机上现有容器的路径；
* 要从沙箱构建的目录的路径；
* singularity 定义文件的路径；

`build` 可以生成两种不同格式的容器，如下：
* 只读的 singularity sif 格式文件（默认）；
* 交互式开发沙箱，具有可写的根目录；
  
`build` 可以接受现有容器为目标，并将其转换为另一种形式。
对于复杂容器的构建，建议使用沙箱模式在容器内进行配置，完成后转换为 SIF 文件格式。

### 3.1. 容器构建示例

下面将以示例对不同构建方法进行介绍。

**从容器库下载**

```bash
$ sudo singularity build lolcow.simg library://sylabs-jms/testing/lolcow
```

**从Docker Hub下载**

```bash
$ sudo singularity build lolcow.sif docker://godlovedc/lolcow
```

**创建沙箱目录**

```bash
$ sudo singularity build --sandbox lolcow/ library://sylabs-jms/testing/lolcow
```
生成的目录可以像 SIF 文件中的容器一样运行。
```bash
$ sudo singularity shell --writable lolcow/
```

**容器格式转换**

如果在本地保存了一个容器，可以将其用作构建新容器的目标。例如，有一个沙箱容器 `development/`，可以使用如下命令转换为 SIF 容器。
```bash
$ sudo singularity build production.sif development/
```

**从定义文件构建容器**

singularity 也可以采用定义文件作为构建容器目标，这种方法与 Dockfile 形式类似。
定义文件示例如下，在后面定义文件中会详细介绍定义文件编写规则。

```bash
Bootstrap: docker
From: ubuntu:16.04

%post
    apt-get -y update
    apt-get -y install fortune cowsay lolcat

%environment
    export LC_ALL=C
    export PATH=/usr/games:$PATH

%runscript
    fortune | cowsay | lolcat
```

使用定义文件构建容器命令如下，必须使用 root 账户运行。
```bash
$ sudo singularity build lolcow.sif lolcow.def
```

### 3.2. 自定义文件

自定义文件中包含如下内容用于构建 singularity 容器的命令：

* 基本操作系统或基本容器
* 要从主机系统添加的文件
* 要安装的软件
* 在运行时设置的环境变量
* 容器元数据

基本自定义文件如下

```
Bootstrap: ...   # "Header"
From: ...        #

%files           # "Section"
    ...          #

%post
    ...

%environment
    ...

%runscript
    ...

%labels
    ...

%help
    ...
```

**文件标题**

标题在文件顶部，并设置基本操作系统或基本容器。
Bootstrap 关键字始终是必需的。它描述了要使用的引导模块，而 `From` 关键字指定制作容器的源目录或 URL。

* `library` 使用 Container Library 中的现有容器作为“基础”；
* `docker` 使用 Docker 层创建基础映像；
* `localimage` 从主机系统上的现有 Singularity 容器构建容器；
 
**文件主体**

在配置文件主体部分，每个部分都以 `%` 开始，各个部分都是可选的。

* `%files` 内容将文件复制到容器中；
* `%post` 指定容器构建时的安装命令；
* `%environment` 定义在运行时环境变量，但是在构建时不可用；
* `%runscript` 使用 `singularity run` 指定的命令列表；
* `%labels` 以键值对的形式添加元数据；
* `%help` 显示的帮助文本；

## 4. 交互式运行

构建好容器后，可以通过如下形式以交互模式运行。

```bash
$ singularity shell lolcow_latest.sif
Singularity>
```

使用 `shell` 模式可以进入容器内运行。在交互式运行时，可以在容器内访问以下目录：

* 个人目录 `/home`
* 系统目录 `/tmp`，`/sys`，`/proc`，`/dev`，`/usr`
* 当前的工作目录

singularity 运行时会自动挂载这些目录，容器内其他目录归 root 账户所有，无法进行修改。
如果需要运行时挂载其他目录或文件，可以使用 `--bind` 或 `-B` 参数。

## 5. 执行自定义命令

使用 `exec` 命令可以运行自定义命令而无需进入容器：

```bash
$ singularity exec <SIF> <COMMAND>
```

可以将命令写入脚本中，然后通过 `exec` 命令运行，例如：

```bash
$ cat my_bash_script.sh
#!/bin/bash
for i in {1..5}; do
    fortune
done

$ singularity exec lolcow_latest.sif /bin/bash my_bash_script.sh
```
