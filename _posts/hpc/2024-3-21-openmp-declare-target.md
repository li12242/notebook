---
layout: post
title: OpenMP declare target
date: 2024-3-21
categories: hpc
---

# 外部函数和静态变量的编译

在使用OpenMP target引语指定代码在设备端运行时，编译器在编译过程中需要将代码翻译为可以在设备端运行的二进制版本。

现代fortran通过支持模块化，使得代码结构和功能更加清晰。但是在一些情况下，被调用函数和其调用过程可能会放在不同源代码文件中。这导致在编译一些函数的，编译器不知道这些代码是否会在设备端运行。

对于OpenMP target引语内调用的函数，如果编译器在编译卸载语句时可以看到函数定义，此时会将函数直接编译为设备端代码。当编译器无法直接看到函数定义时，需要使用declare target语句在代码中提供说明，在编译时为对应代码生成设备端代码。

# declare target使用

参考Intel Fortran编译器说明，OpenMP引语格式如下：
```fortran
!$omp declare target (extended-list)
```
或
```fortran
!$omp declare target [clause[[,]clause]...]
```
其中可选参数包括
```fortran
to(extended-list)
link(list)
device_type(host | nohost | any)
indirect[(invoked-by-fptr)]
```

- 在引语内如果没有`extended-list`内容，会隐式地添加`to`引语，其参数列表为被包含的subroutine或function名称。对于任何被包含在`to`参数内函数和变量，会编译为设备端版本代码。
- 此引语必须出现在作用域声明部分。

```fortran
SUBROUTINE ECCP_NL_OMP(LMDIM, LMMAXC, CDIJ, CPROJ1, CPROJ2, CNL)
   USE prec
   IMPLICIT NONE
   !$OMP DECLARE TARGET
   GDEF CNL
   INTEGER LMDIM, LMMAXC
   OVERLAP CDIJ(LMDIM, LMDIM)
   GDEF CPROJ1(LMMAXC), CPROJ2(LMMAXC)
   ! local
   INTEGER L, LP

!$OMP SIMD reduction(+:cnl) collapse(2)
   DO L = 1, LMMAXC
      DO LP = 1, LMMAXC
         CNL = CNL + CDIJ(LP, L)*CPROJ1(LP)*GCONJG(CPROJ2(L))
      END DO
   END DO
   
END SUBROUTINE ECCP_NL_OMP
```




