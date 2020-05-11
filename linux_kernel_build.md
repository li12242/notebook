# linux 内核编译

## 下载

首先在 [kernel.org](https://www.kernel.org/) 网站上下载最新的内核版本 5.6.12。选择 tarball 版本下载 https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.6.12.tar.xz。

## 编译

### 模块设置

下载完成后解压，进入解压后压缩包

    $ tar -xvf linux-5.6.12.tar.xz
    $ cd linux-5.6.12

在正式编译之前，首先配置需要包含哪些模块。一个简单方法是拷贝当前内核配置文件，然后使用 `menuconfig` 命令来做必要的更改。

    $ cp /boot/config-3.10.0-693.el7.x86_64 .config

在编译之前，需要安装一些必须的依赖包。

    # yum install ncurses-devel flex bison gcc-c++ openssl-devel elfutils-libelf-devel

使用 `menuconfig` 模式进行模块选择

    # make menuconfig

在最后一行选择 `Load` 选项，加载当前 `.config` 文件内配置。随后选择 `Save` 选项，储存当前设置后，即可退出。

![menuconfig setting](http://ww1.sinaimg.cn/large/7a1c18a8ly1geolfbdbcyj20s40dwq5p.jpg)

![menuconfig load .config.png](http://ww1.sinaimg.cn/large/7a1c18a8ly1geolgdd9s7j20s40dw75l.jpg)

### 编译

在编译时，必须使用高于 6.8.0 版本的 gcc 和 g++ 编译器进行编译。
此外，运行时必须使用 root 账户进行编译。由于编译过程较长，可以使用并行编译命令

    # make -j8

### 安装

编译完成后，有的配置选项编译进核心，有的编译成模块，所以安装时需要将模块和核心分别安装。

    make modules_install
    make install

执行完毕后，可以在 `/boot/` 目录下看到新编译的内核。
    
```bash
# ls /boot/vmlinuz*
```

## 引导更新

在安装好内核后，需要修改 `/boot/grub2/grub.cfg` 文件来更新引导。推荐使用 `grub2-mkconfig` 命令，其会根据 `/boot` 目录下内核文件自动更新 `grub` 文件。

    grub2-mkconfig -o /boot/grub2/grub.cfg

执行完毕后，下次启动即可选择新编译的内核。如想要修改默认启动的内核，可以修改 `/etc/default/grub` 文件。
