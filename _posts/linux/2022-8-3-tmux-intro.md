---
layout: post
title: tmux 使用说明
date: 2022-8-3
categories: linux
---

# tmux 简介

tmux 为 Terminal MultipleXer（终端复用器）缩写，可以在一个终端上开启多个复用终端，可以在多个用户间共享，并且避免由于长时间无操作导致终端无相应问题。

# tmux 使用

## 新建对话

新建 tmux 对话可以直接使用 `tmux` 命令即可，但是这种方法创建对话会默认用数字0、1、2等标识，识别较为困难。
```bash
$ tmux
```

新建终端时候，可以通过如下命令为终端命令，此时根据特定名称可以对终端识别，方便管理。
```bash
$ tmux new -s of-test-gpu-4
```

## 登录指定终端

使用 `tmux ls` 可以查看当前所有终端，如下所示：
```bash
$ tmux ls
gpu-16: 1 windows (created Wed Aug  3 02:35:39 2022) (attached)
monitor: 1 windows (created Wed Aug  3 06:11:05 2022) (attached)
```

当需要登录指定终端时，可以通过如下命令直接进入。
```bash
$ tmux a -t gpu-16
```
此时，进入终端后可以看到上次操作所有内容，并且可以继续进行未完成操作。

## 分屏功能

进入 tmux 的终端后，可以对单个终端进行分屏，得到多个终端。分屏有两种形式：

1. `ctr + b`，随后输入 `"` 得到上下分屏；
2. `ctr + b`，随后输入 `%` 得到左右分屏；

分屏后，需要切换屏幕时可以通过命令 `ctr + b`，随后输入 `o` 进行切换。
此外，左右分屏和上下分屏还可以通过 `ctr + b`，输入空格进行切换。
当需要关闭一个屏幕时，输入 `ctr + b` 加 `x` 即可。
