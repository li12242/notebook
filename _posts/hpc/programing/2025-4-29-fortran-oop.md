---
layout: post
title: Fortran面相对象编程
date: 2025-4-29
categories: hpc
---

## 面向对象术语

* 类（class）：在OOP术语中，类是创建对象的关键，而派生类型是Fortran特有的术语。
* 实例（Instance）：实例总是指程序中的一个类中对象的具体实现。当完成了类定义后，就可以声明该类的多个不同实例。
* 组件（Component）：类中可以包含任意类型、任意数量的变量，标量或数组，甚至是相同或其他类的实例。将把所有这些都称为类组件。
* 方法（Method）：类中绑定的子例程称为方法。首先，方法总是伴随着类实例而来，不需要单独导入。其次，方法可以访问一个类的所有组件和其他方法。

## 派生类型

### 派生类型定义

在 fortran 语言中，派生类型是变量的集合，其允许用户使用基本数据类型来设计和构建任何复杂程度的任意数据类型。具体通过 `type :: <type-name>` 和 `end type <type-name>` 关键字来实现派生类型定义，如下代码所示。

```fortran
type :: Person
  character(len=20) :: name
end type Person
```

### 继承

通过 `extends` 关键字，可以实现派生类型继承，如下类型中增加了 `address` 组件。

```fortran
type, extends(Person) :: Student
  character(len=20) :: address
end type
```

### 默认初始化

通过调用类型的名称来初始化派生类型实例，并在参数中传递其组件的值，如下代码所示。这是一种默认的初始化派生类型的方式，它要求所有的组件都作为参数传递给类型构造器。

```fortran
type(Person) :: some_person
some_person = Person('Jill')
```

如包含多个组件，在使用默认构造函数时，任何位置参数必须按照类型定义中组件定义的顺序列出，另外一种方法则是通过关键字参数。当混用两种方法传入参数时，关键字参数需要在默认参数之后。

```fortran
type :: Person
  character(len=20) :: name
  integer :: age
  character(len=20) :: occupation
end type Person
...

some_person = Person('Bob', 32, 'Engineer')
some_person = Person(occupation='Engineer', name='Bob', age=32)
some_person = Person('Bob', occupation='Engineer', age=32)
```

### 附：面向对象代码结构 

> 如果想定义一个Field派生类型，最好在mod_field 模块中定义它和它的所有方法，并将它存储在mod_field.f90文件中。在某些情况下，在同一个模块中定义紧密相关的派生类型是有意义的。

## 组件与方法

### 组件

* 派生类型内定义组件可以通过百分号访问，如 `some_person % name`。
* 组件可以有默认值，此时在默认构造函数中可以省略此组件参数传入。
* 组件可以声明为 `private` 类型，禁止外部访问。
* 类型内包含 allocatable 和 pointer 属性组件，允许定义为自身类型，这种类型通常称为递归类型。

> 当组件为私有属性时，无法使用默认构造函数对派生类型进行初始化。

### 方法绑定

通过如下定义，能够将 `copy_point2d` 函数绑定到派生类型 `point2d` 上，并重命名为 `copy`。

```fortran
  type :: point2d
    real :: x, y
  contains
    procedure :: copy => copy_point2d
  end type
contains
  .......
  subroutine copy_point2d(to, from)
    class(point2d), intent(inout) :: to
    class(point2d), intent(in) :: from
    to%x = from%x
    to%y = from%y
  end subroutine
```

* 在 `copy_point2d` 函数的类型定义中使用 `class` 而非 `type`，这是出于继承需求。`class(point2d)` 允许将从 `point2d` 类型扩展的任何子类传递给该子程序。
* 第一个传入参数一般为类型实例本身，也可使用 `self` 或 `this` 等 OOP 中常用变量名代替。当类型实例不是第一个参数，需在绑定语句中使用 `pass` 关键字指明函数声明中哪个参数是绑定类型。如果参数没有绑定类型，则需要 `nopass` 关键字。
* 通过 `contains` 语句将该过程绑定到派生类型定义内部，对于类型绑定过程，类型必须被声明为 `intent(in)` 或 `intent(inout)` 参数。
* 所有类型的组件和方法都是默认可见的。派生类型定义内部的单个 `private` 语句意味着默认情况下所有后续组件都将被声明为私有。单个 `public` 声明也是如此。

