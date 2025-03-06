---
layout: post
title: Modern CMake for WRF
date: 2025-3-4
categories: tools
---

# 现代 CMake 工具在 WRF 软件中应用

相较于传统 Make 工具，CMake 实现了软件项目构建跨平台和自动化等多种功能，并提供了众多模块支持[^1]。在传统HPC应用中，许多应用一直使用 Makefile 对软件进行构建，但也有一些HPC应用，如WRF[^2]等，开始使用CMake作为软件构建的替代方案。

本文将以WRF中CMake项目编写为例，对CMake使用方法及功能进行详细介绍。

[^1]: https://cmake.org/cmake/help/v3.16/

[^2]: https://github.com/wrf-model/WRF

# WRF 构建过程介绍

下面首先介绍WRF使用CMake命令编译软件过程。
与常规 cmake 编译过程不同，在WRF中直接运行`./configure_new`命令，出现如下交互式命令，选择合适的编译器，最终在当前目录内生成 `_build` 目录。

```bash
$ ./configure_new
Using default build directory : _build
Using default install directory : /mnt/beegfs/lilongxiang/project/wrf-for-intel-nre/WRFV4.6.0/install
0   Linux         gfortran    /    gcc         /    mpif90      /    mpicc      GNU (gfortran/gcc)
1   Linux         ifx         /    icx         /    mpif90      /    mpicc      INTEL (ifx/icx) : oneAPI LLVM
!! - Compiler not found, some configurations will not work and will be hidden
Select configuration [0-1] Default [0] (note !!)  : 1
Select option for WRF_CORE from WRF_CORE_OPTIONS [0-4]
        0 : ARW
        1 : CONVERT
        2 : DA
        3 : DA_4D_VAR
        4 : PLUS
Default [0] :
Select option for WRF_NESTING from WRF_NESTING_OPTIONS [0-3]
        0 : NONE
        1 : BASIC
        2 : MOVES
        3 : VORTEX
Default [1] :
Select option for WRF_CASE from WRF_CASE_OPTIONS [0-13]
        0 : EM_REAL
        1 : EM_FIRE
        2 : EM_SCM_XY
        3 : EM_TROPICAL_CYCLONE
        4 : EM_HELDSUAREZ
        5 : EM_B_WAVE
        6 : EM_GRAV2D_X
        7 : EM_HILL2D_X
        8 : EM_LES
        9 : EM_QUARTER_SS
        10 : EM_SEABREEZE2D_X
        11 : EM_CONVRAD
        12 : EM_SQUALL2D_X
        13 : EM_SQUALL2D_Y
Default [0] :
[DM] Use MPI?    Default [Y] [Y/n] :
[SM] Use OpenMP? Default [N] [y/N] : y
Configure additional options? Default [N] [y/N] :
```

生成 `_build` 目录就是默认的 cmake 构建目录。查看此目录可以看出，其中增加了 `wrf_config.cmake` 编译设置文件及 `WRFConfig.cmake` 等模块设置文件。随后在 `_build` 目录中，执行 make 命令，即可完成 wrf 可执行程序的编译和链接过程。

从上面 WRF 等编译过程可以看出，脚本 `configure_new` 自动对当前环境进行检查，配置对应编译器选项，并根据输入的编译模块和并行设置，配置CMake编译脚本。

# WRF 软件 CMake 文件分析

下面将对 WRF 软件内 CMakeLists.txt 文件内容进行详细分析，通过介绍常见一些命令和用法来理解 CMake 执行过程。

## 基本环境设置

L1-L8为项目基本设置，包括要求 cmake 最低版本，支持语言，设置项目名称等。
```cmake
cmake_minimum_required( VERSION 3.20 )
cmake_policy( SET CMP0118 NEW )

enable_language( C )
enable_language( CXX )
enable_language( Fortran )

project( WRF )
```

## 变量设置

L9-L25对WRF编译时使用的一些变量进行设置。

```cmake
set( EXPORT_NAME ${PROJECT_NAME} )

if ( DEFINED CMAKE_TOOLCHAIN_FILE )
  set( WRF_CONFIG ${CMAKE_TOOLCHAIN_FILE} )
  # message( STATUS "Loading configuration file... : ${WRF_CONFIG}" )
  # include( ${WRF_CONFIG} )
endif()

# list( APPEND CMAKE_MODULE_PATH         )
list( APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/ ${PROJECT_SOURCE_DIR}/cmake/modules )

# Use link paths as rpaths
set( CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE )
set( CMAKE_Fortran_PREPROCESS          ON )

# This is always set
list( APPEND CMAKE_C_PREPROCESSOR_FLAGS -P -nostdinc -traditional )
```

CMake 变量大致可以分为两种，分别是环境变量、缓存变量，普通变量。前两种需要使用 `$ENV{}` 和 `$CACHE{}` 进行引用。后一种类似普通编程语言中变量，并且还可以为列表形式。

普通变量赋值使用 `set` 命令，若此变量为列表，可以通过 `list` 命令向列表内加入新的元素，如下所示。
```cmake
list( APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/ ${PROJECT_SOURCE_DIR}/cmake/modules )
```


