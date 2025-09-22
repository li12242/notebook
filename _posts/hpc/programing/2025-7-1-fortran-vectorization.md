---
layout: post
title: Fortran向量化编程
date: 2025-7-1
categories: hpc
---

## 概述

本文档提供了 Fortran 中各种数组类型的示例及其作为局部变量、函数/子程序参数的用法，以及 Fortran 指针及其用法的示例。本文档展示了编译器如何对各种数组数据类型和参数进行向量化处理。这包括高级源代码示例，并解释了编译器针对这些情况生成的代码。

## Fortran 数组数据、参数传递与向量化示例

现代 Fortran 语言拥有多种类型的数组以及数组切片功能，允许将数组的子部分作为参数传递给函数或被 Fortran 指针指向。对于操作数组的循环，编译器可以生成单位步长（unit stride）向量代码、非单位步长代码，或者生成多版本代码（在运行时决定执行哪个版本）。编译器为循环生成的代码取决于程序员选择的数组类型、是否对该数组使用了切片、以及数组是否作为函数参数传递。

为了使用 SIMD（单指令多数据）指令，具有非单位步长内存访问的代码会使用收集/散布（`gather`/`scatter`）指令（如 `vgather`/`vscatter`）。具有单位步长内存访问的代码可以使用效率高得多的加载和存储指令。根据目标内存地址的对齐情况，具有单位步长内存访问的代码可以使用对齐加载/存储指令（如 `vmovaps` 或其等效指令）或非对齐加载/存储指令（如 `vloadunpack`/`vpackstore`）。

在编译器生成单位步长向量化代码的情况下，让数组对齐以实现良好性能（即使用对齐加载/存储指令）也非常重要。程序员可以使用编译器选项（如 `-align array64byte`）或指令（如 `!dir$ attribute align`）指示编译器为数组分配对齐的内存。然而，这对于为循环生成对齐的向量代码来说是不够的。程序员还应使用指令（如 `!dir$ vector aligned`, `!dir$ assume_aligned`）告诉编译器，在每个循环中哪些数组将以对齐方式访问。本指南在标题为“数据对齐以辅助向量化”的文章中包含了关于对齐的详细信息。

对于包含未对齐数组的循环，编译器通常会生成一个“剥离循环”（peel loop）。剥离循环位于主循环之前，对数组的数据元素进行操作，直到其中一个数组对齐为止（向量化的剥离循环使用收集/散布指令）。剥离循环之后，一个使用该数组对齐内存访问的“内核循环”（kernel loop）在已对齐的数据上操作。在剥离循环和内核循环之后，一个“余数循环”（remainder loop）在内核循环结束点开始，对数组末尾剩余的任何未对齐数据进行操作，直到完成所有迭代（向量化的余数循环使用收集/散布指令）。对于内核循环，编译器可以进一步生成针对第二个数组对齐性的动态检查，并生成多版本循环（一个循环对该数组使用对齐访问，另一个使用非对齐访问）。

### 显式形状数组 (Explicit Shape Arrays)

对于参数为显式形状数组的子程序，编译器在子程序/函数中生成的代码假定这些参数的数据是连续的（contiguous）。

调用过程：编译器为调用此子程序自动生成的代码将确保传递的数组数据是连续的。如果实参（actual argument）确实是连续的或是连续的数组切片，则将指向实际数组或连续数组切片的指针作为参数传递。如果数据是非连续的（例如非连续数据的数组切片），调用者将生成一个数组临时变量（array temporary），将非连续的源数据收集（gather）到这个连续的数组临时变量中，并传递一个指向这个连续临时数据的指针。实际的子程序/函数调用使用这些临时数组作为参数。当调用返回时，调用代码再将临时数据解包（scatter）回原始的非连续源数组切片，并销毁临时数组。这会带来数组临时变量的开销，以及收集和散布操作所需的内存移动开销。

示例1.1： 作为参数传递的显式形状数组。

```fortran
subroutine explicit1(A, B, C)
  real, intent(in), dimension(400,500) :: A
  real, intent(out), dimension(500) :: B
  real, intent(inout), dimension (400) :: C
  integer i

  !..loop 1
  do i=1,500
      B(i) = A(3,i)
  end do

  !..loop 2
  do i=1,400
      C(i) = C(i) + A(i, 400)
  end do
end subroutine explicit1
```

对于循环1：

* 源代码中 B 是单位步长访问，A 是非单位步长访问。
* 循环被剥离直到 B 对齐。
* 剥离循环被向量化，但对加载 A（未对齐）使用 vgather，对存储 B（未对齐）使用 vscatter。
* 内核循环对存储 B（对齐）使用 vmovaps，对加载 A（未对齐）使用 vgather。
* 余数循环被向量化：对存储 B（对齐）使用 vmovaps，对加载 A（未对齐）使用 vgather。

对于循环2：

* 源代码在加载 A 和加载/存储 C 上都是单位步长访问。
* 循环被剥离直到存储 C 对齐。剥离循环被向量化。
* 加载 A 可能对齐也可能未对齐。编译器为内核循环生成多版本代码：
  * 版本1：对加载 A（未对齐）使用 loadunpack，对加载 C（对齐）使用 vaddps，对存储 C（对齐）使用 vmovaps。
  * 版本2：对加载 A（对齐）使用 vmovaps，对加载 C（对齐）使用 vaddps，对存储 C（对齐）使用 vmovaps。
* 余数循环被向量化。

示例 1.2： 作为参数传递的对齐显式形状数组。

