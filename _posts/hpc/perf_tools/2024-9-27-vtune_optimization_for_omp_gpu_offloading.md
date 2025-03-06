---
layout: post
title: 使用 VTune 对 OpenMP GPU Offloading 应用分析和优化
date: 2024-3-22
categories: tools
---

# Profiling an OpenMP* Offload Application running on a GPU[^1]

[^1]: https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-0/profiling-openmp-offload-application.html


本教程说明如何构建和编译卸载到英特尔 GPU 上的 OpenMP* 应用程序。 该教程还介绍了如何使用英特尔® VTune Profiler 在 OpenMP 应用程序上运行具有 GPU 功能（高性能计算性能特征、GPU 卸载和 GPU 计算/媒体热点）的分析并检查结果。

## Run HPC Performance Characterization Analysis on the OpenMP Offload Application

要获得 OpenMP Offload 应用程序性能的高级摘要，请运行 HPC Performance Characterization 分析。该分析类型可帮助您了解应用程序如何利用 CPU、GPU 和可用内存。 您还可以了解代码的矢量化程度。

对于 OpenMP 卸载应用程序，HPC 性能特征分析会显示与每个 OpenMP 卸载区域相关的硬件指标。

## Analyze HPC Performance Characterization Data

从查看 Summary 窗格开始分析。 查看 Effective Physical Core Utilization（或 Effective Logical Core Utilization）和 "GPU Stack Utilization"部分，查看突出显示的问题。

在 "GPU 堆栈利用率 "部分，查看根据在这些区域花费的卸载时间排序的顶级 OpenMP 卸载区域。 您可以看到每个卸载区域的 GPU 利用率。

如果您使用全套调试信息编译应用程序，区域名称将包含其源代码位置。



