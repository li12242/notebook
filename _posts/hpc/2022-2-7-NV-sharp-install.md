---
layout: post
title: NVIDIA Sharp服务安装与使用
date: 2022-2-7
categories: hpc
---

# NV Sharp

## 部署

按照官网说明安装对应版本 OFED 驱动，在目录 /opt/mellanox/sharp 下存在 sharp 配置目录。

## 启动服务

首先寻找 SM 所在节点，通过 ibstat 与 iblinkinfo 确定 SM 节点号，随后安装 sharp 服务。
使用 `$HPCX_SHARP_DIR/sbin/sharp_daemons_setup.sh` 脚本在 SM 节点和计算节点上分别安装 sharp_am 和 sharpd 服务。

1. 在计算节点上，运行 `$HPCX_SHARP_DIR/sbin/sharp_daemons_setup.sh -s -d sharpd` 安装 sharpd 服务，随后运行 `systemctl start sharpd` 启动 sharpd 服务。
2. 在 SM 节点上，运行 `$HPCX_SHARP_DIR/sbin/sharp_daemons_setup.sh -s -d sharp_am` 安装 sharp_am 服务，随后启动服务 `systemctl start sharp_am`。

注意在启动服务后检查是否正常运行，例如当启动 sharp_am 服务非 SM 节点时，会出现如下错误。
此时需要更换 sharp_am 服务启动节点，保证与 SM 节点一致。

```
FabricProvider must bind to port with master SM (SM LID: 7 local LID:1)
FabricGraph: fabric provider local port is no valid
LoadFabric critical error. Exiting.
```

当路由类型不符时，会显示如下错误：

```
LoadFabric critical error: failed to set topology type according to routing engine minhop. Exiting.
```

此时需要修改 `/etc/opensm/opensm.conf` 配置文件，将路由拓扑类型改为 Fat tree。

```
# Routing engine
# Multiple routing engines can be specified separated by
# commas so that specific ordering of routing algorithms will
# be tried if earlier routing engines fail.
# Supported engines: minhop, updn, dnup, file, ftree, lash,
#    dor, torus-2QoS, kdor-hc, dfsssp (EXPERIMENTAL),
#    sssp (EXPERIMENTAL), chain,
#    pqft (EXPERIMENTAL),
#    dfp, dfp2 (EXPERIMENTAL), ar_updn, ar_ftree, ar_torus, ar_dor (AR)
routing_engine ar_ftree,ar_updn
```

## 测试

通过 osu-micro-benchmarks-5.6.3 中集合通信进行基准测试，下载、安装和运行程序命令如下：

```bash
# 下载 osu-micro-benchmarks
wget http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-5.6.3.tar.gz
tar xvf osu-micro-benchmarks-5.6.3.tar.gz

# 配置环境
module use /opt/hpcx-v2.9.0-gcc-MLNX_OFED_LINUX-5.2-1.0.4.0-redhat8.2-x86_64/modulefiles/
module load hpcx

# 编译 osu-micro-benchmark
./configure CC=mpicc
make

# 测试
mpirun -H cu03,cu04 --allow-run-as-root --bind-to core --map-by node -x LD_LIBRARY_PATH -mca pml ucx -mca coll_hcoll_enable 1 -x UCX_NET_DEVICES=mlx5_0:1 -x HCOLL_MAIN_IB=mlx5_0:1  -x SHARP_COLL_LOG_LEVEL=3 -x HCOLL_ENABLE_SHARP=3  ./osu_allreduce -m 0:4096
```

## 附录 

### SM 节点查询

通过 ibstat 命令查看 SM 节点 LID 号。

```bash
[root@cu02 ~]# ibstat
CA 'mlx5_0'
	CA type: MT4123
	Number of ports: 1
	Firmware version: 20.30.1004
	Hardware version: 0
	Node GUID: 0xb8cef60300025b90
	System image GUID: 0xb8cef60300025b90
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 200
		Base lid: 17
		LMC: 0
		SM lid: 1
		Capability mask: 0x2651e848
		Port GUID: 0xb8cef60300025b90
		Link layer: InfiniBand
```

随后通过 iblinkinfo 查看对应 LID 编号对应的节点名，如下所示。