```fortran
subroutine explicit2(A, B, C)
  real, intent(in), dimension(400,500) :: A
  real, intent(out), dimension(500) :: B
  real, intent(inout), dimension(400) :: C
  !dir$ assume_aligned A(1,1):64 ! 假设 A(1,1) 64字节对齐
  !dir$ assume_aligned B(1):64   ! 假设 B(1) 64字节对齐
  !dir$ assume_aligned C(1):64   ! 假设 C(1) 64字节对齐

  integer i

  !..loop 1
  do i=1,500
      B(i) = A(3,i)
  end do

  !..loop 2
  do i=1,400
      C(i) = C(i) + A(i, 400)
  end do
end subroutine explicit2
```

`assume_aligned` 指令用于声明这三个数组是对齐的。此指令提供的信息在子程序内的两个循环中都有效。

循环 1:

* 不需要剥离循环，因为数组已经对齐。
* 内核循环使用 vmovaps 存储 B 和使用 vgatherd 加载 A 进行向量化。
* 余数循环只有 4 次迭代（在编译时已知）。由于向量化效率不高，因此未向量化（可能是标量执行）。

循环 2:

* 不需要剥离循环。
* 内核循环使用 vmovaps 进行所有三次内存访问（两次加载 A 和 C，一次存储 C）进行向量化。
* 不需要余数循环（400*4 是 64 的倍数）。

示例 1.3： 作为参数传递的对齐显式形状数组（另一种方法）。

```fortran
subroutine explicit3(A, B, C)
  real, intent(in), dimension(400,500) :: A
  real, intent(out), dimension(500) :: B
  real, intent(inout), dimension(400) :: C
  integer i

  !dir$ vector aligned ! 向量循环，假设所有内存访问对齐
  do i=1,500
      B(i) = A(3,i)
  end do

  !dir$ vector aligned ! 向量循环，假设所有内存访问对齐
  do i=1,400
      C(i) = C(i) + A(i, 400)
  end do
end subroutine explicit3
```

此方法与上例中的方法相同。`!dir$ vector aligned` 指令指示给定循环中的所有内存访问都是对齐的。此指令需要在两个循环中重复使用。

### 可变数组 (Adjustable Arrays)

与我们刚刚描述的显式形状数组类似，为参数为可调数组的子程序/函数生成的代码假设这些参数是连续的（具有单位步长）。在调用例程的调用点（call sites），一个更大数组的切片可能已作为参数传递给被调用过程。因此，在调用点，编译器会生成代码，在调用前将有步长的数组参数打包（pack）到连续的临时数组中，并在调用后解包（unpack）回原始步长的位置（将实际数据收集到/散布自临时数组）。实际的子程序/函数调用使用这些临时数组作为参数。这会带来数组临时变量的开销，以及收集和散布操作所需的内存移动开销。

示例 2.1： 作为参数传递的 1D 可调数组。

```fortran
subroutine adjustable1(Y, Z, N)
  real, intent(inout), dimension(N) :: Y
  real, intent(in), dimension(N) :: Z
  integer, intent(in) :: N

  Y = Y + Z

  return
end subroutine adjustable1
```

* 循环被剥离直到存储 Y 对齐。
* 内核循环中加载 Z 可能对齐也可能未对齐。编译器生成多版本代码：
  * 版本 1：对加载 Z（未对齐）使用 loadunpack，对加载 Y（对齐）使用 vaddps，对存储 Y（对齐）使用vmovaps。
  * 版本 2：对加载 Z（对齐）使用 vmovaps，对加载 Y（对齐）使用 vaddps，对存储 Y（对齐）使用 vmovaps。
* 余数循环被向量化。

示例 2.2： 作为参数传递的 2D 可调数组。

```fortran
subroutine adjustable2(Y, Z, M, N)
  real, intent(inout), dimension(M, N) :: Y
  real, intent(in), dimension(M, N) :: Z
  integer, intent(in) :: M
  integer, intent(in) :: N

  Y = Y + Z

  return
end subroutine adjustable2
```

* 对应于两个数组维度的两个循环被编译器折叠（collapse）成一个循环。
* 循环被剥离直到存储 Y 对齐。
* 内核循环中加载 Z 可能对齐也可能未对齐。编译器生成多版本代码：
  * 版本 1：对加载 Z（非对齐？原文此处写aligned，但上下文应为unaligned？）使用 vloadunpack，对加载 Y（对齐）使用 vaddps，对存储 Y（对齐）使用 vmovaps。
  * 版本 2：对加载 Z（对齐）使用 vmovaps，对加载 Y（对齐）使用 vaddps，对存储 Y（对齐）使用 vmovaps。
* 余数循环被向量化。

示例 2.3： 作为参数传递的对齐 1D 可调数组。

```fortran
subroutine adjustable3(YY, ZZ, NN)
  real, intent(inout), dimension(NN) :: YY
  real, intent(in), dimension(NN) :: ZZ
  integer, intent(in) :: NN
  !dir$ assume_aligned YY:64, ZZ:64 ! 假设YY, ZZ 64字节对齐

  YY = YY + ZZ
  return
end subroutine adjustable3
```

* `assume_aligned` 指令用于告诉编译器，在此函数中，YY 和 ZZ 指向 64 字节对齐的位置。
* 不需要剥离循环，因为 YY 和 ZZ 是对齐的。
* 内核循环被向量化，使用 vmovaps 加载 ZZ（对齐），使用 vaddps 加载 YY（对齐），使用 vmovaps 存储 YY（对齐）。
* 余数循环被向量化（使用非对齐访问指令）。

示例 2.4： 对齐 2D 可调数组示例。

```fortran
subroutine adjustable4(YY, ZZ, MM, NN)
    real, intent(inout), dimension(MM, NN) :: YY
    real, intent(in), dimension(MM, NN) :: ZZ
    integer, intent(in) :: MM
    integer, intent(in) :: NN
    !dir$ assume_aligned YY:64, ZZ:64 ! 假设YY, ZZ 64字节对齐

    YY = YY + ZZ

    return
end subroutine adjustable4
```

