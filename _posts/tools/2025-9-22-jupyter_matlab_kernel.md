---
layout: post
title: Matlab kernel for jupyter
date: 2025-9-22
categories: tools
---

本文对jupyter内部署matlab kernel方法进行介绍。

# 软件部署

## conda 环境设置

创建 conda 虚拟环境，避免软件设置对其他配置造成影响。

```bash
$ conda create -n matlab python=3.9
$ source activate matlab
```

## matlab engine

进入 matlab 目录 `extern/engines/python` 安装 MATLAB Engine API for Python。本次使用是 matlab R2022a版本，对应安装目录为 `/mnt/beegfs/lilongxiang/app/matlab_r2022a`，则完整路径为 `/mnt/beegfs/lilongxiang/app/matlab_r2022a/extern/engines/python/`。

使用如下命令将 matlab engine 安装在用户目录 `lib` 下。注意 `setup.py` 脚本要求将 `setuptools` 模块使用如下命令回退到特定版本，否则会出现如下 `Invalid version: 'R2022a'` 报错。

```bash
pip install setuptools==65.0.0
```

```
$ python setup.py build -b ${HOME}
Traceback (most recent call last):
  File "/mnt/beegfs/lilongxiang/app/matlab_r2022a/extern/engines/python/setup.py", line 80, in <module>
    setup(
  File "/mnt/beegfs/lilongxiang/app/miniconda3/envs/matlab/lib/python3.9/site-packages/setuptools/_distutils/core.py", line 148, in setup
    _setup_distribution = dist = klass(attrs)
  File "/mnt/beegfs/lilongxiang/app/miniconda3/envs/matlab/lib/python3.9/site-packages/setuptools/dist.py", line 334, in __init__
    self.metadata.version = self._normalize_version(self.metadata.version)
  File "/mnt/beegfs/lilongxiang/app/miniconda3/envs/matlab/lib/python3.9/site-packages/setuptools/dist.py", line 370, in _normalize_version
    normalized = str(Version(version))
  File "/mnt/beegfs/lilongxiang/app/miniconda3/envs/matlab/lib/python3.9/site-packages/setuptools/_vendor/packaging/version.py", line 202, in __init__
    raise InvalidVersion(f"Invalid version: {version!r}")
packaging.version.InvalidVersion: Invalid version: 'R2022a'
```

正常安装完成后，可以看到 `${HOME}/lib/matlab` 目录。将此目录移动至 conda 环境默认软件安装路径下，即 `~/app/miniconda3/envs/matlab/lib/python3.9/site-packages/`，对应命令如下。

```bash
$ mv ~/lib/matlab ~/app/miniconda3/envs/matlab/lib/python3.9/site-packages/
```

## matlab_kernel

使用如下命令安装并配置 matlab_kernel

```bash
$ pip install matlab_kernel
$ python -m matlab_kernel install --user
```

# 测试

测试 matlab engine 是否可以在 python 内正常工作。运行如下命令，查看是否报错：

