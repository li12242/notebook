# docker 简明教程


## docker 基本概念

docker 是一个开源项目，诞生于 2013 年初，其目标是建立一个轻量级的操作系统虚拟化解决方案。
docker 以 linux 容器（LCX）技术为基础，并在此基础上进行了进一步的封装，使得操作更加简便。
与传统虚拟化方式不同的是，docker 在操作系统层面上实现虚拟化，直接复用了本地主机系统，而传统虚拟机则是在硬件层面实现。
因此与传统虚拟化方案相比，docker 有如下几点优势：

**更快**：容器的启动可以在秒级实现，这比传统虚拟机快的多。

**更简单**：docker 提供了简单方法快速创建容器，并且可以一次创建，在任意地方正常运行，对于开发和运维来说非常具有吸引力。

**更高效**：传统虚拟方案在运行多个应用时需要启动多个虚拟机，而 docker 只需要启动多个容器即可，此外 docker 对系统利用率很高，容器除了运行其中应用外，基本不消耗额外的系统资源，系统开销很小，因此单个主机上可以同时运行数千个容器。

docker 中，两个重要的概念是镜像和容器。
容器是从镜像创建的运行实例，实质是镜像和读写层的组合。
容器可以启动，开始，停止，删除，每个容器之间都是相互隔离的，从而保证了容器的安全性。
镜像是一个只读的模板，可以用来创建容器。
一般来说，镜像文件都是包含一个完整的操作系统环境和一些定制的应用。
在下面将具体介绍 docker 使用，并且以 texlive 环境搭建为例展示 docker 应用。

## docker 配置

这里主要介绍 centos 环境下 docker 配置过程。首先启动 docker 服务

```bash
$ systemctl start docker
```

此时输入 `docker ps` 命令，可能显示如下错误

```bash
$ docker ps
docker: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post http://%2Fvar%2Frun%2Fdocker.sock/v1.38/containers/create: dial unix /var/run/docker.sock: connect: permission denied.
```

这主要是因为 docker 进程使用 Unix Socket 而不是 TCP 端口。默认情况下，Unix socket 需要 root 权限才能访问。
为了保证普通用户也能使用，需要建立 docker 用户组，并且将用户添加入。
使用 root 账户添加 docker 用户组，并且添加当前用户。

```bash
$ sudo groupadd docker

$ sudo gpasswd -a <USER> docker
```

此时，需要在普通用户环境中手动更新用户组，随后再输入 docker 命令完成测试。

```bash
# 更新 docker 用户组
$ newgrp docker

# 测试 docker 命令
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

## 镜像

镜像是 docker 中一个只读的模板，在容器运行前需要本地存在对应的镜像，当镜像不存在时，docker 会从镜像仓库中下载。
下面介绍与镜像相关的一些操作命令。

### 获取镜像

docker 镜像安装一般有两种方法，第一种是直接从仓库中拖取镜像，第二种是使用 dockerfile 定制安装。
准确来说，第二种也是从镜像仓库获取的，只是此基础上进行了一些定制化操作。
第二种方法将在 dockerfile 小节中详细介绍。

对于直接拉取镜像安装方法，首先使用 `docker search ubuntu` 命令搜索镜像仓库中相关镜像，找到对应镜像名称后用 `pull` 命令拖取。

```bash
$ docker pull ubuntu:12.04
```

### 查看

在安装完镜像后，使用 `docker image ls` 或 `docker images` 可以查看本地已经安装好的镜像。

```bash
$ docker image ls
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
ubuntu                             14.04               f17b6a61de28        18 months ago       188MB
```

可以看到，在列出信息中有几个关键字段，分别是：

* REPOSITORY 仓库
* TAG 标记
* IMAGE ID 镜像 ID 号
* CREATED 创建时间
* SIZE 大小

其中，镜像的 ID 号唯一的标识了镜像，尽管有些镜像的标记号不同，但是当 ID 号相同时实际上是同一镜像。

### 删除

当不再需要镜像时，使用 `docker rmi` 命令，其中参数可以为镜像对应名字或者 ID。

```bash
# 通过镜像名删除镜像
$ docker rmi <image name>

# 通过镜像 ID 删除
$ docker rim <image id>
```

### 镜像修改

当本地有安装好的镜像时，可以在此基础上启动容器，并且进行进一步的修改。
在这里引入容器只是为了介绍保存镜像修改内容的方法，关于容器的具体使用会在后面介绍。

```bash
# 以交互方式启动容器
$ docker run -it ubunut /bin/bash

# 进入容器检查软件包更新
root@<tag>:# apt update