* `assume_aligned` 指令用于告诉编译器，在此函数中，YY 和 ZZ 指向 64 字节对齐的位置。
* 两个循环被编译器折叠成一个循环。
* 不需要剥离循环，因为 YY 和 ZZ 是对齐的。
* 内核循环被向量化，使用 vmovaps 加载 ZZ（对齐），使用 vaddps 加载 YY（对齐），使用 vmovaps 存储 YY（对齐）。
* 余数循环被向量化（使用非对齐访问指令）。

示例 2.5： 作为参数传递的对齐 1D 可调数组（另一种方法）。

```fortran
subroutine adjustable5(YY, ZZ, NN)
  real, intent(inout), dimension(NN) :: YY
  real, intent(in), dimension(NN) :: ZZ
  integer, intent(in) :: NN

  !dir$ vector aligned ! 向量循环，假设所有内存访问对齐
  YY = YY + ZZ
  return
end subroutine adjustable5
```

* `vector aligned` 指令用于告诉编译器，在此函数中，YY 和 ZZ 是对齐的。
* 不需要剥离循环，因为 YY 和 ZZ 被声明为对齐。
* 内核循环被向量化，使用 vmovaps 加载 ZZ（对齐），使用 vaddps 加载 YY（对齐），使用 vmovaps 存储 YY（对齐）。
* 余数循环被向量化（使用非对齐访问指令）。

### 假定形状数组 (Assumed Shape Arrays)

当假定形状数组用作过程参数时，与上面的可调数组示例不同，数组的大小不是由程序员作为显式参数传递的。此外，为函数/子程序生成的代码不假设传递的参数是连续的（单位步长）。参数可以是非连续的（有步长的，例如其他数组的切片）。在这种情况下，编译器不会在调用点前后生成将有步长数组打包到/解包自临时连续数组的代码。相反，它生成的代码使得对于每个数组的每个维度，下/上边界和步长信息与数组基地址一起从调用者传递给被调用者。然后，在被调用者上生成的代码使用这些边界和步长，从而消除了在调用者处进行任何打包/解包的需要。这对于调用和返回来说更高效。

当调用者和被调用者需要分开编译时，为了使编译器知道要调用的函数具有假定形状数组（因此不需要打包/解包但需要传递边界和步长信息），调用者一侧需要有一个接口声明（interface declaration）（除非被调用者是内部过程，此时接口将是隐式的）。此接口将相应的函数参数声明为假定形状数组。请注意，单位步长向量化代码的性能（在数组在运行时实际上是连续的情况下适用）优于非单位步长向量化代码。

您可以使用 Fortran 2008 的 CONTIGUOUS 属性与假定形状数组一起使用，以告诉编译器数据占据一个连续的内存块。这允许编译器对假定形状数组进行优化。此属性将在本文后面的部分更详细地描述。请注意，连续和非连续数组都可以在不打包/解包的情况下传递给这些子程序/函数。因此，编译器在编译被调用子程序时，不能盲目地为假定形状数组生成单位步长代码。相反，在大多数此类情况下，编译器会生成多版本代码：

1. 一个向量化版本，它假设所有假定形状数组都是以单位步长访问的。
2. 一个向量化版本，它假设所有假定形状数组都是以非单位步长访问的。在过程内部运行时，会检查所有数组的步长，并选择向量化区域的正确版本执行。

请注意，第二种情况是保守的回退情况。如果只有一个（可能在多个中）实际的假定形状数组参数是非单位步长的，那么无论其余参数是否是单位步长，都会执行所有数组都使用非单位步长向量化的第二个版本。

示例 3.1： 作为参数的 1D 假定形状数组。

```fortran
subroutine assumed_shape1(Y)
  real, intent(inout), dimension(:) :: Y

  Y = Y + 1

  return
end subroutine assumed_shape1
```

为数组步长生成了多版本代码：

* 版本 1：假设 Y 是单位步长的向量代码。
  * 剥离循环直到 Y 对齐（使用收集/散布）。
  * 内核循环使用 vaddps 加载 Y（对齐），使用 vmovaps 存储 Y（对齐）。
  * 余数循环被向量化（使用收集/散布）。
* 版本 2：假设 Y 是非单位步长的向量代码。
  * 无剥离循环。
  * 内核循环使用 vgatherd 加载 Y，使用 vscattered 存储 Y。
  * 余数循环是标量的。

示例 3.2： 作为参数的 2D 假定形状数组。

```fortran
subroutine assumed_shape2(Z)
  real, intent(inout), dimension(:,:) :: Z

  Z = Z + 1

  return
end subroutine assumed_shape2
```

* 两层深的循环嵌套未被折叠成一维循环。外层循环已经是非单位步长，未被向量化。
* 为内部循环的数组步长生成了两个版本：
  * 版本 1：假设 Z 在第一维（内循环维度）是单位步长的向量代码。
    * 剥离循环直到 Z 对齐（使用收集/散布）。
    * 内核循环使用 vaddps 加载 Z（对齐），使用 vmovaps 存储 Z（对齐）。
    * 余数循环被向量化（使用收集/散布）。
  * 版本 2：假设 Z 在第一维（内循环维度）是非单位步长的向量代码。
    * 无剥离循环。
    * 内核循环使用 vgather 加载 Z，使用 vscatter 存储 Z。
    * 两个余数循环：使用半向量大小的收集/散布和一个标量循环。

示例 3.3： 两个 1D 假定形状数组作为参数。

```fortran
subroutine assumed_shape3(A, B)
  real, intent(out), dimension(:) :: A
  real, intent(in), dimension(:) :: B

  A = B + 1

  return
end subroutine assumed_shape3
```

