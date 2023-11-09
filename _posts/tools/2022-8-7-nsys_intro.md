---
layout: post
title: nsys 使用介绍
date: 2022-8-7
categories: tools
---

## 简介

在 cuda-11.x 版本中，nvprof 等监控工具已经逐渐被 Nsight Systems、Nsight Compute 等工具取代，基于以上工具，可以对计算过程中进程同步、数据移动、计算与通信重叠和 GPU 核心调用等内容进行分析和优化。
Nsight Systems 在任务运行过程中，对数据移动、并行负载均衡性、MPI 通讯、CUDA 内核函数等过程进行监控，并可以对 CUDA、OpenACC、MPI、OpenMP 等并行模式进行详细分析，有利于判断当前任务主要运行瓶颈。

## 监控命令

采用 Nsight Systems 监控时，整体运行命令如下，其中 `--trace` 参数说明了需要对 cuda 和 MPI 等命令进行监控。

```bash
/opt/nvidia/nsight-systems/2022.3.4/bin/nsys profile --stats=true --trace=cuda,mpi --output=nsys.$SOLVER.$VAR_TYPE.np$NP --force-overwrite=true mpirun $OMP_FLAGS -n $NP icoFoam -parallel 2>&1
```

监控完成后，在目录会生成 *.nsys-rep 和 *.sqlite 两个结果文件。