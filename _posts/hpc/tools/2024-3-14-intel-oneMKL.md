---
layout: post
title: GPU support in Intel MPI
date: 2024-3-14
categories: hpc
---

Intel oneMKL GPU 版本函数 zheev 调用测试

# 简介

Intel OneMKL 目前支持 GPU 卸载计算，通过使用处理器自带的 UHD 显卡即可进行测试。本文对 Fortran 版本 OneMKL 函数的调用，编译和测试进行介绍。
关于 Fortran 版本 MKL 函数详细使用可以参考 Intel 手册 《Developer Reference for Intel® oneAPI Math Kernel Library for Fortran》。

# OneMKL GPU 卸载计算


# 函数调用

## zheev

查询手册中关于 zheev 函数介绍。此函数针对 Hermitian 矩阵 A 进行特征值计算，反馈对应特征值和特征矩阵。调用形式如下：

```fortran
call zheev(jobz, uplo, n, a, lda, w, work, lwork, rwork, info)
```

矩阵大小为 $n \times n$，此时工作空间数组 work 长度为 $max(1, lwork)$，而 $lwork >= max(1, 2n-1)$。

为了获取合适的工作数组 work 的长度，也可设置 lwork = -1，此时返回的 work 矩阵中第一个元素即是所需空间大小。此时对应 zheev 函数应该为两次调用，对应形式为：

```fortran
CALL ZHEEV(JOBZ, UPLO, N, A, LDA, W, WORK ,    -1, RWORK_, INFO)

LWORK_ = WORK(1)
ALLOCATE(WORK_(LWORK_))
CALL ZHEEV(JOBZ, UPLO, N, A, LDA, W, WORK_, LWORK, RWORK_, INFO)
```