为数组步长生成了多版本代码：
* 版本 1：A 和 B 都是单位步长的向量化版本。
  * 剥离循环直到 A 对齐。
  * 内核循环还有自己的多版本代码：
    * 版本 1a：假设 B 是对齐的。
    * 版本 1b：假设 B 是未对齐的。
    * 余数循环被向量化。
* 版本 2：A 和 B 都不是单位步长的向量代码。如果至少有一个数组是非单位步长，则进入此情况。
    * 对所有数组使用收集和散布。
    * 余数循环是标量的。

```fortran
subroutine assumed_shape4(A, B, C)
  real, intent(out), dimension(:) :: A
  real, intent(in), dimension(:) :: B
  real, intent(in), dimension(:) :: C
  A = B + C

  return
end subroutine assumed_shape4
```

为数组步长生成了多版本代码：

* 版本 1：A, B 和 C 都是单位步长的向量化版本。
  * 剥离循环直到 A 对齐。
  * 内核循环有多版本：
    * 版本 1a：假设 B 对齐。假设 C 未对齐。
    * 版本 1b：假设 B 未对齐。假设 C 未对齐。
    * 注意：为了防止多版本嵌套过深，没有为 C 的对齐创建第三级多版本。
* 版本 2：所有数组都是非单位步长的向量代码。如果至少有一个数组是非单位步长，则进入此情况。
  * 对所有数组使用收集/散布。
  * 余数循环是标量的。

示例 3.5： 三个对齐 1D 假定形状数组作为参数。

```fortran
subroutine assumed_shape5(A, B, C)
  real, intent(out), dimension(:) :: A
  real, intent(in), dimension(:) :: B
  real, intent(in), dimension(:) :: C
  !dir$ assume_aligned A:64, B:64, C:64 ! 假设A, B, C 64字节对齐

  A = B + C

  return
end subroutine assumed_shape5
```

`assume_aligned` 指令用于指示所有三个数组都是对齐的。

为步长问题生成了多版本代码：

* 版本 1：A, B, C 都是单位步长的向量代码。
  * 不需要剥离循环，因为数组已声明对齐。
  * 内核循环不需要多版本，因为所有数组已被断言为对齐。
  * 余数循环使用收集/散布。
* 版本 2：没有数组是单位步长的向量代码。
  * 使用收集/散布。（数组是否对齐无关紧要）
  * 余数循环是标量的。

### 假定大小数组 (Assumed Size Arrays)

为参数为假定大小数组的子程序/函数生成的代码假设这些参数是单位步长的。与上面解释的传递可调数组作为参数类似，在每个调用点，编译器会生成代码，在调用前将任何非单位步长的数组参数打包（pack）成单位步长的临时数组，并在调用后解包（unpack）回其原始有步长的位置。

示例 4.1： 作为参数传递的 1D 假定大小数组。

```fortran
subroutine assumed_size1(Y)
    real, intent(inout), dimension(*) :: Y ! 假定大小数组
    ! 上界由程序员已知并硬编码
    ! Y(:) = Y(:) + 1 => 非法，因为编译器不知道 Y 的大小

    Y(1:500) = Y(1:500) + 1 ! 显式指定索引范围
    return
end subroutine assumed_size1
```

* 循环被剥离直到 Y 对齐（使用收集/散布）。
* 内核循环使用 vaddps 加载 Y（对齐）和 vmovaps 存储 Y（对齐）进行向量化。
* 余数循环使用收集/散布进行向量化。

```fortran
subroutine assumed_size2(Y)
  real, intent(inout), dimension(20,*) :: Y ! 假定大小数组
  ! 内部维度大小（即 20）提供给编译器，因此它可以生成相应的代码。
  ! 外部维度对编译器未知，因此它必须是程序员给出的常量或常量范围。
  ! Y(:,:) => 非法，因为 Y 的第二维大小未知。

  Y(:,1:10) = Y(:,1:10) + 1 ! 显式指定索引范围

  return
end subroutine assumed_size2
```

* 循环被剥离直到 Y 对齐（使用收集/散布）。
* 内核循环使用 vaddps 加载 Y（对齐）和 vmovaps 存储 Y（对齐）进行向量化。
* 余数循环使用收集/散布进行向量化。

示例 4.3： 作为参数传递的对齐 1D 假定大小数组。

```fortran
subroutine assumed_size3(Y)
    real, intent(inout), dimension(*) :: Y ! 假定大小数组
    !dir$ assume_aligned Y:64 ! 假设Y 64字节对齐

    Y(1:500) = Y(1:500) + 1 ! 显式指定索引范围

    return
end subroutine assumed_size3
```

* `assume_aligned` 指令用于告诉编译器，在此子程序中，Y 是 64 字节对齐的。
* 不需要剥离循环，Y 已经对齐。
* 内核循环使用 vaddps 加载 Y（对齐）和 vmovaps 存储 Y（对齐）进行向量化。
* 余数循环只有 4 次迭代，因此未进行向量化（效率不高）。它被完全展开（unrolled）并以标量方式执行。

示例 4.4： 作为参数传递的对齐 1D 假定大小数组（另一种方法）。

```fortran
subroutine assumed_size4(Y)
  real, intent(inout), dimension(*) :: Y ! 假定大小数组
  !dir$ vector aligned ! 向量循环，假设所有内存访问对齐

  Y(1:500) = Y(1:500) + 1 ! 显式指定索引范围

  return
end subroutine assumed_size4
```

* `vector aligned` 指令用于告诉编译器，在此子程序中，Y 是对齐的。
* 结果与上例相同。

### 可分配数组 (Allocatable Arrays)

为参数为可分配数组的子程序/函数生成的代码假设这些参数是单位步长的。与上面解释的传递可调数组作为参数类似，在每个调用点，编译器会生成代码，在调用前将任何非单位步长的数组参数打包（pack）成单位步长的临时数组，并在调用后解包（unpack）回其原始有步长的位置。

