---
layout: post
title: Fortran面相对象编程
date: 2025-4-29
categories: tools
---

本文参考自

# 扩展类型与绑定过程

允许创建面向对象程序的基本特征是类型扩展和类型绑定过程。例如，考虑二维空间中的点。下面从几个方向对这个例子进行阐述。出发点就是这种衍生类型：

```fortran
type point2d 
  real :: x, y
end type point2d
```

可以使用Fortran90/95语言以经典的方式定义操作，如向该点添加一个向量或对原点进行缩放：

```fortran
module points2d
  implicit none

  type point2d
    real :: x, y
  end type point2d

  contains 
  
    type(point2d) function add_vector( point, vector ) 
      type(point2d), intent(in) :: point, vector  
      
      add_vector%x = point%x + vector%x add_vector%y = point%y + vector%y 
    end function add_vector  
    
    type(point2d) function scale_by_factor( point, factor ) 
      type(point2d), intent(in) :: point, vector 
      real, intent(in) :: factor  
      
      scale_by_factor%x = factor * point%x scale_by_factor%y = factor * point%y 
    end function scale_by_factor 

end module points2d
```

# 抽象数据类型和泛型编程

随着Fortran 90中指针的引入，开发操纵数组以外其他数据结构的代码变得非常简单。事实上，任何递归定义的抽象数据类型，如链表和二叉树，都可以直接在Fortran 90中实现：

```fortran
type linked_list 
  integer :: value 
  type(linked_list), pointer :: next 
end type linked_list
```

它的基本特征是允许使用一个指向尚未(完全)定义的类型的组件的指针。定义应该出现在同一个编译单元中。

面临的主要问题是要存储的数据类型。Fortran 90不容易允许不同的数据类型存储在一个链表或树中。也就是说，对于一个应该包含实数和字符串数组的列表，你需要一个类似的结构：

```fortran
type linked_list 
  real, dimension(:), pointer :: array 
  character(len=80) :: string ! Not used if array 
                              ! is associated 
  type(linked_list), pointer :: next 
end type linked_list
```

这与第11.1节中讨论的处理二维和三维空间中点的方法非常类似。

一种完全不同的方法是将数据转换为您所存储的类型，并分别跟踪原始类型：

```fortran
type linked_list 
  integer, dimension(:), pointer :: array 
  integer :: type_indicator ! 1 - real array, 
                            ! 2 - character string 
  type(linked_list), pointer :: next 
end type linked_list
```

下面是带有关联惯例的代码：

```fortran
subroutine store_real_array( element, array ) 
  type(linked_list) :: element 
  real, dimension(:) :: array  
  
  element%type_indicator = 1 
  call store_integer_array( & 
          element, transfer(array,element%array) ) 
end subroutine store_real_array

subroutine store_character_string( element, string ) 
  type(linked_list) :: element 
  character(len=*) :: string  
  
  element%type_indicator = 2 
  call store_integer_array( & 
          element, transfer(string,element%array) ) 
end subroutine store_character_string

! Private routine - here we know the exact size 
subroutine store_integer_array( element, data ) 
  type(linked_list) :: element 
  integer, dimension(:) :: data  
  
  allocate( element%array(size(data) ) 
  element%array = data 
end subroutine store_integer_array

... Similar routines to retrieve the data in the original form
```

抽象数据类型的代码很容易被重用来存储任何类型的数据，如果该类型是派生类型的话：

```fortran
module linked_list_points2d 
  use points2d, stored_type => point2d

  private public :: linked_list, add, ....  
  type linked_list 
    type(stored_type) :: data 
    type(linked_list), pointer :: next 
  end type linked_list  
  
  !  
  ! Define generic names for the functionality 
  ! - makes using different types of lists easier 
  !  
  interface add 
    module procedure :: add_element 
  end interface  
  
  ... Other interfaces  

contains
  
  ... Subroutines and functions needed  

end module linked_list_points2d
```