```bash
$ python
Python 3.9.23 (main, Jun  5 2025, 13:40:20)
[GCC 11.2.0] :: Anaconda, Inc. on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import matlab.engine
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/mnt/beegfs/lilongxiang/app/miniconda3/envs/matlab/lib/python3.9/site-packages/matlab/__init__.py", line 214, in <module>
    from matlabmultidimarrayforpython import double, single, uint8, int8, uint16, \
ImportError: /mnt/beegfs/lilongxiang/app/matlab_r2022a/extern/engines/python/../../../extern/bin/glnxa64/../../../sys/os/glnxa64/libstdc++.so.6: version `GLIBCXX_3.4.26' not found (required by /mnt/beegfs/lilongxiang/app/matlab_r2022a/extern/engines/python/../../../extern/bin/glnxa64/matlabmultidimarrayforpython.cpython-39-x86_64-linux-gnu.so)
```

此问题是 matlab 自带 `libstdc++.so.6` 版本过低所致，可以将系统内 `/usr/lib64/libstdc++.so.6` 直接将此文件替换，命令如下：

```bash
# 备份 libstdc++.so.6
$ mv /mnt/beegfs/lilongxiang/app/matlab_r2022a/extern/engines/python/../../../extern/bin/glnxa64/../../../sys/os/glnxa64/libstdc++.so.6 /mnt/beegfs/lilongxiang/app/matlab_r2022a/extern/engines/python/../../../extern/bin/glnxa64/../../../sys/os/glnxa64/libstdc++.so.bk
# 覆盖文件
$ cp /usr/lib64/libstdc++.so.6 /mnt/beegfs/lilongxiang/app/matlab_r2022a/extern/engines/python/../../../extern/bin/glnxa64/../../../sys/os/glnxa64/libstdc++.so.6
```

再次测试，此时无保存内容出现。

```bash
$ python
Python 3.9.23 (main, Jun  5 2025, 13:40:20)
[GCC 11.2.0] :: Anaconda, Inc. on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import matlab.engine
>>>
```

# jupyter 使用

在 vscode 内 terminal 内加载 conda 的 matlab 虚拟环境，并启动 jupyter lab。

```bash
$ jupyter lab
[I 2025-09-22 03:41:41.397 ServerApp] jupyter_lsp | extension was successfully linked.
[I 2025-09-22 03:41:41.404 ServerApp] jupyter_server_terminals | extension was successfully linked.
[I 2025-09-22 03:41:41.412 ServerApp] jupyterlab | extension was successfully linked.
[I 2025-09-22 03:41:41.703 ServerApp] notebook_shim | extension was successfully linked.
[I 2025-09-22 03:41:41.729 ServerApp] notebook_shim | extension was successfully loaded.
[I 2025-09-22 03:41:41.731 ServerApp] jupyter_lsp | extension was successfully loaded.
[I 2025-09-22 03:41:41.731 ServerApp] jupyter_server_terminals | extension was successfully loaded.
[I 2025-09-22 03:41:41.736 LabApp] JupyterLab extension loaded from /mnt/beegfs/lilongxiang/app/miniconda3/envs/matlab/lib/python3.9/site-packages/jupyterlab
[I 2025-09-22 03:41:41.736 LabApp] JupyterLab application directory is /mnt/beegfs/lilongxiang/app/miniconda3/envs/matlab/share/jupyter/lab
[I 2025-09-22 03:41:41.737 LabApp] Extension Manager is 'pypi'.
[I 2025-09-22 03:41:41.784 ServerApp] jupyterlab | extension was successfully loaded.
[I 2025-09-22 03:41:41.784 ServerApp] Serving notebooks from local directory: /mnt/beegfs/lilongxiang/project/pr-fvcom
[I 2025-09-22 03:41:41.784 ServerApp] Jupyter Server 2.16.0 is running at:
[I 2025-09-22 03:41:41.784 ServerApp] http://localhost:8888/lab?token=9e855eae07147035144291dd9454fa913713aa44ccfa3792
[I 2025-09-22 03:41:41.784 ServerApp]     http://127.0.0.1:8888/lab?token=9e855eae07147035144291dd9454fa913713aa44ccfa3792
[I 2025-09-22 03:41:41.785 ServerApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 2025-09-22 03:41:41.791 ServerApp] 
    
    To access the server, open this file in a browser:
        file:///mnt/beegfs/lilongxiang/.local/share/jupyter/runtime/jpserver-724262-open.html
    Or copy and paste one of these URLs:
        http://localhost:8888/lab?token=9e855eae07147035144291dd9454fa913713aa44ccfa3792
        http://127.0.0.1:8888/lab?token=9e855eae07147035144291dd9454fa913713aa44ccfa3792
[I 2025-09-22 03:41:41.824 ServerApp] Skipped non-installed server(s): bash-language-server, dockerfile-language-server-nodejs, javascript-typescript-langserver, jedi-language-server, julia-language-server, pyright, python-language-server, python-lsp-server, r-languageserver, sql-language-server, texlab, typescript-language-server, unified-language-server, vscode-css-languageserver-bin, vscode-html-languageserver-bin, vscode-json-languageserver-bin, yaml-language-server
```

由于 vsocode 可以自动转发服务器端口，此时点击 `http://localhost:8888/lab?token=9e855eae07147035144291dd9454fa913713aa44ccfa3792` 后即可在本地浏览器内打开 jupyter 网页，并支持进行画图等操作。