示例 5.1： 作为局部变量的 1D 可分配数组。

```fortran
subroutine allocatable_array1(N)
    integer N
    real, allocatable :: Y(:)

    allocate (Y(N)) ! 分配内存
    !..loop 1
    Y = 1.2 ! 初始化赋值
    call dummy_call() ! 可能影响状态的调用

    !..loop 2
    Y = Y + 1 ! 计算赋值
    call dummy_call() ! 可能影响状态的调用

    deallocate(Y) ! 释放内存
    return
end subroutine allocatable_array1
```

循环 1:

* 循环被剥离直到 Y 对齐。剥离循环使用 vscatter 进行向量化。
* 内核循环使用 vmovaps（对齐存储）进行向量化。
* 余数循环使用 vscatter 进行向量化。

循环 2:

* 循环被剥离直到 Y 对齐。剥离循环使用 vgather/vscatter 进行向量化。
* 内核循环使用 vaddps 加载 Y（对齐）和 vmovaps 存储 Y（对齐）进行向量化。
* 余数循环使用 vgather/vscatter 进行向量化。

示例 5.2： 作为局部变量的 2D 可分配数组。

```fortran
subroutine allocatable_array2(N)
    integer N
    real, allocatable :: Z(:,:)

    allocate(Z(N, N)) ! 分配内存
    !..loop 1
    Z = 2.3 ! 初始化赋值
    call dummy_call() ! 可能影响状态的调用

    !..loop 2
    Z = Z + 1 ! 计算赋值
    call dummy_call() ! 可能影响状态的调用

    deallocate(Z) ! 释放内存
    return
end subroutine allocatable_array2
```

循环 1:

* 循环被剥离直到 Z 对齐。剥离循环使用 vscatter 进行向量化。
* 内核循环使用 vmovaps 存储 Z（对齐）进行向量化。
* 余数循环使用 vscatter 进行向量化。

循环 2:

* 循环被剥离直到 Z 对齐。剥离循环使用 vgather/vscatter 进行向量化。
* 内核循环使用 vaddps 加载 Z（对齐）和 vmovaps 存储 Z（对齐）进行向量化。
* 余数循环使用 vgather/vscatter 进行向量化。

示例 5.3： 作为参数传递的 1D 可分配数组。

```fortran
subroutine allocatable_array3(Y)
    real, intent(inout), allocatable, dimension(:) :: Y
    !..loop 1
    Y = 1.2 ! 初始化赋值
    call dummy_call() ! 可能影响状态的调用

    !..loop 2
    Y = Y + 1 ! 计算赋值
    call dummy_call() ! 可能影响状态的调用

    return
end subroutine allocatable_array3
```

循环 1:

* 循环被剥离直到 Y 对齐。剥离循环使用 vscatter 进行向量化。
* 内核循环使用 vmovaps 存储 Y（对齐）进行向量化。
* 余数循环使用 vscatter 进行向量化。

循环 2:

* 循环被剥离直到 Y 对齐。剥离循环使用 vgather/vscatter 进行向量化。
* 内核循环使用 vaddps 加载 Y（对齐）和 vmovaps 存储 Y（对齐）进行向量化。
* 余数循环使用 vgather/vscatter 进行向量化。

示例 5.4： 作为参数传递的 2D 可分配数组。

```fortran
subroutine allocatable_array4(Z) ! 注意：原文示例名称为5，但内容应为4
    real, intent(inout), allocatable, dimension(:,:) :: Z
    !..loop 1
    Z = 2.3 ! 初始化赋值
    call dummy_call() ! 可能影响状态的调用

    !..loop 2
    Z = Z + 1 ! 计算赋值
    call dummy_call() ! 可能影响状态的调用

    return
end subroutine allocatable_array4
```

与上例相同。剥离循环使用 `vgather`/`vscatter`。内核循环使用对齐访问。余数循环使用 `vgather`/`vscatter`。

示例 5.5： 作为参数传递的对齐 1D 可分配数组。

```fortran
subroutine allocatable_array5(Y) ! 注意：原文示例名称为6，但内容应为5
    real, intent(inout), allocatable, dimension(:) :: Y
    !dir$ assume_aligned Y:64 ! 假设Y 64字节对齐
    Y = 1.2
    call dummy_call()
    Y = Y + 1
    call dummy_call()

    return
end subroutine allocatable_array5
```

（原文提示参考）另见 Intel Fortran Compiler 中的 Fortran 可分配数组和指针的对齐。

示例 5.6： 作为参数传递的对齐 1D 可分配数组（另一种方法）。

```fortran
subroutine allocatable_array6(Y) ! 注意：原文示例名称为6，与上例同名，内容不同
    real, intent(inout), allocatable, dimension(:) :: Y
    !dir$ vector aligned ! 向量循环，假设内存访问对齐 (用于第一个循环)

    Y = 1.2
    call dummy_call()
    !dir$ vector aligned ! 向量循环，假设内存访问对齐 (用于第二个循环)
    Y = Y + 1
    call dummy_call()

    return
end subroutine allocatable_array6
```

* vector aligned 指令用于为两个循环生成对齐的内存访问。
* 不需要剥离循环，因为程序员告诉编译器 Y 是对齐的。（程序员需要确保 Y 实际对齐）
* 内核循环使用对齐的加载/存储进行向量化。
* 余数循环被向量化（使用非对齐访问指令）。

### 指针 (Pointers)

Fortran 指针可以指向一个在多个维度中可能具有非单位步长的数组切片。因此，为具有指针参数的例程生成的向量代码是多版本的：

1. 单位步长向量化版本
2. 非单位步长向量化版本。此外，还可以为数组的对齐生成多版本代码。

