---
layout: post
title: HPC-X使用Intel编译器
date: 2022-2-7
categories: hpc
---

# ucx 编译

在 hpcx 中使用如下命令对 ucx 重新进行编译安装，其中 `--with-xpmem` 指向安装完成后的 xpmem 函数库。
关于 xpmem 编译可以参考相关内容。

```bash
./contrib/configure-release --prefix=${HPCX_HOME}/ucx-intel --enable-optimizations --enable-mt CC=icc CXX=icpc
make -j 16
make install
```

# OpenMPI 编译

参考 HPC-X 官网说明使用 Intel 对 OpenMPI 重新编译，编译过程如下：

```
$ tar xfp ${HPCX_HOME}/sources/openmpi-gitclone.tar.gz
$ cd ${HPCX_HOME}/sources/openmpi-gitclone
$ ./configure CC=icc CXX=icpc F77=ifort FC=ifort --prefix=${HPCX_HOME}/ompi-intel \
--with-hcoll=${HPCX_HOME}/hcoll \
--with-ucx=${HPCX_HOME}/ucx \
--with-platform=contrib/platform/mellanox/optimized \
2>&1 | tee config-icc-output.log
$ make -j32 all 2>&1 | tee build_icc.log && make -j24 install 2>&1 | tee install_icc.log
```

# 错误

在编译过程中，可能出现如下错误：

```
ld: cannot find -lnuma
ld: cannot find -lbfd
make[2]: *** [Makefile:1919: libmca_common_ucx.la] Error 1
make[2]: Leaving directory '<HPCX_DIR>/sources/openmpi-gitclone/opal/mca/common/ucx'
make[1]: *** [Makefile:2419: all-recursive] Error 1
make[1]: Leaving directory '<HPCX_DIR>/sources/openmpi-gitclone/opal'
make: *** [Makefile:1901: all-recursive] Error 1
```

解决方法是使用 yum 命令安装 `numactl-devel` 和 `binutils-devel` 模块即可。

# 参考

1. [Building HPC-X with the Intel Compiler Suite](https://docs.mellanox.com/display/HPCXv29/Installing+and+Loading+HPC-X#InstallingandLoadingHPCX-BuildingHPC-XwiththeIntelCompilerSuite)
