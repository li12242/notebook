---
layout: post
title: pip 与 conda
date: 2020-5-7
categories: tools
---
# pip 与 conda

## 简介

pip 是接触 python 后最早认识的包管理工具。通过使用 pip 能够自动下载和解决不同 python 模块的依赖问题，使 python 的配置过程变得简单。

与 pip 类似，conda 也是一个开源软件的包管理系统和环境管理系统。conda 可分为 anaconda 和 miniconda，anaconda 包含一些科学计算常用的 python 包，miniconda 为精简版。conda 安装的包是二进制版本，因此其作用不仅包括安装 python 包，还可以安装任何语言编写的程序。此外，还有报告表明 conda 安装的软件能够提供更好的性能[^134]。conda 的出现解决了 pip 只能安装 python 包的问题，对于 HDF5、MKL、LLVM 等不提供 setup.py 的安装模式的软件也可以直接安装。关于 conda 和 pip 的不同区别可以看下表[^12]

[^134]: https://zhuanlan.zhihu.com/p/46599887

[^12]: https://www.jianshu.com/p/5601dab5c9e5

|    类别    |        conda         |        pip         |
| :--------: | :------------------: | :----------------: |
|    管理    |        二进制        |    wheel 或源码    |
| 需要编译器 |          no          |        yes         |
|    语言    |         any          |       Python       |
|  虚拟环境  |         支持         | virtualenv 或 venv |
| 依赖性检查 |         yes          |      用户选择      |
|   包来源   | Anaconda repo和cloud |        PyPi        |


## pip 使用

由于 conda 现在已经可以实现 pip 全部功能，因此并不推荐继续使用 pip，但是这里还是介绍一些常用命令，方便用户将用 pip 安装的软件进行卸载。

```bash
> pip list 													# 查看安装软件列表
> pip uninstall <pkg_name>					# 删除软件
> pip install <pkg_name>						# 安装软件
> pip install --upgrade <pkg_name> 	# 升级软件
```

## conda 使用

conda 使用时提供了三个概念：虚拟环境、通道和包。

通过虚拟环境，conda 提供了 python 编程时环境隔离功能。conda 的环境与 module 环境管理不同，conda 更多用于管理不同版本的 python 和不同版本包。在不同环境中，安装的包，如 numpy、scipy、torch、tensorflow 可以互不相同，只要虚拟环境内部保证软件的依赖得到满足即可。

下面介绍 conda 使用时常用的命令。创建和激活环境的命令为

```python
> conda create -n <env_name> python=3.6 # 使用特定 py 版本新建环境
> conda activate <env_name>             # 激活环境
> conda deactivate <env_name>           # 退出环境
> conda remove -n <env_name> --all      # 删除环境及所有包
```

通道（channel）指包下载包时使用的镜像源网址。在国内可以使用清华镜像网站提高下载速度

```python
> conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
> conda config --remove channels <channel_name or url>
```

conda 在解析环境时，由于要将所有 channel 扫一遍，在遇到 repo.continuum 时 会非常慢。根据清华源的文档设置镜像后，可进一步手动修改 `~/.condarc` 屏蔽默认的 channel (defaults) 以提高解析速度。

```bash
channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
  - defaults  << 去掉这一行！
show_channel_urls: true
```

安装软件包相关命令包括

```python
> conda install -c <channel_name> -n <env_name> <pkg_name>
> conda uninstall -c <channel_name> -n <env_name> <pkg_name>
```

## 集群中 conda 配置使用

在集群中，intel-2020 版本的 intelpython3 自带了 conda 包。使用如下命令初始化 intelpython 环境

```bash
> source /opt/intel/2020/intelpython3/bin/activate
```

此时，查看 conda 是否为 intel 包中软件

```bash
> which conda
/opt/intel/2020/intelpython3/condabin/conda
```

输入 conda 初始化命令

```bash
> conda init
no change     /opt/intel/2020/intelpython3/condabin/conda
no change     /opt/intel/2020/intelpython3/bin/conda
no change     /opt/intel/2020/intelpython3/bin/conda-env
no change     /opt/intel/2020/intelpython3/bin/activate
no change     /opt/intel/2020/intelpython3/bin/deactivate
no change     /opt/intel/2020/intelpython3/etc/profile.d/conda.sh
no change     /opt/intel/2020/intelpython3/etc/fish/conf.d/conda.fish
no change     /opt/intel/2020/intelpython3/shell/condabin/Conda.psm1
no change     /opt/intel/2020/intelpython3/shell/condabin/conda-hook.ps1
no change     /opt/intel/2020/intelpython3/lib/python3.7/site-packages/xontrib/conda.xsh
no change     /opt/intel/2020/intelpython3/etc/profile.d/conda.csh
modified     /home/lilongxiang/.bashrc
No action taken.
```

可以看到，此时个人目录下 `.bashrc` 文件已经修改，添加了如下 conda 初始化命令。

```bash
# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/opt/intel/2020/intelpython3/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/opt/intel/2020/intelpython3/etc/profile.d/conda.sh" ]; then
        . "/opt/intel/2020/intelpython3/etc/profile.d/conda.sh"
    else
        export PATH="/opt/intel/2020/intelpython3/bin:$PATH"
    fi
fi
unset __conda_setup
# <<< conda initialize <<<
```

当使用 zshell 时，需要把上面的 `shell.bash` 改为 `zshell`，然后放入 `.zshrc` 配置文件中。

下面以 pandoc 为例，介绍 conda 安装软件过程。输入如下命令开始安装

```bash
> conda install pandoc
Collecting package metadata (current_repodata.json): done
Solving environment: done

## Package Plan ##

  environment location: /opt/intel/2020/intelpython3

  added / updated specs:
    - pandoc


The following NEW packages will be INSTALLED:

  gmp                pkgs/main/linux-64::gmp-6.1.2-h6c8ec71_1
  pandoc             pkgs/main/linux-64::pandoc-2.2.3.2-0


Proceed ([y]/n)?
```

输入 yes 后显示安装并没有成功。

```bash
Preparing transaction: done
Verifying transaction: failed

EnvironmentNotWritableError: The current user does not have write permissions to the target environment.
  environment location: /opt/intel/2020/intelpython3
  uid: 1014
  gid: 1015
```

其根本原因是在当前环境下（base），用户没有权限执行安装命令。解决办法是新建一个虚拟环境 `myenv`，新的环境默认位置为 `${HOME}/.conda/envs/myenv`。此时，切换到新环境后再进行安装即不再有权限问题。

```bash
> conda install pandoc
Collecting package metadata (current_repodata.json): done
Solving environment: done

## Package Plan ##

  environment location: /home/lilongxiang/.conda/envs/myenv

  added / updated specs:
    - pandoc


The following NEW packages will be INSTALLED:

  gmp                pkgs/main/linux-64::gmp-6.1.2-h6c8ec71_1
  intelpython        conda_channel/linux-64::intelpython-2020.0-1
  libgcc-ng          conda_channel/linux-64::libgcc-ng-9.1.0-hdf63c60_0
  libstdcxx-ng       conda_channel/linux-64::libstdcxx-ng-9.1.0-hdf63c60_0
  pandoc             pkgs/main/linux-64::pandoc-2.2.3.2-0
  zlib               conda_channel/linux-64::zlib-1.2.11-h14c3975_7


Proceed ([y]/n)? y

Preparing transaction: done
Verifying transaction: done
Executing transaction: done
(myenv)
```