```
[root@cu02 ~]# iblinkinfo
CA: cu01 HCA-1:
      0xb8cef60300025b8c      1    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   40[  ] "Quantum Mellanox Technologies" ( )
CA: Mellanox Technologies Aggregation Node:
      0x0c42a103008982ac      4    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   41[  ] "Quantum Mellanox Technologies" ( )
CA: cu05 HCA-1:
      0x043f720300f744ea     21    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   38[  ] "Quantum Mellanox Technologies" ( )
CA: cu06 HCA-1:
      0x043f720300f28976     12    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   37[  ] "Quantum Mellanox Technologies" ( )
CA: cu10 HCA-1:
      0xb8cef60300025b70      7    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   26[  ] "Quantum Mellanox Technologies" ( )
CA: io01 HCA-1:
      0x0c42a1030022a516     14    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   36[  ] "Quantum Mellanox Technologies" ( )
CA: cu09 HCA-1:
      0xb8cef60300025b7c      8    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   25[  ] "Quantum Mellanox Technologies" ( )
CA: cu13 HCA-1:
      0x043f720300da665a     11    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   24[  ] "Quantum Mellanox Technologies" ( )
CA: cu17 HCA-1:
      0xb8cef60300025b84     16    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   23[  ] "Quantum Mellanox Technologies" ( )
CA: cu18 HCA-1:
      0xb8cef60300025b88     22    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   22[  ] "Quantum Mellanox Technologies" ( )
CA: cu14 HCA-1:
      0x043f720300da668e     19    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   21[  ] "Quantum Mellanox Technologies" ( )
CA: cu12 HCA-1:
      0xb8cef60300025b80      5    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   16[  ] "Quantum Mellanox Technologies" ( )
CA: cu15 HCA-1:
      0x043f720300f744ee     10    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   15[  ] "Quantum Mellanox Technologies" ( )
CA: cu11 HCA-1:
      0xb8cef60300025d24      6    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   14[  ] "Quantum Mellanox Technologies" ( )
CA: cu16 HCA-1:
      0x043f720300f4bd6e     13    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   13[  ] "Quantum Mellanox Technologies" ( )
CA: cu19 HCA-1:
      0x043f720300d7506e     15    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   12[  ] "Quantum Mellanox Technologies" ( )
CA: cu20 HCA-1:
      0x043f720300da66a2     18    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   11[  ] "Quantum Mellanox Technologies" ( )
CA: cu08 HCA-1:
      0x043f720300f4bcca     23    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3    3[  ] "Quantum Mellanox Technologies" ( )
CA: cu04 HCA-1:
      0xb8cef60300025b74      9    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3    4[  ] "Quantum Mellanox Technologies" ( )
CA: cu03 HCA-1:
      0xb8cef60300025b94      2    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3    2[  ] "Quantum Mellanox Technologies" ( )
CA: cu07 HCA-1:
      0x043f720300f744e2     20    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3    1[  ] "Quantum Mellanox Technologies" ( )
Switch: 0x0c42a103008982a4 Quantum Mellanox Technologies:
           3    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      20    1[  ] "cu07 HCA-1" ( )
           3    2[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       2    1[  ] "cu03 HCA-1" ( )
           3    3[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      23    1[  ] "cu08 HCA-1" ( )
           3    4[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       9    1[  ] "cu04 HCA-1" ( )
           3    5[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3    6[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3    7[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3    8[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3    9[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   10[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   11[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      18    1[  ] "cu20 HCA-1" ( )
           3   12[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      15    1[  ] "cu19 HCA-1" ( )
           3   13[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      13    1[  ] "cu16 HCA-1" ( )
           3   14[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       6    1[  ] "cu11 HCA-1" ( )
           3   15[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      10    1[  ] "cu15 HCA-1" ( )
           3   16[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       5    1[  ] "cu12 HCA-1" ( )
           3   17[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   18[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   19[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   20[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   21[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      19    1[  ] "cu14 HCA-1" ( )
           3   22[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      22    1[  ] "cu18 HCA-1" ( )
           3   23[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      16    1[  ] "cu17 HCA-1" ( )
           3   24[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      11    1[  ] "cu13 HCA-1" ( )
           3   25[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       8    1[  ] "cu09 HCA-1" ( )
           3   26[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       7    1[  ] "cu10 HCA-1" ( )
           3   27[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   28[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   29[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   30[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   31[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   32[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   33[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   34[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   35[  ] ==(                Down/ Polling)==>             [  ] "" ( )
           3   36[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      14    1[  ] "io01 HCA-1" ( )
           3   37[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      12    1[  ] "cu06 HCA-1" ( )
           3   38[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      21    1[  ] "cu05 HCA-1" ( )
           3   39[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>      17    1[  ] "cu02 HCA-1" ( )
           3   40[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       1    1[  ] "cu01 HCA-1" ( )
           3   41[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       4    1[  ] "Mellanox Technologies Aggregation Node" ( )
CA: cu02 HCA-1:
      0xb8cef60300025b90     17    1[  ] ==( 4X        53.125 Gbps Active/  LinkUp)==>       3   39[  ] "Quantum Mellanox Technologies" ( )
```

### sharp 服务无法启动

使用 `-x HCOLL_ENABLE_SHARP=3` 参数时会强制使用 sharp，运行出现如下错误。

```bash
[cu02:2:10642 - context.c:276] ERROR failed to open sharp session with SHARPD daemon. please check daemon status
[cu02:19:10655 unique id 1827930113] ERROR Not connected in sharp_init_client_session.

[cu02:19:10655 - context.c:276] ERROR failed to open sharp session with SHARPD daemon. please check daemon status
[LOG_CAT_SHARP] Failed to initialize SHArP collectives:Cannot connect to SHArPD(-8)  job ID:1827930113
[LOG_CAT_SHARP] Fallback is disabled. exiting ...
--------------------------------------------------------------------------
Primary job  terminated normally, but 1 process returned
a non-zero exit code. Per user-direction, the job has been aborted.
--------------------------------------------------------------------------
--------------------------------------------------------------------------
mpirun detected that one or more processes exited with non-zero status, thus causing
the job to be terminated. The first process to do so was:

  Process name: [[27892,1],27]
  Exit code:    255
--------------------------------------------------------------------------
```

检查本地 sharpd 服务，需要保证启动的 sharpd 服务为 hpcx 的版本，不能使用 /opt/mellanox 目录下的 sharp 建立服务。

## 参考

1. [Running MLNX SHARP Daemons](https://docs.mellanox.com/display/sharpv214/Running+MLNX+SHARP+Daemons)

2. [Using Mellanox SHARP with Open MPI](https://docs.mellanox.com/display/sharpv214/Using+Mellanox+SHARP+with+Open+MPI)