### 方法重载

#### 子例程重载

当使用 `point3d` 继承派生类型 `point2d` 时，同时可将 `copy` 函数进行重载，此时针对 `point3d` 类型变量进行复制操作。

```fortran
  type, extends(point2d) :: point3d
    real :: z
  contains
    procedure :: copy => copy_point3d
  end type point3d
contains
    subroutine copy_point3d(to, from)
    class(point3d), intent(inout) :: to
    class(point2d), intent(in) :: from

    select type (from)
    class is (point2d)
      to%point2d = from
      to%z = 0
    class is (point3d)
      to%point2d = from%point2d
      to%z = from%z
    end select
  end subroutine
```

* 在 `point3d` 中对 `copy` 进行重载时，其实际指向函数名为 `copy_point3d`，注意 Fortran 中不允许函数名重复。
* 重载函数 `copy_point3d` 声明必须与 `copy_point2d` 完全一致，除绑定类型可以变为 `class(point3d)`。
* 由于拷贝变量时，需考虑源数据为 `point3d` 类型，因此 `from` 变量类型需修改为 `class(point2d)` 从而实现多态变量。
* 针对 `class(point2d)` 类型多态变量，无法直接访问其扩展类型组件，如 `from%z`。为确定多态变量实际类型，可以使用 Fortran 2003 中内置函数 `select type`，在其中 `class is (point3d)` 分支内，此时 `from` 类型自动转换为 `point3d`，并允许访问 `from%z`。

#### 函数重载

对于函数重载时，由于函数声明一致的要求，返回值必须为 `class(point2d)`。为了能够返回扩展类型变量 `point3d`，只能令其为动态变量，从而能够在申请空间时按照 `point3d` 类型申请内存空间。

```fortran
class(point2d) function add_vector_2d( point, vector ) 
  class(point2d), intent(in) :: point 
  class(point2d), intent(in) :: vector 
  ...
end function add_vector_2d
```

```fortran
function add_vector_3d(point, vector)
  class(point3d), intent(in) :: point
  class(point2d), intent(in) :: vector

  class(point2d), allocatable :: add_vector_3d
  class(point3d), allocatable :: add_result

  allocate (add_result)
  add_result%point2d = point%point2d%add(vector)
  add_result%z = 0.0

  select type (vector)
  class is (point3d)
    add_result%z = point%z + vector%z
  end select

  call move_alloc(add_result, add_vector_3d)
end function
```

* 在原始函数和重载函数中，返回类型都为 `class(point2d), allocatable ::`，从而保证函数声明一致。
* 在重载函数中定义了动态辅助变量 `add_result`。当输入参数 `vector` 为扩展类型 `point3d` 时，会对 `add_result%z` 组件进行赋值。
* fortran 中支持使用 `mod_alloc` 实现移动语义而不是拷贝，可以实现创建对象的副本之后丢弃原始对象。

### 过程指针

Fortran 中可以在组件中包含过程指针（procedure pointer），其优点在于使用过程中与绑定过程完全一致，同时还可以针对不同类型指向不同函数。这使得不同类型同属于一个类，但是可以针对自身特殊情况定义不同执行方法。

```fortran
module polygons
  implicit none
  
  type polygon_type
    real, dimension(:), allocatable :: x, y
    procedure(compute_value), & pointer :: area => area_polygon 
  contains 
    procedure :: draw => draw_polygon 
  end type polygon_type

  abstract interface 
    real function compute_value(polygon) 
      import :: polygon_type 
      class(polygon_type) :: polygon 
    end function compute_value 
  end interface
```

在随后的多边形类型定义中，针对矩阵和正方向形等特殊类型，可以进一步对 `area` 指针组件进行定义，如下所示。