在指针赋值的情况下，步长信息（单位步长或非单位步长）会被编译器用来消除两个版本中的一个，因为它永远不会被执行。然而，当存在函数调用时，这个步长信息会丢失，因为函数可以修改指针并使当前的步长信息无效。

您可以使用 Fortran 2008 的 CONTIGUOUS 属性与指针数组一起使用，以告诉编译器数据占据一个连续的内存块。此属性将在本文后面的部分更详细地描述。

示例 6.1： 作为参数传递的 1D 指针。

```fortran
subroutine pointer1(Y)
    real, intent(inout), pointer, dimension(:) :: Y
    Y = Y + 1
    return
end subroutine pointer1
```

创建了两个版本：

* 版本 1：使用单位步长访问的向量化。
  * 循环被剥离直到 Y 对齐（使用 vgather/vscatter）。
  * 内核循环使用 vmovaps（对齐访问）进行向量化。
  * 余数循环使用 vscatter 进行向量化。
* 版本 2：使用非单位步长访问的向量化。使用 vgather/vscatter。

示例 6.2： 三个 1D 指针作为参数传递。

```fortran
subroutine pointer2(A, B, C)
    real, intent(inout), pointer, dimension(:) :: A, B, C
    A = A + B + C + 1
    return
end subroutine pointer2
```

循环被分成两个循环：

* 第一个循环写入一个对齐到 64B 的临时数组 A_tmp。
* 第二个循环从 A_tmp（对齐）复制到 A（对齐情况未知）。

对于这两个循环，我们还为步长生成了多版本代码：

* 循环 2a: (A_tmp = A + B + C + 1)

  * 版本 1：假设 A, B 和 C 都是单位步长的向量化。

    * 不需要剥离循环，因为 A_tmp 已经对齐。
    * 内核循环假设 A, B, C 未对齐进行向量化。（原文此处描述似乎矛盾，通常假设对齐才用对齐指令）
    * 余数循环使用 vgather/vscatter（未对齐）。

  * 版本 2：假设 A, B 和 C 都是非单位步长的向量化。

    * 剥离循环是标量的。
    * 内核循环使用 vgather/vscatter。
    * 余数循环是标量的。

* 循环 2b: (A = A_tmp)

  * 版本 1：假设 A 是单位步长的向量化。A_tmp 已知是单位步长且对齐。

    * 剥离循环是标量的。剥离直到 A 对齐。
    * 对于内核循环：

      * 版本 1a：假设 A 对齐，A_tmp 对齐进行向量化。（即剥离循环未执行任何迭代）
      * 版本 1b：假设 A 对齐，A_tmp 未对齐进行向量化。（原文描述似乎矛盾，A_tmp应是对齐的）

    * 余数循环是标量的。

  * 版本 2：假设 A 非单位步长的向量化。

    * 剥离循环是标量的。
    * 内核循环使用 vmovaps 加载 A_tmp（对齐）和 vscatter 存储 A（未对齐且有步长）进行向量化。
    * 余数循环是标量的。

示例 6.3： 步长信息由编译器传播的指针赋值。

```fortran
module mod1
    implicit none
    real, target, allocatable :: A1(:,:) ! 可分配目标数组
    real, pointer :: Z1(:,:) ! 指针
contains
    subroutine pointer_array3(N)
        integer N
        Z1 => A1(:, 1:N:2) ! 指针赋值：Z1 指向 A1 的列切片 (步长2)
        Z1 = Z1 + 1 ! 通过指针操作数组
        return
    end subroutine pointer_array3
end module mod1
```

* Z1 是 A1 的一个切片。它的内部维度是连续的，但外部维度是非连续的。
* 这个信息在循环中被使用，只生成了一个假设内部布局连续的内部循环向量化版本。
* 循环被剥离直到 A1 对齐。剥离循环使用 vgather/vscatter 进行向量化。
* 内核循环使用对齐的加载和存储进行向量化。
* 余数循环使用 vgather/vscatter 进行向量化。

示例 6.4： 由于函数调用导致步长信息丢失的指针示例。

```fortran
module mod2
    implicit none
    real, target, allocatable :: A2(:,:) ! 可分配目标数组
    real, pointer :: Z2(:,:) ! 指针
contains
    subroutine pointer_array4(N)
        integer N
        Z2 => A2(:, 1:N:2) ! 指针赋值：Z2 指向 A2 的列切片 (步长2)
        !..loop 1
        Z2 = 2.3 ! 通过指针初始化赋值

        call dummy_call() ! 可能修改 A2 位置的函数调用

        !..loop 2
        Z2 = Z2 + 1 ! 通过指针计算赋值

        return
    end subroutine pointer_array4
end module mod2
```

Z2 是 A2 的一个切片。它的内部维度是连续的，但外部维度是非连续的。

* 循环 1:

  * A2 的步长信息在循环中被使用，只生成了一个假设内部布局连续的内部循环向量化版本。
  * 循环被剥离直到 A2 对齐。剥离循环使用 vgather/vscatter 进行向量化。
  * 内核循环使用对齐的加载和存储进行向量化。
  * 余数循环使用 vgather/vscatter 进行向量化。

* 循环 2:

  * 对 dummy_call() 的函数调用可能潜在地修改 A2 所指向的位置。因此，关于 A2 的步长信息丢失了。
  * 为 A2 的步长生成了多版本代码：

    * 版本 1：假设 A2 是单位步长的向量代码（针对内循环）

      * 剥离循环直到 A2 对齐。剥离循环使用收集/散布进行向量化。
      * 内核循环使用对齐的加载和存储进行向量化。
      * 余数循环使用收集/散布进行向量化。

    * 版本 2：假设 A2 是非单位步长的向量代码（针对内循环）

      * 内核循环使用收集/散布进行向量化。
      * 余数循环有两个版本：一个使用收集/散布的向量化版本和一个标量版本。

### 间接数组访问 (Indirect Array Access)