缓存变量如下定义所示（L83），其中关键字 `CACHE` 表明此变量为缓存变量。在使用 ccmake 等 GUI 工具时，缓存变量会显示，并按照其类型提供相应设置方式。如下面代码中 `STRING` 表面此缓存变量为字符类型，第二个参数 `""` 为初始默认值，最后参数 `"WRF_CORE"` 为变量提示。
```cmake
set( WRF_CORE "" CACHE STRING "WRF_CORE" )
```

## 函数和宏

在 L27-L36，通过 `include()` 引入了附加的 cmake 配置文件。这些文件默认以 `*.cmake` 结尾，通过 `include()` 函数引入后，可以对函数内定义的函数或宏进行调用（后面用function 和 marco 代指）。

```cmake
include( CMakePackageConfigHelpers )
include( CheckIPOSupported )
include( c_preproc   )
include( m4_preproc  )
include( target_copy )
include( confcheck   )
include( gitinfo     )
include( printOption )
include( wrf_case_setup )
include( wrf_get_version )

check_ipo_supported( RESULT IPO_SUPPORT )

# First grab git info
wrf_git_commit(
                RESULT_VAR        GIT_VERSION
                WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
                )
```

如上所示代码中，`wrf_git_commit` 是 `gitinfo.cmake` 模块内定义的 marco 过程。函数 `wrf_git_commit` 对传入的 `GIT_VERSION` 和 `${PROJECT_SOURCE_DIR}` 进行处理，最终获得git commit 信息，并返回给 `GIT_VERSION` 变量。

在模块文件 `gitinfo.cmake` 内，宏 `wrf_git_commit` 结构如下所示。与C语言函数定义形式不同，在 cmake 中声明 marco 或 function 可以不定义输入变量，而通过命令 `cmake_parse_arguments` 对命令进行匹配。回忆下此函数调用代码，其中 `${PROJECT_SOURCE_DIR}` 是输入目录路径，而 `GIT_VERSION` 是返回的 git 信息，变量前面的 `RESULT_VAR` 和 `WORKING_DIRECTORY` 则类似函数的（关键字）参数。

```cmake
# WRF Macro to identify the commit where the compiled code came from
macro( wrf_git_commit )

  set( options        )
  set( oneValueArgs   WORKING_DIRECTORY RESULT_VAR )
  set( multiValueArgs )

  cmake_parse_arguments(
                        WRF_GIT_COMMIT
                        "${options}"  "${oneValueArgs}"  "${multiValueArgs}"
                        ${ARGN}
                        )
  
  ......

endmacro()
```

在进入 `wrf_git_commit` 宏之后，在执行 `cmake_parse_arguments` 之前，会使用 `set` 命令，定义三组变量 `options`、`oneValueArgs` 和 `multiValueArgs`，分别表示逻辑参数、单个参数和多个参数等类型。关键字参数（`RESULT_VAR` 和 `WORKING_DIRECTORY`）按照其实际传入参数类型，赋值给对应变量。随后调用 `cmake_parse_arguments` 对参数进行匹配，其中第一个参数 `WRF_GIT_COMMIT` 是匹配参数的 `prefix`，所有匹配后的参数形式为 `WRF_GIT_COMMIT_XXX`。以下面调用过程为例，返回的匹配参数有两个，分别是 `WRF_GIT_COMMIT_RESULT_VAR` 和 `WRF_GIT_COMMIT_WORKING_DIRECTORY`。

## 生成器表达式

在 WRF 中通过 `add_compile_definitions()` 命令添加预编译指令，如下所示。其中大量使用了生成器表达式工具。

```cmake
add_compile_definitions(
                        MAX_DOMAINS_F=${MAX_DOMAINS_F}
                        ......
                        # Only define if set, this is to use #ifdef/#ifndef preprocessors
                        # in code since cmake cannot handle basically any others :(
                        # https://gitlab.kitware.com/cmake/cmake/-/issues/17398
                        $<$<BOOL:${ENABLE_CHEM}>:WRF_CHEM=$<BOOL:${ENABLE_CHEM}>>
                        $<$<BOOL:${ENABLE_CHEM}>:BUILD_CHEM=$<BOOL:${ENABLE_CHEM}>>
                        $<$<BOOL:${ENABLE_CMAQ}>:WRF_CMAQ=$<BOOL:${ENABLE_CMAQ}>>
                        ......
                        )
```

生成器表达式基本形式如下，其中 `Expression` 为特定类型关键字（布尔类型、关键字参数等），arg1、arg2、arg3 等为逗号分隔的参数。

```cmake
$<Expression:arg1,arg2,arg3>
```

生成器表达式可以将复杂的IF命令缩减到只有一行，例如下面两个代码是等价的。其中使用生成器表达式过程中，使用了嵌套形式：

1. `$<CMAKE_SYSTEM_NAME:LINUX>` 判断当前平台是否为 `LINUX`，如果为真则返回1，反之返回0。
2. `$<1:LINUX=1>` 如果为真，则表达式最终值为 `LINUX=1`。
3. `$<0:LINUX=1>` 如果为假，则表达式最终值是空值。

```cmake
if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")  
        target_compile_definitions(myProject PRIVATE LINUX=1)  
endif()
```

```cmake
target_compile_definitions(myProject PRIVATE $<$<CMAKE_SYSTEM_NAME:LINUX>:LINUX=1>)
```

一些常用的表达式除判断 `AND`、`OR` 外，还有判断字符串是否相等的 `STREQUAL` 等。通过生成器表达式可以大幅缩减 cmake 代码行数。