# 退出容器
root@<tag>:# exit
```

当退出后，修改后的容器并没有消失只是终止，可以再次启动。
用 `docker commit` 命令将更新后的容器制作成新的镜像，也可以替换原始镜像。

```bash
$ docker commit -m <comment> -a <user> <container id> <image name>
```

其中 `<image name>` 是新建镜像的名称，`<container id>` 是容器对应 ID，`-m` 提交说明信息，`-a` 指定更新用户信息。

### dockerfile

从上面可以看出，当需要一种特定容器时候，需要在镜像中启动容器，并在此基础上进行一系列操作，最终建立满足要求的镜像。
dockerfile 则提供了一种自动化方法来构建定制镜像。
一个简单的 dockerfile 格式如下。

```docker
# This is a comment
FROM ubuntu:14.04
LABEL maintainer=lilongxiang01@inspur.com
RUN apt-get -qq update
RUN apt-get -qqy install ruby
ADD test /root/
```

从上述文件中，出现的 dockerfile 相关语法为：
* `#` 为注释语句；
* `FROM` 告诉 docker 以哪个镜像为基础进行构建；
* `LABEL` 语句提供了维护者信息；
* `RUN` 指令会在创建过程运行；
* `ADD` 命令将本地文件复制到镜像中；

注意，在复制文件时，必须指定镜像中和当前文件夹同名的文件夹才能成功。也就是说，当 `test` 为一个文件时，上述 dockerfile 是没问题的，但是当 `test` 为文件夹时，则需要将目标路径改为 `/root/test`。

将上述文件命令为 `Dockerfile`，并放入目录 `ubuntu_ruby` 目录下。
随后使用 docker build 命令，根据 Dockerfile 构建镜像。

```bash
$ mkdir ubuntu_ruby
$ cd ubuntu_ruby
$ docker build -t ubuntu/ruty:v1 .
```

其中 `.` 为 Dockerfile 所在路径。也可以在目录上层进行构建，此时对应的路径为 `ubuntu_ruby`。

### 迁移

当需要把当前镜像作为一个文件进行保存时候，可以按照如下方式进行[^32432]。

```bash
# 将镜像保存成一个tar压缩包
$ docker save -o ubuntu_ruty.tar ubuntu/ruty
```

随后可将压缩包复制到另外一台主机中，并使用如下命令加载镜像。

```bash
$ docker load -i ubuntu_ruty.tar

# 检查镜像
$ docker images
```

[^32432]: http://jvm123.com/2019/07/docker-export.html

## 容器

### 查看

在 docker 中新建容器并运行完指令后，容器会自动停止。
此时容器并未删除，可以再次启动。
想要查看当前所有容器使用 `docker ps -a` 命令。

```bash
$ docker ps -a
CONTAINER ID        IMAGE                                     COMMAND             CREATED             STATUS                       PORTS               NAMES
5d1ebf88e299        centos/teye                               "/bin/bash"         18 hours ago        Exited (0) 16 hours ago                          reverent_mcnulty
```

### 停止和删除

删除不需要的容器时，使用 `docker rm` 命令，删除时容器可以通过名称或 ID 指定。

```bash
$ docker rm 5d1ebf88e299
```

对于正在运行的容器，需要先将其停止后再删除。
停止容器的命令为 `docker stop`，对应容器同样可以通过名称或 ID 指定。

```bash
$ docker stop 5d1ebf88e299
```

### 启动

在前面镜像的介绍中，展示了通过镜像启动容器的过程。
启动容器有两种方法，第一种是基于镜像新建容器启动，第二种是启动一个在终止状态的容器。

在第一种方式中，新建容器启动使用 `docker run` 命令。以上面 `ubuntu/ruty` 镜像为例，新建一个容器并输出 `Hello World` 命令如下。

```bash
$ docker run ubuntu/ruty /bin/echo "Hello World"
```

上面启动方法是非交互式的，也可以用参数 `-it` 通过交互式进行启动。
这种方式会启动一个命令行终端，其中 `-t` 参数分配一个伪终端并绑定到容器的标准输入上，`-i` 将容器的标准输入保持打开。
完整命令如下

```bash
# 以交互模式启动容器
$ docker run -it ubuntu/ruty /bin/bash

# 进入容器终端
root@<tag>:#
```

当使用 `docker run` 创建容器后，docker 在后台执行的标准操作包括：

* 检查本地是否存在镜像，不存在则从仓库下载；
* 利用镜像启动一个容器；
* 分配一个文件系统，并在只读的镜像层外挂载一层可读写层；
* 从宿主主机的网桥接口中建立一个虚拟接口给容器；
* 从地址池中配置一个 ip 地址给容器；
* 执行用户指定的应用程序；
* 执行完毕后容器被终止；

在第二种方式中，使用的是 `docker start` 命令启动终止容器。
首先用 `docker ps -a` 查看所有存在的容器，随后启动对应容器。

```bash
# 查看所有容器
$ docker ps -a
CONTAINER ID        IMAGE                                     COMMAND             CREATED             STATUS                       PORTS               NAMES
5d1ebf88e299        centos/teye                               "/bin/bash"         18 hours ago        Exited (0) 16 hours ago                          reverent_mcnulty

# 启动对应 id 容器
$ docker start 5d1ebf88e299
```

当容器启动后，并不会自动以交互模式进入容器中。想要以交互模式进入需要使用 `docker attach` 或 `docker exec` 命令[^12432]：