对数组的间接访问在具有稀疏数组的应用程序、基于自适应网格的应用程序以及许多 N 体应用程序中非常常见。在这种情况下，顺序访问一个索引数组，并将其用作基数组/矩阵的偏移量。内存访问是非连续的，需要使用收集/散布指令。

示例 7.1： 使用 Fortran 数组表示法进行间接数组访问。

```fortran
subroutine indirect1(A, B, ind, N)
    real A(N), B(N)
    integer N
    integer ind(N) ! 索引数组

    A(ind(:)) = A(ind(:)) + B(ind(:)) + 1 ! 数组表示法：对所有元素同时赋值

    return
end subroutine indirect1
```

* 数组 A 和 B 通过索引数组 ind 被间接访问。
* Fortran 数组表示法的语义要求执行此赋值语句的结果等同于计算右侧所有数组元素的值并将结果赋给目标。由于在此赋值语句的右侧和左侧（关于 A）之间存在循环携带的流依赖（flow dependence），因此该赋值通过使用一个临时数组来存储整个右侧结果，然后使用第二个循环将其复制到 A() 数组来实现向量化。因此，编译器生成两个循环：

  * `T(:) = A(ind(:)) + B(ind(:)) + 1`
  * `A(ind(:)) = T(:)`

* 循环 1 (T = ...):

  * 无剥离循环。T 被分配为已对齐。
  * 内核循环使用 vgather 加载 A 和 B，使用非对齐加载加载 ind，使用对齐存储存储 T 进行向量化。
  * 余数循环使用 vgather 加载 A 和 B，使用 vscatter 存储 T 进行向量化。

* 循环 2 (A = T):

  * 无剥离循环。T 已经对齐。
  * 内核循环使用对齐加载加载 T，使用 vscatter 存储 A，使用非对齐加载加载 ind 进行向量化。
  * 余数循环是标量的。

示例 7.2： 使用 Fortran do 循环进行间接数组访问。

```fortran
subroutine indirect2(A, B, ind, N)
    real A(N), B(N)
    integer N
    integer ind(N) ! 索引数组
    integer i
    !dir$ ivdep ! 忽略向量依赖指令

    do i=1, N
        A(ind(i)) = A(ind(i)) + B(ind(i)) + 1 ! Do循环顺序赋值
    end do

    return
end subroutine indirect2
```

* 数组 A 和 B 通过索引数组 ind 被间接访问。
* do 循环的语义与上面的 Fortran 数组表示法不同。do 循环意味着循环迭代的顺序执行，由于在此赋值语句的右侧和左侧（关于 A）之间存在循环携带的流依赖，这将阻止此循环的向量化。然而，程序员提供的 ivdep 指令表明忽略循环携带的流依赖并向量化此代码是安全的。
* 由于此循环的语义和指令，编译器不需要生成临时数组和两个循环。

  * 循环被剥离直到 ind 对齐。剥离循环使用 vgather 加载 A, B, ind 和 vscatter 存储 A 进行向量化。
  * 内核循环使用 vgather 加载 A, B，对齐加载 ind，和 vscatter 存储 A 进行向量化。
  * 余数循环使用 vgather 加载 A, B，对齐掩码加载（aligned masked load）加载 ind，和 vscatter 存储 A 进行向量化。

### Fortran90 模块中的全局数组 (Global Arrays in Fortran90 modules)

模块为定义全局数据提供了一种有效的方法。您可以在模块内部使用 align 子句来全局对齐数组：

示例 8.1： 在模块中声明的具有已知大小的全局数组。

```fortran
module mymod
    !dir$ attributes align:64 :: a ! 属性：a 64字节对齐
    !dir$ attributes align:64 :: b ! 属性：b 64字节对齐
    real (kind=8) :: a(1000), b(1000) ! 静态数组
end module mymod

subroutine add_them()
    use mymod ! 使用模块
    implicit none

    ! 数组语法
    a = a + b ! 无需显式指令告诉编译器 A 和 B 对齐，USE 带来了该信息
end subroutine add_them
```

* 模块内的 attributes align 指令用于告诉编译器 A 和 B 是 64 字节对齐的。
* 没有生成剥离循环，A 和 B 已经对齐。请注意，这里不需要在循环前单独使用 vector-aligned 指令。
* 内核循环使用 vmovapd（对齐双精度移动）进行向量化。

示例 8.2： 在模块中声明的全局可分配数组，但在其他地方分配。

```fortran
module mymod
    real, allocatable :: a(:), b(:) ! 可分配数组
end module mymod

subroutine add_them()
    use mymod ! 使用模块
    implicit none

    !dir$ vector aligned ! 向量循环，假设内存访问对齐
    a = a + b
end subroutine add_them
```

* 模块内的 attributes align 指令在这里不够 - 因为 A 和 B 的实际分配是在稍后调用 ALLOCATE 时发生的。
* 如果如上所示使用 vector aligned 指令，则不会生成剥离循环，编译器假设 A 和 B 已经对齐。
* 如果移除 vector aligned 指令，编译器没有正确的对齐信息，则会生成用于对齐数组的剥离循环。
* 内核循环使用 vmovaps, vaddps（对齐单精度移动和加法）进行向量化。

请注意，更改模块以添加 attributes align 子句在这里没有帮助：

```fortran
module mymod
    real, allocatable :: a(:), b(:)
    !dir$ attributes align:64 :: a ! 对齐指针变量本身，而非指向的数据
    !dir$ attributes align:64 :: b ! 对齐指针变量本身，而非指向的数据
end module mymod
```

这个模块定义并没有说明分配的数据指针（pointer value）是否是 64 字节对齐的。[想想看：指针变量本身位于 64 字节边界，但存储在该指针变量中的指针值（即地址）可能对齐也可能未对齐 64 字节]

