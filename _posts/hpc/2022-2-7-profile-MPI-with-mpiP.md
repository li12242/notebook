---
layout: post
title: mpiP对MPI应用进行分析
date: 2022-2-7
categories: MPI, hpc
---

# 简介

[mpiP](https://software.llnl.gov/mpiP/)是一个轻量级MPI监控工具，基本功能包括MPI函数调用时间统计，函数调用栈，源代码和行号追踪等。mpiP在监控完成后会在进程间进行通信，将监控结果合并到一个日志文件中。

mpiP使用时比较简单，不需要对可执行程序重新编译，只需通过定义LD_PRELOAD环境变量加载libmpiP.so动态库即可进行监控。为了能够正确监控文件名和行号，需要在可执行程序编译时使用 -g 参数。

# 部署

mpiP 按照如下步骤即可完成安装

```bash
$ configure
$ make
$ make install
```

在 configure 过程中，会检查需要的函数库是否存在，例如下面提示需要的 libunwind 函数库不存在。

```
checking for libunwind.h... no

  mpiP on x86_64 platforms requires libunwind.
  Please install libunwind and, if necessary, configure mpiP with
  appropriate CFLAGS and LDFLAGS settings.
```

可以使用 yum 命令安装，或者直接从 pkgs 网站直接下载 [libunwind-devel 安装包](http://mirror.centos.org/centos/7/os/x86_64/Packages/libunwind-devel-1.2-2.el7.x86_64.rpm)，再使用 `rpm -ivh <rpm_packages>` 命令进行安装。

# 使用

使用 mpiP 时可以直接通过设置环境变量 `export LD_PRELOAD=<mpiP_dir>/libmpiP.so` 来使用。
mpiP 运行参数也直接通过环境变量 `MPIP` 进行调整。

```bash
$ export MPIP="-t 10.0 -k 2"
```

运行完成后，mpiP 会生成对应结果文件，文件名为 `<应用名称>.<进程数>.<进程号>.<数字>.mpiP` 形式命名。

## OpenMPI

对于 OpenMPI，需要再运行命令中通过 `-x LD_PRELOAD=<mpiP_dir>/libmpiP.so` 添加运行环境变量，修改完毕后运行命令如下所示。

```bash
$ export LD_PRELOAD=/home/lilx/software/mpiP-3.5/libmpiP.so mpirun -n 320 
```

## Intel-MPI

使用 Intel MPI运行时，仅需定义如下环境变量即可。

```bash
export LD_PRELOAD=${HOME}/app/mpip/lib/libmpiP.so
export MPIP="-t 10.0 -k 4"
```

# 结果分析

mpiP 所有监控结果保存在结果文件中。
第一部分为执行任务情况和进程绑定等信息，如下所示：
```bash
  0 @ mpiP
  1 @ Command : ../../build/inspur/ice_ocean_SIS2/opt/MOM6
  2 @ Version                  : 3.5.0
  3 @ MPIP Build date          : Jan  6 2022, 17:45:06
  4 @ Start time               : 2022 01 06 17:30:57
  5 @ Stop time                : 2022 01 06 17:42:03
  6 @ Timer Used               : PMPI_Wtime
  7 @ MPIP env var             : -t 10.0 -k 4
  8 @ Collector Rank           : 0
  9 @ Collector PID            : 69624
  10 @ Final Output Dir         : .
  11 @ Report generation        : Single collector task
  12 @ MPI Task Assignment      : 0 cu01
  13 @ MPI Task Assignment      : 1 cu01
  14 @ MPI Task Assignment      : 2 cu01
```
**MPI Time** 第二部分为应用程序和MPI时间，其中 Apptime 是从 MPI_Init 结束到 MPI_Finalize 开始的墙钟时间。
MPI_Time 是 Apptime 中包含的所有 MPI 调用的墙钟时间。
MPI% 显示此 MPI_Time 与 Apptime 的比率。 
最后一行星号 (*) 是整个应用程序的统计结果。
```bash
---------------------------------------------------------------------------
@--- MPI Time (seconds) ---------------------------------------------------
---------------------------------------------------------------------------
Task    AppTime    MPITime     MPI%
  ┊0        664        320    48.23
  ┊1        665        324    48.79
  ┊2        665        292    43.88
  ┊3        665        169    25.43
  ┊4        665        229    34.47
```
**Callsites** 第三部分对应用中所有MPI函数调用进行了识别。
ID列为MPI调用对应的callsite ID，Lev是堆栈级别，File/Address是文件位置，Line是行号，Parent_Funct为父函数，MPI_Call为调用MPI。
```bash
29 ---------------------------------------------------------------------------
28 @--- Callsites: 241962 ----------------------------------------------------
27 ---------------------------------------------------------------------------
26  ID Lev File/Address                    Line Parent_Funct             MPI_Call
25   1   0 0x2af1d6b5ad43                       mpi_allreduce_           Allreduce
24   1   1 mpp_sum_mpi.h                     39 [unknown]
23   1   2 mpp_chksum.h                      35 [unknown]
22   1   3 MOM_surface_forcing_gfdl.F90    1630 [unknown]
21   2   0 0x2af1d6b60c39                       mpi_wait_                Wait
20   2   1 mpp_util_mpi.inc                 237 [unknown]
19   2   2 mpp_group_update.h               657 [unknown]
18   2   3 MOM_domains.F90                 1102 [unknown]
17   3   0 0x2af1d6b5ee57                       mpi_isend_               Isend
16   3   1 mpp_transmit_mpi.h                86 [unknown]
15   3   2 mpp_group_update.h               533 [unknown]
14   3   3 MOM_domains.F90                 1102 [unknown]
```
**Aggregate Time** 第四部分显示聚合时间信息，为耗时前20个MPI调用的预览。Call为出现调用类型，Site 为调用 ID，Time为总调用耗时（毫秒），COV 列通过显示从单个过程时间计算的变化系数来指示此调用点的单个过程的时间变化，此数值越大表示处理时间之间的差异越大。
```bash
1 ---------------------------------------------------------------------------
2 @--- Aggregate Time (top twenty, descending, milliseconds) ----------------
3 ---------------------------------------------------------------------------
4 Call                 Site       Time    App%    MPI%      Count    COV
5 Wait                 133032   5.88e+04    0.07    0.31       1200   0.00
6 Wait                 91174   5.77e+04    0.07    0.30        240   0.00
7 Wait                 87168   5.66e+04    0.07    0.30       1920   0.00
8 Wait                 133105   5.66e+04    0.07    0.30        150   0.00
9 Wait                 183007   5.59e+04    0.07    0.29       1920   0.00
10 Wait                 124646   5.59e+04    0.07    0.29       1200   0.00
```
**Aggregate Send Message Size** 第五部分仍为聚合信息，显示前20个发送消息最多的MPI调用。
```bash
1 ---------------------------------------------------------------------------
2 @--- Aggregate Sent Message Size (top twenty, descending, bytes) ----------
3 ---------------------------------------------------------------------------
4 Call                 Site      Count      Total       Avrg  Sent%
5 Isend                232532        720   5.13e+08   7.12e+05   0.06
6 Isend                231511        720   4.19e+08   5.82e+05   0.05
7 Isend                239687        720   3.85e+08   5.35e+05   0.05
8 Isend                227851        960   3.85e+08   4.01e+05   0.05
9 Isend                224014        960   3.85e+08   4.01e+05   0.05
10 Isend                202184        960   3.85e+08   4.01e+05   0.05
```
当启用了集合直方图或点对点监控（MPIP=-y或-p），下一节将提供每个 MPI 集合通讯和点对点通信调用的直方图数据，报告特定通信大小和数据大小的总 MPI 集合时间百分比。

**Callsite statistics** 会报告所有任务中每个调用站点的统计数据列表，然后是一个聚合行（由 Rank 列中的星号表示）。 第一部分是操作时间，然后是消息大小。
```bash
1 ---------------------------------------------------------------------------
2 @--- Callsite Time statistics (all, milliseconds): 241962 -----------------
3 ---------------------------------------------------------------------------
4 Name              Site Rank  Count      Max     Mean      Min   App%   MPI%
5 Allreduce            1    *      1    0.106    0.106    0.106   0.00   0.00
6
7 Allreduce            6    *      1    0.161    0.161    0.161   0.00   0.00
8
9 Allreduce           14    *      1    0.149    0.149    0.149   0.00   0.00
......
......
1 ---------------------------------------------------------------------------
2 @--- Callsite Message Sent statistics (all, sent bytes) -------------------
3 ---------------------------------------------------------------------------
4 Name              Site Rank   Count       Max      Mean       Min       Sum
5 Allreduce            1    0       1         8         8         8         8
6 Allreduce            1    *       1         8         8         8         8
7
8 Allreduce            6    0       1         8         8         8         8
9 Allreduce            6    *       1         8         8         8         8
```
