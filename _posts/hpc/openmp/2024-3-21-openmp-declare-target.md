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

## device 函数声明

下面例子中，Fortran函数fib包含declare target声明，从而告诉编译器构造一个device版本函数。函数名没有在declare target引语内，因此为隐式声明。当主函数引用use module_fib语句加载模块时，其同时引入了函数的显式接口，告知编译器fib包含declare target声明。

```fortran
!!%compiler: gfortran
!!%cflags: -fopenmp

! name: declare_target.1
! type: F-free
! version: omp_4.0
module module_fib
contains
   subroutine fib(N)
      integer :: N
      !$omp declare target
      !...
   end subroutine
end module
module params
integer :: THRESHOLD=1000000
end module
program my_fib
use params
use module_fib
   !$omp target if( N > THRESHOLD )
      call fib(N)
   !$omp end target
end program
```

当使用外部函数，未使用use module_fib函数引入函数接口时，需要为编译器提供函数的接口说明，并使用declare target引语。

```fortran
!!%compiler: gfortran
!!%cflags: -fopenmp

! name: declare_target.2
! type: F-free
! version: omp_4.0
program my_fib
integer :: N = 8
interface
  subroutine fib(N)
    !$omp declare target
    integer :: N
  end subroutine fib
end interface
   !$omp target
      call fib(N)
   !$omp end target
end program
subroutine fib(N)
integer :: N
!$omp declare target
     print*,"hello from fib"
     !...
end subroutine
```

## 全局变量卸载

模块内全局变量也可通过declare target引语卸载到device端。如下例子中，fortran模块内全局变量通过加入declare target引语列表内，实现映射至device端。

```fortran
!!%compiler: gfortran
!!%cflags: -fopenmp

! name: declare_target.3
! type: F-free
! version: omp_4.0
module my_arrays
!$omp declare target (N, p, v1, v2)
integer, parameter :: N=1000
real               :: p(N), v1(N), v2(N)
end module
subroutine vec_mult()
use my_arrays
   integer :: i
   call init(v1, v2, N);
   !$omp target update to(v1, v2)
   !$omp target
   !$omp parallel do
   do i = 1,N
     p(i) = v1(i) * v2(i)
   end do
   !$omp end target
   !$omp target update from (p)
   call output(p, N)
end subroutine
```

## 静态数据映射控制

在OpenMP 4.5标准中，扩展了declare target引语通过link关键字允许静态数据映射。在link参数内列举的变量仅通过map引语隐式或显式地映射后，才可在device端使用。

link引语用于无法将全局变量同时加载到device情况，同时变量不需要同时使用情况。

```fortran
!!%compiler: gfortran
!!%cflags: -fopenmp

! name: declare_target.6
! type: F-free
! version: omp_4.5
module m_dat
   integer, parameter :: N=100000000
   !$omp declare target link(sp,sv1,sv2)
   real :: sp(N), sv1(N), sv2(N)

   !$omp declare target link(dp,dv1,dv2)
   double precision :: dp(N), dv1(N), dv2(N)

contains
   subroutine s_vec_mult_accum()
   !$omp declare target
      integer :: i

      !$omp parallel do
      do i = 1,N
        sp(i) = sv1(i) * sv2(i)
      end do

   end subroutine s_vec_mult_accum

   subroutine d_vec_mult_accum()
   !$omp declare target
      integer :: i

      !$omp parallel do
      do i = 1,N
        dp(i) = dv1(i) * dv2(i)
      end do

   end subroutine
end module m_dat

program prec_vec_mult
   use m_dat

   call s_init(sv1, sv2, N)
   !$omp target map(to:sv1,sv2) map(from:sp)
     call s_vec_mult_accum()
   !$omp end target
   call s_output(sp, N)

   call d_init(dv1, dv2, N)
   !$omp target map(to:dv1,dv2) map(from:dp)
     call d_vec_mult_accum()
   !$omp end target
   call d_output(dp, N)

end program
```

# 编译器实现

## Intel OneAPI

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

# 参考文献

1. BOARD O A R. OpenMP Application Programming Interface Specification Version 5.1[M]. SUPINSKI B de, KLEMM M. Independently published, 2020:210.
2. Intel® Fortran Compiler Classic and Intel® Fortran Compiler Developer Guide and Reference [M]. 2024.0 edn. 2023:1347.
3. https://passlab.github.io/Examples/contents/Chap_devices/13_Declare_Target_Directive.html#declare-target-directive-for-variables