### 使用 `-align array64byte` 选项对齐所有数组

编译器选项 `-align array*n*byte` 作用于所有 Fortran 数组（COMMON 块中的数组除外）。它将数组的起始地址对齐到 n 字节边界。n 可以是 8、16、32、64、128 或 256。n 的默认值是 8。数组元素之间没有填充（padding）。这适用于具有固定边界、假定形状（assume-shape）、假定大小（assume-size）、自动（automatic）、可分配（allocatable）或指针（pointer）属性的数组（但是，这不适用于 Cray 指针数组。这些数组没有对齐控制）。

Fortran 数组有三种内存样式：
1. 静态内存（Static memory） – 数组中内存的起始地址在 n 字节边界上对齐。
2. 栈内存（Stack memory） – 局部的、自动的、临时的 – 数组中在栈上的起始地址在 n 字节边界上对齐。
3. 堆内存（Heap memory） – 具有可分配或指针属性（非 Cray 指针数组） – 由调用 aligned_malloc 的 ALLOCATE 语句分配内存，因此数组中在堆上的起始地址在 n 字节边界上对齐。

`-align array*n*byte` 是在一次编译中对齐所有数组的一种方法。如果您想改为对齐单个数组，请对每个单独的数组使用 ALIGN 属性。此外，程序员还应在每个循环之前使用指令（`!dir$ vector aligned, !dir$ assume_aligned`）明确告知编译器哪些数组将以对齐方式访问。

### Fortran 2008 CONTIGUOUS 属性

这是一个数组属性，用于告诉编译器数据占据一个连续的内存块。这允许编译器对指针和假定形状数组进行优化。默认情况下，编译器必须假设它们可能是非连续的。但添加此属性将提供额外信息，编译器可以自由地假设数据访问将是连续的。请注意，如果用户断言错误且数据在运行时不连续，则结果是不确定的（错误答案，段错误）。

```fortran
real, pointer, contiguous :: ptr(:) ! 连续指针
real, contiguous :: arrayarg(:,:) ! 连续假定形状数组参数
```

指针的目标（POINTER target）必须是连续的。对应于假定形状数组的实际参数必须是连续的。

还有一个 Fortran 2008 内在函数：

```fortran
is_contiguous() ! 返回逻辑值
IF ( is_contiguous(thisarray) ) THEN
    ptr => thisarray ! 仅在连续时才赋值
```

以下是 contiguous 属性的使用示例：

```fortran
subroutine foo1(X, Y, Z, low1, up1)
    real, contiguous, intent(inout), dimension(:) :: X ! 连续假定形状
    real, contiguous, intent(in), dimension(:) :: Y, Z ! 连续假定形状
    integer i, low1, up1

    do i = low1, up1
        X(i) = X(i) + Y(i)*Z(i)
    enddo

    return
end subroutine foo1
```
用户添加了 `CONTIGUOUS` 属性，断言 foo1 的所有调用点对三个数组参数都使用连续的切片。编译器利用此属性，为循环生成单位步长向量化代码，而无需为步长进行任何循环多版本处理。

### 使用 Fortran 数组表示法实现单位步长向量化

Fortran 语言语义允许对许多数组类型（如可分配数组、可调数组、显式数组、假定大小数组等）进行单位步长向量化。这仍然要求向量循环索引位于最后一个维度。当此条件不成立时，可能可以通过重构代码使用 F90 数组表示法来获得更高性能。下面是一个示例。

原始源代码：

```fortran
do I=1, m
    do index=1, n
        A(I, j, k, index) = B(I, j, k, index) + C(I, j, k, index) * D
    enddo
enddo
```

* 内层循环使用非单位步长向量化（因为 index 不在最内层维度）。
* 导致（效率较低的）收集/散布操作。

重构后的源代码：

```fortran
do I=1, m, VLEN ! VLEN 是向量长度（例如 4, 8, 16）
    do index=1, n
        ! 使用数组切片表示法在 j,k,index 固定时，对 I 维度进行切片
        A(I:I+VLEN-1, j, k, index) = B(I:I+VLEN-1, j, k, index) + &
                                     C(I:I+VLEN-1, j, k, index) * D
    enddo
enddo
```

使用 Fortran 数组表示法有助于在内层循环中使用单位步长进行向量化（因为切片 `I:I+VLEN-1` 在内存中是连续的）。

## 要点总结

本文档提供了 Fortran 中各种数组类型的示例及其作为局部变量、函数/子程序参数的用法，以及 Fortran 指针及其用法的示例。本文档展示了编译器如何对各种数组数据类型和参数进行向量化处理。这包括高级源代码示例，并解释了编译器针对这些情况生成的代码。

有相当多的示例 - 要点是什么？

1. 间接数组访问代价高昂且导致低效代码。 是的，编译器仍然可以使用 vgather/vscatter 对具有间接内存引用的循环进行向量化，但这并不意味着它是高效的代码！！！高效的代码是单位步长访问。如果您的代码在追踪指针或使用间接引用，那么无论是否生成了向量代码，它都不是最优的。如果可能，尽量避免间接内存引用，或者理解编译器无法神奇地使此代码高效快速。
2. 避免用于参数传递的数组临时变量： 这可以通过传递连续的数组数据、使用显式接口（explicit interfaces）和假定形状数组（assumed shape arrays）来实现。更好的方法是把数据放在模块中，并使用 USE 模块，而不是显式地传递参数。
3. 尽管 Fortran 指针比 C 指针限制多得多，但它们仍然是别名（aliases）。 基于指针的变量在左侧表达式（LHS）和右侧表达式（RHS）上的潜在重叠（overlap）将导致创建数组临时变量来保存 RHS 表达式，因为它需要在存储到 LHS 之前被求值。如果可能，避免使用指针而使用可分配数组。
