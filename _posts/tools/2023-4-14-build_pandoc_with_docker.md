---
layout: post
title: Windows 平台使用 docker 部署 pandoc 工具
date: 2023-04-14
categories: tools
---

# 简介

前面介绍过如何通过 pandoc 将 markdown 转换为 latex 格式并输出为 PDF 文件。在本文中，将介绍如何通过 docker 将这些服务部署到镜像中。

# docker 部署

在 Windows 平台，下载并安装 docker desktop 软件。在安装完成后，打开 docker 后可能提示需要将 wsl 内核升级。以管理员权限打开 Windows 下命令行工具，输入下面命令对 wsl 内核进行升级。

```powershell
>> wsl --update
```

# 镜像部署

在部署好 docker 服务后，打开命令行工具开始进行 pandoc 镜像部署。

## Ubuntu 环境

为了后面省去 latex 软件安装过程，选择包含 latex 环境镜像。首先搜索相关镜像结果如下：

```powershell
>> docker search xelatex
NAME                            DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
moss/xelatex                    Docker container to compile XeLaTeX files in…   7                    [OK]
......
amnay/xelatex                   Xelatex container I use to build my resume. …   0
kitakami/xelatex                XeLaTeX 基本编译环境 （中文）                             0
templex/xelatex-ngerman2        dsvfdsfg                                        0                    [OK]
danhatesnumbers/xelatex                                                         0
weirdgiraffe/xelatex            Docker image to generate cv using xelatex       0
```

选择包含中文 XeLaTeX 环境的镜像 `kitakami/xelatex`，使用如下 docker 命令拉取。

```powershell
>> docker pull kitakami/xelatex
```

随后新建容器，并且以交互模式进入。

```powershell
>> docker run -it -–name=pandoc-md kitakami/xelatex bash
```

在进入新的 Ubuntu 系统后，许多工具和命令无法找到，并且默认 apt 源也无法搜索到这些工具。需要首先对 apt 源进行配置，随后通过 apt 命令安装。
采用如下命令更新 apt 源文件。

```bash
mv /etc/apt/sources.list /etc/apt/sources.list.bak
echo "deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse" > /etc/apt/sources.list
echo "deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse" >> /etc/apt/sources.list
echo "deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse" >> /etc/apt/sources.list
echo "deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse" >> /etc/apt/sources.list
echo "deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse" >> /etc/apt/sources.list
echo "deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse" >> /etc/apt/sources.list
echo "deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse" >> /etc/apt/sources.list
echo "deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse" >> /etc/apt/sources.list
echo "deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse" >> /etc/apt/sources.list
echo "deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse" >> /etc/apt/sources.list
```

修改完成 `/etc/apt/sources.list` 后，使用如下命令更新 apt 源。随后即可使用 apt-get 命令对不同工具进行安装。

```bash
$ apt-get update
```

## pandoc 环境

pandoc 依赖 python 工具，需要使用如下命令安装相应工具。

```bash
$ apt-get install -y pandoc pip 
```

由于 pandoc-xnos 只能使用 pip 安装，运行如下命令安装 pandoc 插件，保证在文档中图片和表格索引能够正常转换。

```bash
$ pip install pandoc-xnos
$ pip install pandoc-fignos
$ pip install pandoc-tablenos
```

## latex 环境

由于系统中已经部署了完成的 xelatex 环境，因此不需要进行安装。查询 Latex 相关工具位置，如下所示：

```bash
$ which latex
/usr/bin/pdflatex

$ which xelatex
/usr/bin/xelatex
```

# pandoc 配置及使用

## 模板文件

在本地新建 pandoc 配置文件及模板，上传到 docker 容器中目录 `/data`。

```powershell
>> docker cp .\0-setting.md 544523df5180:/data
>> docker cp .\makefile 544523df5180:/data
>> docker cp -r .\config 544523df5180:/data
```

## 字体文件

在 pandoc 编译时，需要使用 Windows 等线字体，因此也需要将字体文件上传到 docker 容器内。

