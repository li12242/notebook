---
layout: post
title: Intel APS 使用手册
date: 2021-1-5
categories: hpc
---

# 简介

Intel APS（Application Performance Snapshot）为 Intel 推出的一种能够快速分析应用瓶颈的工具。使用 APS 可以分析 CPU 利用率、内存访问效率、向量化、IO 和内存占用等方面对计算密集型应用运行性能影响。APS 可以展示关键优化点，并推荐对应的分析工具进行下一步深入分析，例如 Intel Advisor 对线程和向量化分析，Intel VTune Profiler 对热点过程进行分析。[^12321]

![](https://software.intel.com/content/dam/dita/develop/get-started-with-application-performance-snapshot/GUID-56C80296-921B-4E7C-9FD2-8BDF5F5DE5A1.png/_jcr_content/renditions/original)

[^12321]: https://software.intel.com/en-us/node/836966

Intel APS 需要与 Intel 编译器和 MPI 函数库支持。使用 Intel 编译器可以对 OpenMP 多线程不平衡问题进行分析，而 Intel MPI 函数库可以提供 MPI 进程负载不均衡的信息。

# 使用说明

## 环境设置

进行 APS 初始化设置则输入如下命令

```bash
$ source <install_aps_dir>/performance_snapshots/apsvars.sh
```

当使用 zshell 等 shell 环境时，加载 APS 环境时可能显示软件安装存在错误，如下所示

```
Invalid product installation. Please reinstall the tool.
```

这种情况下需要进入 bash 环境下，重新加载 aps 初始化脚本。
成功加载 aps 环境后，这时在环境变量中就可以找到 `aps` 和 `aps-report` 可执行程序，这两个程序分别用来对应用进行监控和显示监控结果。

## MPI 应用分析

在使用 APS 对 MPI 应用进行分析时，直接将 aps 参数加入 `mpirun` 命令即可，其格式为

```bash
$ mpirun <mpi_parameters> aps <my_app> <app_parameter>
```

注意此时是由 aps 调用 MPI 程序 `<my_app>` 运行。

尽管添加 APS 监控指令后，会对 MPI 应用运行效率有一定影响。为了将对性能影响降低，可以通过参数 `--collection-mode` 设置 aps 运行模式，

## APS 结果分析

在运行完毕后，会生成一个结果文件夹，文件夹名称为 `aps_result_<date>` 格式，其中 `<date>` 为 `20200421` 形式的日期。

使用 `aps-report` 对结果进行查看时，结果命令格式为 `aps-report [keys] [options] <file-name(s)>`。
使用 keys 对 
