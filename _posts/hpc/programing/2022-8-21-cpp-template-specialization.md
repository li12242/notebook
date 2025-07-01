---
layout: post
title: C++ 模板特化实现通用get函数
date: 2022-8-21
categories: hpc
---

# 背景

前一段时间写 OneAPI 时候发现，在 device 类里面有个很有意思的函数 `get_info()`，可以对设备多个属性进行输出。例如
```c++
auto devices = platform.get_devices();
for (auto &device : devices) {
    std::cout << "  Device: " << device.get_info<sycl::info::device::name>()
        << std::endl;
    std::cout << "  Device Vendor: " << device.get_info<sycl::info::device::vendor>()
        << std::endl;
}
```
可以看出，`get_info()` 明显是个模板函数，根据输入可以对 name、vendor 等不同属性进行输出。

在一般情况下，获取一个类中多个属性输出需要有多个 get 函数，并且每个属性对应一个单独函数。比如现在有个 `cpu_info` 类，其中包含了下面一些属性：
```c++
class cpu_info{
    const char vendor_name[64];
    const char model_name[64];
    int core_num;
}
```
为了获取这三个属性，需要构造三个独立的 get 函数，在每个函数内对单个属性进行读取。
```c++
class cpu_info{
    const char* get_vendor(){
        return vendor_name;
    }

    const char* get_model(){
        ......

    int get_core_num(){
        ......
}
```
这种情况下 get 函数个数会随着属性增加。

从上面示例就可以看出，目前get函数有两种形式：
1. 第一种类似 C 语言结构体写法，每个属性对应一个 get 函数，这种情况造成 API 函数众多，调用时候需要注意区分。
2. 第二种类似 OneAPI 中 `get_info()` 函数，通过 C++ 模板参数实现该函数复用，这种可以大量减少 API 函数，但是实现和调用时候都比较复杂。

本文就介绍下如何通过 C++ 模板实现第二种 get 函数功能。

# 示例 1

以下面一个基础示例进行说明。

```c++
#include <iostream>

template <int n> struct Typer {};

template <> struct Typer<1> { typedef int Type; };
template <> struct Typer<2> { typedef double Type; };
template <> struct Typer<3> { typedef const char *Type; };

class MyClass {
public:
  template <int typeCode> typename Typer<typeCode>::Type myfunc();
};

template <> Typer<1>::Type MyClass::myfunc<1>() { return 2; }
template <> Typer<2>::Type MyClass::myfunc<2>() { return 3.14; }
template <> Typer<3>::Type MyClass::myfunc<3>() { return "some string"; }

int main(int argc, char **argv) {
  MyClass a;
  std::cout << a.myfunc<2>() << std::endl;
  return 0;
}
```

## MyClass 类

先从 `MyClass` 类开始介绍。在这个类型中，只包含了一个模板函数 `myfunc()` 声明，其输入参数为空，但是返回值为 `typename Typer<typeCode>::Type`。
这个返回值模板内使用了“内嵌依赖类型名”（nested dependent type name），通过 `typename` 参数表明了 `Typer<typeCode>::Type` 是个变量类型，并且可以根据模板参数 `typeCode` 对应不同类型。关于返回值类型实现在后面会进一步介绍。

## 模板函数

在 `MyClass` 类之外，对 `myfunc()` 进行了定义，在函数最前包含了 `template<>`。这种带有空尖括号的 template 告诉编译器 myfunc 遵循模板特化。通过这种方式，在调用 `MyClass::myfunc<typeCode>` 时候，可以根据模板参数 `typeCode` 调用特定 myfunc 版本，例如在上述代码中 `MyClass::myfunc<1>`、`MyClass::myfunc<2>` 和 `MyClass::myfunc<3>` 等。
在最后 main 函数中，在对象 a 中直接调用 `myfunc<2>()`，即可实现上述定义中浮点数 3.14 的输出。

## 返回类型

再来说明下 Typer 结构体。这个结构体包含了整型模板参数，在编译期时候针对每个整数都会单独生成一份代码。
在后面定义中，共包含了三个 `Typer<typeCode>` 实现，每个实现中，定义了变量别名 `Type`：
```c++
template <> struct Typer<1> { typedef int Type; };
template <> struct Typer<2> { typedef double Type; };
template <> struct Typer<3> { typedef const char *Type; };
```
根据上述定义，在定义 `myfunc` 时候，根据模板参数 `template <int typeCode>` 就可以获得不同的返回类型 `typename Typer<typeCode>::Type`。从而实现 `myfunc` 函数只需定义一次，但是在可以由模板参数包含多种实现。

# 示例 2

通过上述定义基本就可以实现 OneAPI 中 device 类的 get_info 函数类似功能。还是以 `cpu_info` 类为例，增加模板函数，同时在类中增加枚举类 `type_code` 作为模板参数。
```c++
class cpu_info {
public:
  template <int n> struct typer {};

  template <int type_code> static typename typer<type_code>::type get_info();

  enum type_code{
    vendor = 1, ///< cpu vendor name
    model_name, ///< cpu model name
  }
}
```
在类外面增加 `cpu_info::typer` 对应类型定义，同时定义 `get_info` 针对不同模板参数实现。
```c++
template <> struct cpu_info::typer<cpu_info::vendor> {
  typedef const char *type;
};

template <> struct cpu_info::typer<cpu_info::model_name> {
  typedef const char *type;
};

template <>
cpu_info::typer<cpu_info::vendor>::type cpu_info::get_info<cpu_info::vendor>() {
  return cpu_info::get_instance()->_vendor;
}

template <>
cpu_info::typer<cpu_info::model_name>::type
cpu_info::get_info<cpu_info::model_name>() {
  return cpu_info::get_instance()->_model_name;
}
```

完成上述定义之后，在使用时，即可通过 `cpu_info::vendor` 和 `cpu_info::model_name` 等枚举类作为模板参数，选择特定的 `get_info` 函数版本，如下所示。
```c++
int main(int argc, char **argv){
    printf("%50s %s\n", "Vendor ID:", cpu_info::get_info<cpu_info::vendor>());
    printf("%50s %s\n", "Model name:", cpu_info::get_info<cpu_info::model_name>());

    return 0;
}
```

# 总结

通过上述模板函数方法，即可实现与 OneAPI 类似的通用 get 函数功能。主要特点就是通过模板参数，定义了多种 `get_info` 函数实现。在每个实现会返回类型中不同属性，并且返回值为对应属性。

传统 C 函数方法也可以实现类似功能。例如在函数中增加参数，根据参数不同输出不同属性，如
```c
void get_info(cpu_info::type_code type){
    swtich (type){
        case vendor:
            return (void)_vendor;
        case model_name:
            return (void)_model_name;
    }

}
```
这种方法缺点在于无法返回多种变量类型属性，强制输出时候必须强制转换为 void 类型，在调用时不知道具体返回类型。
此外，通过模板参数方法可以在编译期定义好全部实现，相较于函数中使用多个 Switch-case 语句，性能上也有一定优势。