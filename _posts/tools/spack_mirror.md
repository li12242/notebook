# Spack 镜像制作

以 MUMPS 和 STRUMPACK 软件安装过程为例，首先在本地使用制作 MUMPS 和 STRUMPACK 镜像，并且包含所有依赖库。

```bash
$ spack mirror create --dependencies strumpack~cuda+scotch+shared~butterflypack mumps+metis+parmetis+scotch
......
==> Successfully updated mirror in file:///home/li12242/app/spack/var/spack/cache
  Archive stats:
    0    already present
    40   added
    0    failed to fetch.
```

可以看到安装完成后，显示有 40 个软件包添加到镜像目录内，默认镜像文件夹路径为 `<SPACK_DIR>/var/spack/cache`，其中 `<SPACK_DIR>` 为软件的安装路径。在镜像文件夹内，`_source-cache` 保存了所有压缩包文件，在其他文件夹内，则各自保存了不同版本的软件，并且包含了链接指向 `_source-cache` 内对应的压缩包。查看文件夹具体内容如下

```
$ tree cache
cache/
├── _source-cache
│   └── archive
│       ├── 04
│       │   └── 0484d275f87e9b8641ff2eecaa9df2830cbe276ac79ad80494822721de6e1693.tar.gz
│       ├── 09
│       │   └── 09c22e5c6fef327d3e48eb23f0d610dcd3a35ab9207f12e0f875701c677978d3
......
......
│           └── ff3922af377d514eca302a6662d470e857bd1a591e96a2050500df5a9d59facf.tar.gz
├── arpack-ng
│   └── arpack-ng-3.7.0.tar.gz -> ../_source-cache/archive/97/972e3fc3cd0b9d6b5a737c9bf6fd07515c0d6549319d4ffb06970e64fa3cc2d6.tar.gz
├── autoconf
│   └── autoconf-2.69.tar.gz -> ../_source-cache/archive/95/
......
```

镜像文件夹制作完成后，将此 `cache` 文件夹打包为 `spack-mirror.tar.gz` 压缩包。

# Spack 镜像使用

在本地将镜像文件 `spack-mirror.tar.gz` 文件解压，并且重命名为 `spack-mirror` 文件夹。
运行如下命令向 spack 中添加本地镜像路径。

```bash
$ spack mirror add local_mirror file://<PATH_TO_MIRROR>/spack-mirror
```

添加完成后，运行 `spack mirror list` 查看添加的本地路径。

```bash
$ spack mirror list
spack-public    https://spack-llnl-mirror.s3-us-west-2.amazonaws.com/
local_mirror    file:///public/home/lilongxiang/software/spack-mirror
```

此时，在本地运行 `strumpack~cuda+scotch+shared~butterflypack` 即可使用本地软件源进行安装，不再需要从外网下载。