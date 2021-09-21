---
layout: post
title: Spack 简明教程
date: 2021-9-18
categories: tools
---
# Spack 简明教程

## spack 简介

Spack 是采用 Python 语言建立的高性能集群包管理器，可以简化各种科学软件的安装过程。通过使用Spack，用户可以建立不同版本、配置、平台和编译器编译的软件。

Spack并不针对特征语言，用户可以使用Python或R程序，并链接到 C、C++ 或 Fortran 等混合编译器。Spack可以用来管理共享的安装包和目录，并结合不同版本软件进行测试。

Spack安装十分简单，下载 Spack 源码压缩包，解压后直接运行配置脚本初始化后即可使用。

```bash
$ unzip spack-develop.zip
# define SPACK_ROOT
$ echo 'export SPACK_ROOT=<path to spack folder>' >> ~/.bashrc
# import setup script for spack
$ echo '. ${SPACK_ROOT}/share/spack/setup-env.sh' >> ~/.bashrc
```

## spack 基本操作

### 软件安装与卸载

在 Spack 中，安装指定版本的软件并解决其依赖非常简单，使用 `@` 符号表示版本，使用 `%` 指明使用编译器即可。如下示例中展示了使用 spack 安装 hdf5 软件的过程。

```bash
# Install a specific version by appending @
$ spack install hdf5@1.10.1

# Specify a compiler (and optional version), with %
$ spack install hdf5@1.10.1 %gcc@4.7.3
```

除了对软件版本和编译器选择外，Spack 还可对软件安装过程进行一些更详细的修改，例如是否安装某个依赖库，以及软件编译参数修改等。当软件依赖于其他软件时，可以用 `^` 关键字，对于依赖软件也可使用其他关键字进行递归调用。例如

```bash
$ spack install tcl ^zlib @1.2.8 %clang
```

在某些软件安装过程中，会包含一些布尔变量指令。这些指令可以用符号 `+` 和 `~` 或 `-` 来增加或去除。其中后两个符号都是表示 `False`。

```bash
# 安装 hdf5 并不使用 mpi 支持
$ spack install hdf5~mpi
```

如下所示：

```bash
# Add special boolean compile-time options with +
$ spack install hdf5@1.10.1 %gcc@4.7.3 +szip

# Add custom compiler flags
$ spack install hdf5@1.10.1 %gcc@4.7.3 cppflags="-O3 -floop-block"

# Cross-compile for compute nodes on a Cray or Blue Gene/Q
$ spack install hdf5@1.10.1 target=backend
```

在 Spack 安装软件的过程中，是通过采用 RPATH 命令来解决软件间依赖关系的，因此当软件运行时，不需要再对 LD_LIBRARY_PATH 环境变量进行修改。

当需要加载某个安装软件时，使用 `spack load <package_name>` 即可直接加载所有需要的环境。除此之外，Spack 还支持使用 Module 命令，在安装好对应软件后，还会配置对应的 modulefiles。

### 编译器选择


### 软件查询

软件安装好之后，可以使用 `spack find` 命令对安装软件进行查询，例如软件依赖库、安装参数和安装位置等等。当使用不同参数安装同个软件的不同版本时，可以使用附件参数进行区分，如 `--long` 等。常用命令及对应参数如下：

```bash
# 查询软件不同版本
$ spack find --long libdwarf
==> 2 installed packages.
-- linux-debian7-x86_64 / gcc@4.4.7 --------------------------------
libdwarf@20130729-d9b90962  libdwarf@20130729-b52fac98

# 查询此软件不同版本依赖的底层库
$ spack find --deps libdwarf
==> 2 installed packages.
-- linux-debian7-x86_64 / gcc@4.4.7 --------------------------------
    libdwarf@20130729-d9b90962
        ^libelf@0.8.12
    libdwarf@20130729-b52fac98
        ^libelf@0.8.13
        
# 查询软件安装参数
$ spack find -v binutils@2.35.1
==> 1 installed package
-- linux-centos8-icelake / clang@13.0.0 -------------------------
binutils@2.35.1+gold~headers~interwork~ld~libiberty~lto+nls~plugins

# 查询所有软件安装位置 
$ spack find --paths 
==> 74 installed packages.
-- linux-debian7-x86_64 / gcc@4.4.7 --------------------------------
    ImageMagick@6.8.9-10  ~/spack/opt/linux-debian7-x86_64/gcc@4.4.7/ImageMagick@6.8.9-10-4df950dd
    adept-utils@1.0       ~/spack/opt/linux-debian7-x86_64/gcc@4.4.7/adept-utils@1.0-5adef8da
...

# 查询特定软件安装位置
$ spack find --paths libelf
-- linux-debian7-x86_64 / gcc@4.4.7 --------------------------------
    libelf@0.8.11  ~/spack/opt/linux-debian7-x86_64/gcc@4.4.7/libelf@0.8.11
    libelf@0.8.12  ~/spack/opt/linux-debian7-x86_64/gcc@4.4.7/libelf@0.8.12
...
```

