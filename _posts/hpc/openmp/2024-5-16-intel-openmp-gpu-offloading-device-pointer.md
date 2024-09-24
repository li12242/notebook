---
layout: post
title: Intel OpenMP GPU Offloading 设备指针 bug 分析
date: 2024-5-16
categories: hpc
---

# Intel Fortran OpenMP GPU Offloading 设备指针 bug 分析

## 错误现象

使用 Intel ifx 2024.0 编译器内 OpenMP GPU Offloading 功能时，使用 Fortran 指针执行卸载操作会出现数据错误。例如下面代码中，当使用指针 `p` 指向数组 `t`，并同时将数组和指针映射到 device 端。在 OpenMP 卸载语句中通过指针 p 对数组进行赋值操作后，先删除设备端映射指针，随后将数据从设备端拷贝回来后，发现结果数据错误。

```fortran
      integer, pointer :: p(:)
      integer, target :: t(len)
      integer :: i

      t = [(i, i = 1, len)]
      p => t

100   format(A12, 20I8)

      write (*, *)
      write (*, *) "================================================="
      write (*, *) "Initial static pointer"
      write (*, 100) "  t:", t

      !$omp target enter data map(to: t, p)
      !$omp target teams distribute parallel do
      do i = 1, len
         p(i) = len - i
      end do
      
      !$omp target exit data map(delete: p) 
      !$omp target exit data map(from: t)

      write (*, *) "Test device pointer with enter/exit data"
      write (*, 100) "  t:", t
      !=================================================================
```

运行结构如下：
```
 =================================================
 Initial static pointer
          t:       1       2       3       4       5       6       7       8       9      10
 Test device pointer with enter/exit data
          t:       1       2       3       4       5       6       7       8       9      10
```

当我们换另外一种映射语法，使用 structured map 引语 `target data/end target data` 时，可以得到正确结果。对应代码和运行结果如下：
```fortran
      !$omp target enter data map(to: t)
      !$omp target data map(to: p)
      !$omp target teams distribute parallel do
      do i = 1, len
         p(i) = len - i
      end do

      !$omp end target data
      !$omp target exit data map(from: t)

      write (*, *) "Test device pointer with target data"
      write (*, 100) "  t:", t
      write (*, *) "================================================="
```

```
 =================================================
 Initial static pointer
          t:       1       2       3       4       5       6       7       8       9      10
 Test device pointer with enter/exit data
          t:       1       2       3       4       5       6       7       8       9      10
 Test device pointer with target data
          t:       9       8       7       6       5       4       3       2       1       0
 =================================================
```

## 问题分析

解决 bug 需要对问题更深入的认知。对于以上 bug，判断问题出现原因需要更详细地测试从而解释以下几个问题：

1. 指针通过 target enter/exit data 引语映射到 device 端后，指针是否仍然指向 data 数据？
2. 上述错误代码中通过 map 引语得到的错误数据是哪里来的？

## 测试

### 测试1

首先通过数据传递判断下 device 端指针对数据修改是否正确。定义一个临时数组 `target_data`，当使用指针对数据修改后，通过临时数组拷贝对象数组值。对应代码如下所示：

```fortran
      integer :: target_data(len)

      !$omp target enter data map(to: target_data)

      ......

      !$omp target
      target_data = t
      !$omp end target

      !$omp target exit data map(from: target_data)        

      write (*, *) "Test device pointer with enter/exit data"
      write (*, 100) "  t:", t
      write (*, 100) "  target_data:", target_data          
```

测试结果如下：
```
 =================================================
 Initial static pointer
          t:       1       2       3       4       5       6       7       8       9      10
 Test device pointer with enter/exit data
          t:       1       2       3       4       5       6       7       8       9      10
  target_dat       9       8       7       6       5       4       3       2       1       0
```

可以看出，临时数组获得了修改后的正确数据，并且由于 `target_data` 在 deivce 端是直接拷贝数组 `t`，而赋值操作是通过指针 `p` 进行的，说明指针指向数据是正确的。

### 测试2

既然指针指向对象没有问题，那么下面就需要看下为何拷贝数据出现错误。

首先看下此错误数据是什么数据，这个是原始数据？是从 device 端上数组 `t` 拷贝下来的吗？判断方法非常简单，在使用 map 引语将 deivce 端指针和数据卸载前，修改 host 端数组值即可，对应代码如下。

```fortran
      !$omp target enter data map(to: t, p)

      !$omp target teams distribute parallel do
      do i = 1, len
         p(i) = len - i
      end do

      t = 0         ! 修改 host 端数组值

      !$omp target exit data map(delete: p) 
      !$omp target exit data map(from: t)
```

获得输出结果如下：
```
 =================================================
 Initial static pointer
          t:       1       2       3       4       5       6       7       8       9      10
 Test device pointer with enter/exit data
          t:       0       0       0       0       0       0       0       0       0       0
```

从这个结果可以看出，此数值并非是从 device 端获得的，而一直是 host 端数组 `t` 的值，也就是最后的 map 引语并没有起作用。

### 测试3

可以看出最后 `map(from: t)` 引语操作并没有执行。那么此问题出现原因可能就是 ifx 错误认为 device 端指针 `p` 和数组 `t` 是同一个数据，当调用 `map(delete: p)` 时对应指针指向数据 `t` 也被删除掉，而后面再次调用 `map(from: t)` 不再起任何作用。

根据上述推论，在代码中将 `map(delete: p)` 修改为 `map(from: p)`，此时对应输出结果正确。
```
 =================================================
 Initial static pointer
          t:       1       2       3       4       5       6       7       8       9      10
 Test device pointer with enter/exit data
          t:       9       8       7       6       5       4       3       2       1       0
```

## 结论

从上面两个测试可以看出，在 Intel ifx 编译器使用过程中，Fortran 指针和目标数组是可以被正确映射到 device 端进行操作的。在操作结束后删除数据时，ifx 可能是错误的将 device 端指针 p 和数组 t 认为是同一个数据。在调用 `map(delete: p)` 引语删除指针时，此时错误认为 deivce 端指针指向数据需要删除掉，在后面再调用 `map(from: t)` 时可能数据已经在 device 端释放，导致此引语并没有执行。

