---
layout: post
title: Intel Vtune 内核模块配置
date: 2025-10-21
categories: hpc
---

本文对 Intel VTune 工具使用前内核模块编译和安装进行介绍。

# 使用问题

首次在节点使用 Intel VTune 工具对应用微架构进行监控时，可能出现如下报错。这是由于 Intel VTune 没有找到 Sampling Drivers 或当前平台 perf 工具，所以无法对系统微架构进行监控。

```
vtune: Error: This analysis requires one of these actions: a) Install Intel Sampling Drivers. b) Configure driverless collection with Perf system-wide profiling. To enable Perf system-wide profiling, set /proc/sys/kernel/perf_event_paranoid to 0 or set up Perf tool capabilities.
vtune: Warning: Access to /proc/kallsyms file is limited. Consider changing /proc/sys/kernel/kptr_restrict to 0 to enable resolution of OS kernel and kernel module symbols.
```

解决办法有两个：

1. 安装 linux 系统 perf 工具，并设置配置文件 `/proc/sys/kernel/perf_event_paranoid` 值为 0，使 Intel VTune 调用 perf 实现微架构监控。
2. 安装 Intel Sampling Drivers，通过内核模块实现微架构监控。

这里主要推荐第 2 种方法，主要原因是 perf 工具和内核相关。对于最新处理器，如果内核比较旧并且未安装系统补丁，那么可能缺少部分性能事件，或导致一些性能事件监控不准确。

# 内核模块配置

1. 首先进入 `${VTUNE_PROFILER_DIR}/sepdk/src` 目录内，运行 `build-driver` 脚本对模块进行重新编译。
2. 运行 `insmod-sep` 脚本安装内核模块。

运行完毕后出现如下说明。注意 Intel Sampling Drivers 默认配置为允许 vtune 用户组调用，因此对于调用 VTune 微架构监控用户需要添加至此用户组内。

```
...
Setting group ownership of device file to group "vtune" ... done.
Setting file permissions of device file to "660" ... done.

The socwatch2_15-x32_64-4.18.0-348.el8.x86_64smp driver has been successfully loaded.

NOTE:

The driver is accessible only to users under the group 'vtune'.
Please add the users to the group 'vtune' to use the tool.

To change driver access group, reinstall the driver using -g <desired_group> option.


NOTE:
The driver is accessible only to users under the group vtune.
Please add the users to the group vtune to use the tool.

To change driver access group, reload the driver using -g <desired_group> option.
```