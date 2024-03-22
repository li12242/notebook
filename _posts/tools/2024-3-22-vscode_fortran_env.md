---
layout: post
title: VSCode Fortran 语言环境设置
date: 2024-3-22
categories: tools
---

# VSCode Fortran 环境

目前VSCode推荐的Fortran插件只有Modern Fortran。此插件内需要配置fortls和fprettify等工具，下面介绍服务器上如何部署此工具。

# 依赖工具安装

## Miniconda

fortls和fprettify最简单的安装方式是通过pip安装，因此首先需要安装Miniconda。安装完毕后在用户目录内设置`.condarc`文件，设置使用国内conda源，内容如下：
```bash
channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
show_channel_urls: true
```

对于机房内无法直接访问外网服务器，需要设置代理登录。在管理节点开启squid服务，随后在计算节点环境变量中设置如下，其中`3128`为squid默认使用端口。
```bash
export http_proxy="http://<管理节点IP>:3128"
export https_proxy="http://<管理节点IP>:3128"
```

## fortls 与 fprettify

使用pip3安装fortls和fprettify，可以看到安装完毕后对应程序放置在`~/.local/bin`目录下。在打开VSCode之前将此目录添加到`PATH`环境变量中即可。

```bash
$ pip3 install --user fortls
Collecting fortls
  Downloading fortls-2.13.0-py3-none-any.whl (92 kB)
     |████████████████████████████████| 92 kB 220 kB/s
Collecting packaging
  Downloading packaging-24.0-py3-none-any.whl (53 kB)
     |████████████████████████████████| 53 kB 363 kB/s
Collecting json5
  Downloading json5-0.9.24-py3-none-any.whl (30 kB)
Installing collected packages: packaging, json5, fortls
  WARNING: The script pyjson5 is installed in '/home/lilongx/.local/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
  WARNING: The script fortls is installed in '/home/lilongx/.local/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
Successfully installed fortls-2.13.0 json5-0.9.24 packaging-24.0

[09:48 AM]-[lilongx@NF5698M7]-[~]-
$ pip3 install --user fprettify
Collecting fprettify
  Downloading fprettify-0.3.7-py3-none-any.whl (28 kB)
Collecting configargparse
  Downloading ConfigArgParse-1.7-py3-none-any.whl (25 kB)
Installing collected packages: configargparse, fprettify
  WARNING: The script fprettify is installed in '/home/lilongx/.local/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
Successfully installed configargparse-1.7 fprettify-0.3.7
```