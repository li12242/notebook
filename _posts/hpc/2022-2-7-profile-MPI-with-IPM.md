---
layout: post
title: IPM对MPI应用进行分析
date: 2022-2-7
categories: MPI, hpc
---

# IPM 简介

[IPM](https://github.com/nerscadmin/IPM) 是 HPC 应用的集成监控软件，包括以下功能：

* MPI 函数调用计数，包括数据量和时间；
* POSIX 函数调用计数，包括数据量和时间；
* 硬件计数；
* 内存使用量计数；

监控完毕后结果保存在 xml 文件内，可以进一步使用后处理工具分析或者绘图。
IPM优点包括：

* 无需重新编译程序，只需要定义 `LD_PROLOAD` 变量即可；
* 正确监控程序要求。

# 部署

安装 IPM 按照以下步骤

```bash
./bootstrap.sh
./configure --prefix=<IPM_DIR>
make 
make install
```

安装时可能出现无法找到 -ludev 库的问题，直接使用如下命令生成链接文件即可。

```bash
yum install libudev-devel
```

# 使用

按照 [IPM](https://github.com/nerscadmin/IPM) 官网说明，需要在程序编译过程中加入 `-L$PREFIX/lib -lipm`。
除此之外，也可以通过定义环境变量 `LD_PROLOAD` 在已经编译好的 MPI 应用中使用。

在 IPM 运行过程中，可以调整以下参数修改监控过程。

* IPM_REPORT=none/terse(default)/full 控制输出文本横幅；
* IPM_REPORT_MEM=no/yes(default) 控制内存使用输出；
* IPM_LOG=none/terse(default)/full 控制XML文件输出内容；
* IPM_HPM=none/PAPI_L1_DCM,PAPI_FP_OPS,PAPI_TOT_INS,PAPI_TOT_CYC(default) 控制硬件监控数据输出，需要安装PAPI；
* IPM_LOGDIR=.(default)/<custom_path> XML文件输出路径。

## OpenMPI 监控

以 OpenMPI 为例，在运行过程中添加 `-x LD_PRELOAD=$IPM_DIR/lib/libipm.so` 环境变量即可能够使用 IPM 对 MPI 函数运行时间进行统计。
修改后的运行命令如下所示：

```bash
export IPM_DIR=/home/lilx/app/ipm-hpcx-icc
export IPM_LOG=full
export IPM_REPORT=full
export IPM_STATS=all
export IPM_KEYFILE=$IPM_DIR/etc/ipm_key_mpi
export LD_PRELOAD=$IPM_DIR/lib/libipm.so

export IPM_ADD_BARRIER_TO_REDUCE=1
export IPM_ADD_BARRIER_TO_ALLREDUCE=1
export IPM_ADD_BARRIER_TO_GATHER=1
export IPM_ADD_BARRIER_TO_ALL_GATHER=1
export IPM_ADD_BARRIER_TO_ALLTOALL=1
export IPM_ADD_BARRIER_TO_ALLTOALLV=1
export IPM_ADD_BARRIER_TO_BROADCAST=1
export IPM_ADD_BARRIER_TO_SCATTER=1
export IPM_ADD_BARRIER_TO_SCATTERV=1
export IPM_ADD_BARRIER_TO_GATHERV=1
export IPM_ADD_BARRIER_TO_ALLGATHERV=1
export IPM_ADD_BARRIER_TO_REDUCE_SCATTER=1

PROFILE_FLAGS="-x LD_PRELOAD -x IPM_LOG -x IPM_REPORT -x IPM_STATS -x IPM_KEYFILE -x IPM_ADD_BARRIER_TO_REDUCE -x IPM_ADD_BARRIER_TO_ALLREDUCE -x IPM_ADD_BARRIER_TO_GATHER -x IPM_ADD_BARRIER_TO_ALL_GATHER -x IPM_ADD_BARRIER_TO_ALLTOALL -x IPM_ADD_BARRIER_TO_ALLTOALLV -x IPM_ADD_BARRIER_TO_BROADCAST -x IPM_ADD_BARRIER_TO_SCATTER -x IPM_ADD_BARRIER_TO_SCATTERV -x IPM_ADD_BARRIER_TO_GATHERV -x IPM_ADD_BARRIER_TO_ALLGATHERV -x IPM_ADD_BARRIER_TO_REDUCE_SCATTER"
NCORES=64
mpirun $PROFILE_FLAGS -x LD_LIBRARY_PATH -n $NCORES -machinefile host$NCORES --allow-run-as-root --report-bindings --mca pml ucx --mca osc ucx --mca coll_hcoll_enable 1 -x UCX_NET_DEVICES=mlx5_0:1 -x HCOLL_MAIN_IB=mlx5_0:1 simpleFoam -parallel > log.simpleFoam
```

## Intel MPI 监控

Intel MPI使用方法与OpenMPI类似，也需要定义IPM相关环境变量。
由于Intel MPI运行过程中会继承当前 Shell 环境内定义的所有环境变量，因此在运行过程中不需要像OpenMPI一样将所有IPM环境变量传递过去，命令行不需要进行任何修改。

# 结果处理

在应用运行完毕后，会出现所有 MPI 函数的统计结果，如下所示。

```bash
##IPMv2.0.6########################################################
#
# command   : simpleFoam -parallel
# start     : Wed Oct 20 01:21:47 2021   host      : amd01
# stop      : Wed Oct 20 01:22:02 2021   wallclock : 15.40
# mpi_tasks : 64 on 1 nodes              %comm     : 27.07
# mem [GB]  : 30.81                      gflop/sec : 0.00
#
#           :       [total]        <avg>          min          max
# wallclock :        984.44        15.38        15.34        15.40
# MPI       :        266.51         4.16         2.32         5.70
# %wall     :
#   MPI     :                      27.07        15.11        37.04
# #calls    :
#   MPI     :       6891903       107685        52249       175418
# mem [GB]  :         30.81         0.48         0.46         0.50
#
#                             [time]        [count]        <%wall>
# MPI_Probe                   121.08           3654          12.30
# MPI_Waitall                  87.18         328419           8.86
# MPI_Allreduce                15.76         287488           1.60
# MPI_Recv                     15.38          79002           1.56
# MPI_Isend                    14.38        3048401           1.46
# MPI_Alltoall                 12.22          16896           1.24
# MPI_Irecv                     0.43        3048401           0.04
# MPI_Send                      0.04          79002           0.00
# MPI_Comm_create               0.04             64           0.00
# MPI_Comm_free                 0.01             64           0.00
# MPI_Buffer_attach             0.00             64           0.00
# MPI_Comm_rank                 0.00            192           0.00
# MPI_Comm_group                0.00             64           0.00
# MPI_Comm_size                 0.00            128           0.00
# MPI_Init                      0.00             64           0.00
#
###################################################################
```

此外，在目录内还会生成一个 xml 统计结果文件。
进一步对 MPI 函数进行后处理分析时，可以使用前处理工具 ipm_parse 对 xml 文件进行处理， 命令如下所示。

```
{IPM_HOME}/bin/ipm_parse -html <ipm_result>.xml
```

## 参考文献

1. [Profile an HPC Application](https://hpcadvisorycouncil.atlassian.net/wiki/spaces/HPCWORKS/pages/558497793/Profile+an+HPC+Application)