```powershell
# cd C:\Windows\Fonts
>> docker cp .\Deng.ttf 544523df5180:/usr/share/fonts/win10/
>> docker cp .\Dengb.ttf 544523df5180:/usr/share/fonts/win10/
>> docker cp .\Dengl.ttf 544523df5180:/usr/share/fonts/win10/
```

在 docker 容器中，通过如下命令更新字体缓存，并且查看 DengXian 字体是否正确安装。

```bash
$ fc-cache -fv

$ fc-list :lang=zh
/usr/share/fonts/truetype/arphic-bkai00mp/bkai00mp.ttf: AR PL KaitiM Big5,文鼎ＰＬ中楷:style=Regular
/usr/share/fonts/win10/Dengl.ttf: DengXian,等线,DengXian Light,等线 Light:style=Light,Regular
/usr/share/fonts/truetype/arphic-gkai00mp/gkai00mp.ttf: AR PL KaitiM GB,文鼎ＰＬ简中楷:style=Regular
/usr/share/fonts/win10/Deng.ttf: DengXian,等线:style=Regular
/usr/share/fonts/truetype/arphic-gbsn00lp/gbsn00lp.ttf: AR PL SungtiL GB,文鼎ＰＬ简报宋:style=Regular
/usr/share/fonts/truetype/arphic-bsmi00lp/bsmi00lp.ttf: AR PL Mingti2L Big5,文鼎ＰＬ細上海宋:style=Regular
/usr/share/fonts/win10/Dengb.ttf: DengXian,等线:style=Bold
```

## 编译验证

上传 `0-setting.md` 和 `makefile` 文件，使用 make 命令运行 pandoc 转换脚本，查看是否能够正常生成 pdf 文件，如下所示。

```bash
$ make
pandoc \
        --filter pandoc-xnos \
        --template=config/template.tex \
        -N -s --toc \
        --listings \
        --pdf-engine=xelatex \
        --number-sections \
        -V geometry:margin=1in \
        -V fontsize=12pt \
        0-setting.md  -o main.pdf
```

## 运行脚本

在使用 docker 容器内 pandoc 时，无法对容器外部 Markdown 文件进行转换，只能在运行时将外部工作目录挂载到容器内，并且调用 pandoc 命令。

编写 run.sh 运行脚本，放在 `/data` 目录下。运行此脚本会自动将 `/build` 目录下所有 markdown 文件名保存在 `SRC` 变量中，随后将文件合并编译为 PDF 文件。

```bash
#!/bin/bash
WDIR=/build
FILE=${WDIR}/main.pdf
SRC=$(ls ${WDIR}/*.md)

cd ${WDIR} & pandoc \
	--filter pandoc-xnos \
	--template=config/template.tex \
	-N -s --toc \
	--listings \
	--pdf-engine=xelatex \
	--number-sections \
	-V geometry:margin=1in \
	-V fontsize=12pt \
	${SRC} -o ${FILE}
```

## 容器保存

在建立好运行脚本并测试无误后，使用如下命令将容器保存为镜像，其中 544523df5180 为容器 ID。

```powershell
>> docker commit 544523df5180 pandoc-md
```

检查当前镜像，确认保存成功。

```powershell
>> docker images -a
REPOSITORY                    TAG       IMAGE ID       CREATED              SIZE
pandoc-md                     latest    ece4de405b52   About a minute ago   1.4GB
ubuntu                        latest    08d22c0ceb15   5 weeks ago          77.8MB
docker/disk-usage-extension   0.2.5     5f8f804dbfa2   8 months ago         2.79MB
kitakami/xelatex              latest    5b1e914e9d50   4 years ago          762MB
```

在 Windows 环境内运行时，在工作目录内打开命令行，将当前目录绑定到容器内 `/build` 目录运行。

```powershell
>> docker run -v ${PWD}:/build --name=pandoc-test pandoc-md /data/run.sh
```


