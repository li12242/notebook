---
layout: post
title: OpenMP declare variant 与 dispatch
date: 2024-3-21
categories: hpc
---

# OpenMP 设备版本函数执行

在调用Intel MKL GPU版本函数时，通过使用`!$OMP TARGET VARIANT DISPATCH`引语即可调用对应设备端函数在GPU计算。参考Intel Fortran编译器手册，关于此功能主要和`target variant dispatch`和`declare variant`引语有关。

# OpenMP 引语作用

## TARGET VARIANT DISPATCH

此引语格式为：
```fortran
!$OMP TARGET VARIANT DISPATCH
call subr (...)
!$OMP END TARGET VARIANT DISPATCH
```

此引语使编译器有条件第执行设备端版本函数。当默认设备可用，或者引语中加入了`DEVICE(integer-expression)`参数，同时指定设备可以执行时，会执行设备端版本二进制代码，否则会执行默认主机端二进制版本函数。

引语`TARGET VARIANT DISPATCH`使用的函数名必须在`DECLARE VARIANT`引语中出现。设备端版本函数接口必须可以被基函数所访问。设备端版本函数必须与基函数有相同接口，只是在其输入参数最后增加`C_PTR`类型指针。这个参数并不会出现在`TARGET VARIANT DISPATCH`引语调用过程中，仅是在编译器选择设备版本函数时所使用。

代码示例

```fortran
MODULE vecadd 
    INTEGER,PARAMETER :: n = 1024 
CONTAINS 
    FUNCTION vecadd_gpu_offload (ptr) RESULT (res) 
        USE,INTRINSIC :: ISO_C_BINDING, ONLY : c_ptr 
        !$DEC ATTRIBUTES NOINLINE :: vecadd_gpu_offload 
        TYPE(c_ptr) :: ptr
        REAL :: res 
        REAL,DIMENSION(n) :: a, b 
        INTEGER :: k 
        
        res = a(k) + b(k) 
        PRINT *, "GPU version of vecadd called"
    END FUNCTION vecadd_gpu_offload

    FUNCTION vecadd_base () RESULT (res) 
        !$DEC ATTRIBUTES NOINLINE :: vecadd_base 
        !$OMP DECLARE VARIANT (vecadd_gpu_offload) match(construct={target variant dispatch}& !$OMP& ,device = {arch (gen)} ) 
        REAL :: res 
        REAL,DIMENSION(n) :: a, b 
        INTEGER :: k 
        
        res = a(k) + b(k) 
        PRINT *, "CPU version of vecadd called" 
    END FUNCTION vecadd_base 
END MODULE vecadd

PROGRAM main 
    USE vecadd 
    REAL :: result = 0.0 
    
    !$OMP TARGET VARIANT DISPATCH 
    result = vecadd_base () 
    !$OMP END TARGET VARIANT DISPATCH 
    
    IF (result == 1048576.0) then 
        PRINT *, "PASSED: correct results" 
    ELSE 
        PRINT *, "FAILED: incorrect results" 
    ENDIF 
END PROGRAM
```

## DECLARE VARIANT

在声明设备端版本函数时，需要使用`DECLARE VARIANT`引语在基函数内指定变种函数接口，其语法格式为
```fortran
!$OMP DECLARE VARIANT ([base-proc-name:]variant-proc-name) clause[[[,]clause]...]
```
引语`DECLARE VARIANT`必须出现在函数说明部分，或接口块的接口中。当调用时匹配`MATCH`引语中指定的上下文匹配时，对应的设备版本函数将会被调用。

在上面示例中，`DECLARE VARIANT`引语中定义了`match(construct={target variant dispatch}& `，因此在调用时只有`TARGET VARIANT DISPATCH`引语调用基函数时，才会转换为对应设备端版本函数。