```fortran
real function area_rectangle( polygon ) 
  type(polygon_type) :: polygon  
  associate( x => polygon%x, y => polygon%y ) 
    area_rectangle = abs( (x(2)-x(1)) * (y(3) - y(2)) ) 
  end associate 
end function area_rectangle

subroutine new_rectangle( rectangle, x1, y1, width, height ) 
  real :: x1, y1, width, height 
  type(polygon_type), allocatable :: rectangle
  allocate( rectangle%x(4), rectangle%y(4) ) 
  rectangle%x = (/ x1, x1+width, x1+width, x1 /) 
  rectangle%y = (/ y1, y1, y1+height, y1+height /)  
  rectangle%area => area_rectangle  
end subroutine new_rectangle
```

## 构造函数与析构函数

### 自定义构造函数

Fortran 提供了默认显式构造方法对派生类型进行初始化。对于包含复杂初始化过程，或者包含私有组件的派生类型来说，则需要使用自定义构造函数对组件进行初始化。

```fortran
  interface person
    module procedure :: create_person
  end interface

contains
  function create_person(name, occupation) result(r)
    type(person) :: r
    character(len=*), intent(in) :: name, occupation
    allocate(r%name, source=name)
    allocate(r%occupation, source=occupation)
  end function
```

* 对于用户自定义构造函数需要为 function，并且返回类型与派生类相同。
* 一般来说，不强制对所有组件进行初始化，但是需要小心未分配内存的 allocatable 或 pointer 类型组件。
* 为了覆盖默认构造函数，需要定义接口，其名称与类型相同，从而告诉编译器使用类型实例创建语法时，就需要调用函数 `create_person`。

### 析构函数

对于包含 allocatable 或 pointer 类型组件的派生类型实例，在使用结束后需要手动释放使用内存。为避免内存泄露，可以在派生类型中定义包含 final 属性的析构函数，能够在以下情况下自动调用，并释放对象中申请的内存。

* 对象出现在赋值语句左侧（执行赋值操作之前）。
* 传入子例程中，对象类型包含 `intent(out)` 属性。
* 对象离开变量作用域。
* 对象（allocatable 或 pointer 类型）被释放。

> 主程序内变量默认具有save属性，因此无法自动调用析构函数。为解决这些问题可以使用 `block` 关键字。

### 符号重载

在派生类型中，通过 `generic ::` 关键字定义函数接口，而操作符重载也是通过此关键字定义。

```fortran
  type :: point2d
    real :: x, y
  contains
    procedure :: add => add_vector
    procedure :: copy => copy_point2d
    generic :: operator(+) => add
    generic :: assignment(=) => copy
  end type
```

## 虚类型与虚方法

通过 `abstrace` 和 `deferred` 关键字可以定义虚类型与虚方法。

```fortran
type, abstract :: abstract_point 
  ! No coordinates, leave that to the extending types 
contains 
  procedure(add_vector), deferred :: add 
end type abstract_point

abstract interface 
  subroutine add_vector( point, vector ) 
    import abstract_point 
    class(abstract_point), intent(inout) :: point 
    class(abstract_point), intent(in) :: vector 
  end subroutine add_vector 
end interface
```

* 对于所有继承 `abstract_point` 的派生类型，都必须将 `add` 绑定过程实例化。
* 可以声明类型为 `class(abstract_point)` 类型指针，其可以指向任何子类实例，如下面的 `point2d` 类型。

```fortran
type, extends(abstract_point) :: point2d 
  real :: x, y 
contains 
  procedure :: add => add_vector_2d 
end type point2d  

class(abstract_point), pointer :: p 
type(point2d), target :: point 
type(point2d) :: vector  
p => point 
call p%add_vector( vector )
```

## 高级面向对象技巧

### 无限制多态对象

fortran支持无限制多态（unlimited polymorphic）对象，这类对象使用 `class(*)` 声明，可以是内部类型、可扩展的派生类型等。如下语句所示。

```fortran
class(*), allocatable :: a_unlimited

allocate(a_unlimited, source=2.5e4)
select type( a_unlimited )
type is (real)
  write(*, *) "a_unlimited is of intrinsic real type with value ", a_unlimited
end select

deallocate( a_unlimited )
allocate( a_unlimited, source=point2d )
select type( a_unlimited )
type is (point2d)
  write(*, *) "x, y = ", a_unlimited%x, a_unlimitedy
end select
```