## 定制化安装

在配置文件 `${SPACK}/etc/spack/defaults/packages.yaml` 中，包含了一些软件自定义方法，可以让 Spack 按照用户指定方式安装应用，或者告诉 Spack 构建新的软件时使用本地已安装的底层软件。

可以在用户目录中新建 `~/.spack/packages.yaml` 来覆盖默认目录中配置文件，此文件格式为 

```yaml
packages:
  package1:
    # settings for package1
  package2:
    # settings for package2
    # ...
  all:
    # settings that apply to all packages.
```

### 本地软件配置

以 OpenMPI 为例，通过设置文件可以指定本地安装的多个软件目录，以及对应的安装说明（`spec`）。在配置文件 `packages.yaml` 中，每个软件以 `packages:` 开头，随后在下方对各个软件进行定义。当需要增加本地安装软件时，在定义 中增加 `externals:` 关键字，其中需要包括安装说明 `- spec:` 以及安装目录 `prefix:` 定义。

```yaml
packages:
  openmpi:
    externals:
    - spec: "openmpi@1.4.3%gcc@4.4.7 arch=linux-debian7-x86_64"
      prefix: /opt/openmpi-1.4.3
    - spec: "openmpi@1.4.3%gcc@4.4.7 arch=linux-debian7-x86_64+debug"
      prefix: /opt/openmpi-1.4.3-debug
    - spec: "openmpi@1.6.5%intel@10.1 arch=linux-debian7-x86_64"
      prefix: /opt/openmpi-1.6.5-intel
```

在配置文件中指定本地安装软件时，并不会阻止 spack 再安装最新版本的软件。如果需要指定 spack 只使用 本地安装软件，可以在配置参数中指定软件不可构建 `buildable: False`。此时 spack 只会使用本地安装版本构建其他程序。

```yaml
packages:
  openmpi:
    externals:
    - spec: "openmpi@1.4.3%gcc@4.4.7 arch=linux-debian7-x86_64"
      prefix: /opt/openmpi-1.4.3
    - spec: "openmpi@1.4.3%gcc@4.4.7 arch=linux-debian7-x86_64+debug"
      prefix: /opt/openmpi-1.4.3-debug
    - spec: "openmpi@1.6.5%intel@10.1 arch=linux-debian7-x86_64"
      prefix: /opt/openmpi-1.6.5-intel
    buildable: False
```

为了能够在 spack 中使用本地安装软件，需要在配置好 `packages.yaml` 文件后，使用 `spack install` 命令将本地安装软件集成在 spack 中，此时会直接对软件进行相关配置，不会再进行编译和安装。

```bash
$ spack install llvm
==> Warning: Missing a source id for llvm@13.0.0
[+] /opt/rocm/llvm (external llvm-13.0.0-otzjpuamhoxaiq4asf3cwqqki7kmayv6)
```

### 自动查找本地软件

除了手动修改 `packages.yaml` 配置文件外，也可以使用 `spack external find` 命令来自动搜索本地安装软件包并添加到 `packages.yaml` 文件中。
运行完毕后，配置文件 `packages.yaml` 会进行相应的修改，随后可以手动进行一步修改，然后再进行安装。

```bash
$ spack external find llvm
==> The following specs have been detected on this system and added to /home/lilx/.spack/packages.yaml
llvm@13.0.0
```

## 镜像