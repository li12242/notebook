---
layout: post
title: GPU support in Intel MPI
date: 2024-1-5
categories: hpc
---

# 简介

本文参考Intel手册《oneAPI GPU Optimization Guide》中*Intel MPI for GPU Clusters*相关章节介绍。将对GPU awared Intel GPU使用方法进行介绍。

在 2019 Updata 8 版本中，Intel MPI 加入了对 GPU buffer 支持功能。在多进程通信中将直接传递 GPU buffer，避免了 host 和 device 端数据传输过程。

# 用法

在介绍 Intel MPI 中 GPU support 用法之前，首先介绍下 OpenMPI 中对于 CUDA 或 OpenACC 用法支持，随后对二者用法进行下比较。

## OpenACC + OpenMPI

在过去异构计算基本以 CUDA 和 OpenACC 为标准，而 NVIDIA 也最早在 OpenMPI 中加入对 GPU-awared 支持。在 OpenACC 中，通过引语 `host_data` 可以将数组的设备地址暴露给 host 端，从而在函数调用过程中，直接使用 device 端数组指针进行操作。`host_data` 引语接受 `use_device` 引语，并且在后者参数中指明哪些设备数组应该暴露给主机。

```fortran
!$ACC HOST_DATA USE_DEVICE(X,C)
CALL MPI_allgatherv(x(1), nrcv(COMM%NODE_ME), MPI_double_complex, &
                    c(1), nrcv(1), prcv(1), MPI_double_complex, &
                    COMM%MPI_COMM, ierror)
CHECK_MPI_ERROR(ierror, 'M_allgatherv_z', 'MPI_allgatherv')
!$ACC END HOST_DATA
```

以上述代码为例，MPI_Allgatherv 函数调用是在 host 端执行，但是参数中 `x` 和 `c` 等数组会替换为对应 device 端数组指针。完成 MPI 通信后，设备端数组包含值会被直接修改。类似用法也会在 OpenACC 代码调用 CUBLAS 函数中用到，如下所示。

```fortran
!$acc data create(x,y)
!$acc kernels
x(:) = 1.0
y(:) = 0.0
!$acc end kernels

!$acc host_data use_devices(x, y)
call cublassaxpy(N, 2.0, x, 1, y, 1)
!$acc end host_data

!$acc update self(y)
!$acc end data
```

## OpenMP + Intel MPI

在 Intel MPI 中，由于 Intel GPU 目前仅支持 OpenMP 和 SYCL 两种方式进行 GPU 卸载计算，所以对应 GPU support 使用 OpenMP + Intel MPI 方法实现。在 OpenMP 中，主要通过两个引语 `use_device_ptr` 或 `is_device_ptr` 声明变量使用对应设备数据指针。在 Intel MPI 运行时，通过环境变量 I_MPI_OFFLOAD 指定对 GPU buffer 直接通信。对应示例代码如下所示：

```c
#pragma omp target data map(to: rank, values[0:num_values], num_values) use_device_ptr(values)
{
    #pragma omp target parallel for is_device_ptr(values)
    for (i = 0; i < num_values; ++i){
        values[i] = values[i] + rank + 1;
    }

    MPI_Send(values, num_values, MPI_INT, dest_rank, tag, MPI_COMM_WORLD);
}
```

对应编译和运行命令为

```bash
$ mpicc -cc=icx -qopenmp -fopenmp-targets=spir64 test.c -o test

$ export I_MPI_OFFLOAD=1
$ mpirun -n 2 ./test
```

## 用法总结

从上述对比可以看出，OpenACC 通过 `host_data` 和 `use_devices` 引语将设备指针暴露给 MPI，而 Intel MPI 中也是利用 OpenMP 中引语 `use_device_ptr` 等实现相同功能。在 MPI 运行时，仍然在 host 端调用，会直接将 GPU buffer 数据进行交换，对于 host 端数据不会进行修改。

# 测试

在 VASP 代码中，对 mpimy 模块内 `M_sum_d` 函数进行修改，保证其可以使用 Intel MPI 中 GPU support 功能。`M_sum_d` 函数通过调用 MPI_Allreduce 实现各进程内数组对应元素求和。VASP 原始代码中通过 OpenACC 引语支持 GPU-awared OpenMPI 运行。为提供针对 Intel MPI 的支持，在函数中加入如下代码。

```fortran
if (OMP_EXEC_ON) then
    !$omp target data map(alloc:DTMP_m) use_device_addr(vec)
    DO j = 1, n, NDTMP
        ichunk = MIN(n - j + 1, NDTMP)
        call MPI_allreduce(vec(j), DTMP_m(1), ichunk, &
                        MPI_double_precision, MPI_sum, &
                        COMM%MPI_COMM, ierror)
        call __DCOPY__(ichunk, DTMP_m(1), 1, vec(j), 1)
    end do
    !$omp end target data

    PROFILING_STOP('m_sumb_d')
    return
end if
```

其中 `OMP_EXEC_ON` 判断使用 OpenMP 卸载计算，此时会调用 GPU support 版本 Intel MPI。在调用前，首先构造对应的 target data 区域，其中为 `DTMP_m` 在设备上申请对应数据空间，并且指明 `vec` 变量使用设备指针。在 `MPI_allreduce` 运行时，会对 GPU 上 vec 数据进行规约，对应结果储存在 DTMP_m 中。

为了对 MPI 通信过程进行测试，可以撰写对应测试函数 `test_m_sumb_d`。每个进程首先调用 `malloc_vec_d` 对数组 `vec` 进行初始化，包括申请设备数组空间并赋值为进程编号。随后调用 `M_sumb_d` 函数对设备数组进行规约，执行完毕后检查 host 和 device 端数值元素值。

```fortran
   !> \brief test m_sumb_d
   subroutine test_m_sumb_d()
      implicit none
      real(q), allocatable :: vec(:)
      integer :: len = 80
      if (comm%node_me == comm%ionode) then
         write (*, *) "=============test_m_sumb_d============="
         write (*, *) "Testing vector length = ", len
         write (*, *) "Initializing vector with rank index = ", comm%node_me
      end if
      call malloc_vec_d(len, vec)

      if (comm%node_me == comm%ionode) then
         write(*, *) "calling M_sumb_d for vector"
      end if
      call M_sumb_d(comm, vec, len)

      if (comm%node_me == comm%ionode) then
         write(*, *) "vector elemnet value in host = ", vec(1)
         !$omp target update from(vec)
         write(*, *) "vector elemnet value in device = ", vec(1)
      end if
      call dealloc_vec_d(vec)
      call mpi_sync()
   end subroutine test_m_sumb_d
```

对应 `malloc_vec_d` 函数内容如下。

```fortran
   subroutine malloc_vec_d(len, vec)
      use openmp
      implicit none

      integer, intent(in) :: len
      real(q), allocatable :: vec(:)

      allocate (vec(len))
      vec = comm%node_me
      !$omp target enter data map(to:vec)
   end subroutine malloc_vec_d
```

使用 Intel MPI 编译运行后，使用 4 个进程并行输出结果如下。

```
 =============test_m_sumb_d=============
 Testing vector length =           80
 Initializing vector with rank index =            1
 calling M_sumb_d for vector
 vector elemnet value in host =    1.00000000000000     
 vector elemnet value in device =    10.0000000000000     
```

可以看出，在调用 M_sumb_d 函数之后，host 端数组值仍为初始值 1。在将 device 端数组拷贝到 host 之后，对应值变为 10（1+2+3+4）。这说明在 MPI 通信时，成功对 device 端数据进行了更新。