* `docker attach` 可以进入一个运行中容器的标准输入中，并执行对应指令。当使用 `exit` 退出时，对应的容器也会停止。
* `docker exec` 需要以 `-it` 参数分配并进行伪终端，但是并不会像 `attach` 命令一样在退出时导致容器停止。

[^12432]: https://blog.csdn.net/halcyonbaby/article/details/46884605

### 与主机数据交换

大部分情况下，仅仅使用 docker 容器建立环境时不够的，还需要主机目录中的应用或者数据进行操作。

对于一些比较小的文件或文件夹，可以在 dockerfile 中使用 `ADD` 命令进行复制。
也可以用 `docker cp` 命令在主机和容器之间进行数据拷贝，基本语法如下。

```bash
# 从容器中复制文件到本地
$ docker cp <CONTAINER>:<SRC_PATH> <DEST_PATH>

# 从本地复制文件到容器中
$ docker cp <DEST_PATH> <CONTAINER>:<SRC_PATH>
```

当经常需要在容器和本地间进行数据交互时，更方便的方法是把本地目录挂载到容器中，此时在本地或容器内都可以对数据和文件直接操作。
挂载方式是在容器启动时添加 `-v` 命令，并指定本地目录和容器中对应挂载位置。

```bash
$ docker run -it -v /usr/local:/usr/local <image> /bin/bash
```

此时，容器内的 `/usr/local` 与本地 `/usr/local` 为同一个目录，在容器内就可以直接操作本地文件和目录。

## vscode 中使用 docker

在 vscode 中提供了 docker 插件，可以方便的操作本地镜像和容器。
当在 vscode 中安装好插件后，打开 docker 侧栏，可以看到当前存在的镜像和容器。
右键选取对应容器，可以启动或停止。
当容器启动后，还可以 attach 到目标容器并打开命令行窗口进行操作，如图所示。

![docker with vscode](http://ww1.sinaimg.cn/large/7a1c18a8ly1gf11zgib0vj22by1iy7wh.jpg)

## 示例：搭建 texlive 环境

本节以搭建 texlive 环境过程来展示 docker 实际应用。在本节搭建 texlive 时将以 ubuntu 系统镜像作为基础。

根据前面介绍，使用 docker 搭建 texlive 环境时有两种方法，第一种是进入容器安装 texlive 各种包实现，第二种是用 dockerfile，在这里仅对第一种方法进行介绍。
首先下载 ubuntu 基础镜像，最新版本镜像仅有 73.8MB 大小。随后建立容器，并安装 texlive 的各种应用。

```bash
# 启动容器
$ docker run -it ubunut:latest /bin/bash

# 检查软件包更新
root@<tag>:# apt update

# 安装 texlive 包
root@<tag>:# apt-get install texlive-xetex texlive-latex-recommended

root@<tag>:# apt-cache search cjk

root@<tag>:# apt-get install latex-cjk-all cjk-latex latex-cjk-chinese

# 安装中文字体
root@<tag>:# apt-get install ttf-mscorefonts-installer
```

随后将上述容器保存，生成新的镜像 `latex/ubuntu`。查看新建镜像，可以看到最终大小为 2.45GB。

```bash
$ docker commit -p latex/ubuntu <container>

$ docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
latex/ubuntu                       latest              a5eb0889d570        5 seconds ago       2.45GB
```

在新的镜像中启动容器，新建一个 latex 中文文档进行测试。

```bash
$ mkdir latex && cd latex

$ vi test.tex
```

在 `test.tex` 文件中输入以下内容。

```bash
\documentclass[11pt]{article}
\usepackage[BoldFont,SlantFont,CJKsetspaces,CJKchecksingle]{xeCJK}
\setCJKmainfont[BoldFont=SimHei]{SimSun}
\setCJKmonofont{SimSun}% 设置缺省中文字体
\parindent 2em   %段首缩进
  
\begin{document}
\section{举例}
\begin{verbatim}
标点。
\end{verbatim}
  
汉字Chinese数学$x=y$空格
\end{document}
```

从新镜像中启动容器，并挂载本地目录 `latex`。

```bash
$ docker run -it -v ~/latex:/latex latex/ubuntu /bin/bash
```

进入容器 `latex` 目录下，使用 `xelatex` 编译测试文档。

```bash
root@<tag>:# cd latex

root@<tag>:# xelatex test.ext -o test.pdf
This is XeTeX, Version 3.14159265-2.6-0.999991 (TeX Live 2019/Debian) (preloaded format=xelatex)
 restricted \write18 enabled.
entering extended mode
(./test.tex
LaTeX2e <2020-02-02> patch level 2
L3 programming layer <2020-02-14>
...
...
...
Output written on test.pdf (1 page).
Transcript written on test.log.
```

生成 `test.pdf` 文件后，可以将文件从服务器下载到本地进行查看。
最终通过使用 docker 容器，实现了服务器上编译 latex 文件功能，并且 texlive 完全使用 ubuntu 环境的 apt 命令安装，不需要在集群中使用 root 命令，并且安装的软件与集群环境相互隔离。

