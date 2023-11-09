---
layout: post
title: Sphinx + matlab
date: 2023-11-09
categories: tools
---

# Sphinx 介绍

Sphinx 是一个基于 Python 的文档生成项目。最早只是用来生成 Python 的项目文档，使用 reStructuredText 格式。但随着 Sphinx 项目的逐渐完善，目前已发展成为一个大众可用的框架，很多非 Python 的项目也采用 Sphinx 作为文档写作工具，甚至完全可以用 Sphinx 来写书。

## reStructuredText

reStructuredText 是一种轻量级标记语言。它是 Python Doc-SIG（Documentation Special Interest Group）的 Docutils 项目的一部分，旨在为 Python 创建一组类似于 Java 的 Javadoc 或 Perl 的 Plain Old Documentation（pod）的工具。Docutils 可以从 Python 程序中提取注释和信息，并将它们格式化为各种形式的程序文档。

前面提到，Sphinx 使用 reST 作为标记语言。实际上，reST 与 Markdown 非常相似，都是轻量级标记语言。由于设计初衷不同，reST 的语法更为复杂一些。

# Matlab文档环境搭建

## 软件部署

sphinx依赖多个python包，为了保证环境的稳定性，可以使用conda构建专门环境，命名为sphinx：

```
conda create --name=sphinx python=3.9
conda activate sphinx
```

进入conda环境后，使用pip3命令安装Sphinx和对应python软件包。由于大部分软件都需要使用pip3安装，可以设置对应配置文件，使用国内pip源加速安装过程。

```
$ mkdir ~/.pip 
$ cat ~/.pip/pip.conf
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn
```

使用如下命令安装对应软件包

```
pip3 install -U Sphinx
pip3 install sphinx-autobuild
pip3 install sphinx_rtd_theme
pip3 install recommonmark
pip3 install sphinx_markdown_tables
```

安装完成后，系统会增加一些 sphinx- 开头的命令。

```
sphinx-apidoc    sphinx-autobuild    sphinx-autogen    sphinx-build    sphinx-quickstart
```

为了使sphinx支持matlab，还需要安装如下模块

```
pip install sphinxcontrib-matlabdomain 
```

## 文档项目创建

下面将以Matlab项目NDG-FEM为例，介绍Sphinx文档的创建过程。

首先进入doc目录，执行sphinx-quickstart快速构建文档框架，出现如下对话窗口：

```
$ sphinx-quickstart
Welcome to the Sphinx 7.2.6 quickstart utility.

Please enter values for the following settings (just press Enter to
accept a default value, if one is given in brackets).

Selected root path: .

You have two options for placing the build directory for Sphinx output.
Either, you use a directory "_build" within the root path, or you separate
"source" and "build" directories within the root path.
> Separate source and build directories (y/n) [n]: y

The project name will occur in several places in the built documentation.
> Project name: NDG-FEM
> Author name(s): li12242
> Project release []: v1

If the documents are to be written in a language other than English,
you can select a language here by its language code. Sphinx will then
translate text that it generates into that language.

For a list of supported codes, see
https://www.sphinx-doc.org/en/master/usage/configuration.html#confval-language.
> Project language [en]: zh_CN

Creating file /home/li12242/project/NDG-FEM/doc/source/conf.py.
Creating file /home/li12242/project/NDG-FEM/doc/source/index.rst.
Creating file /home/li12242/project/NDG-FEM/doc/Makefile.
Creating file /home/li12242/project/NDG-FEM/doc/make.bat.

Finished: An initial directory structure has been created.

You should now populate your master file /home/li12242/project/NDG-FEM/doc/source/index.rst and create other documentation
source files. Use the Makefile to build the docs, like so:
   make builder
where "builder" is one of the supported builders, e.g. html, latex or linkcheck.
```

在上述配置选项中，选择了`Separate source and build directories`，因此在doc目录下将出现build和source两个目录。前者构建生成的各种文件，后者存放文档资源。

为了能够快速访问生成的html文档，可以通过sphinx-autobuild工具启动http服务。

```
$  sphinx-autobuild source build/html
[sphinx-autobuild] > sphinx-build /home/li12242/project/NDG-FEM/doc/source /home/li12242/project/NDG-FEM/doc/build/html
Running Sphinx v7.2.6
/home/li12242/project/NDG-FEM/NdgCell
loading translations [zh_CN]... done
loading pickled environment... done
building [mo]: targets for 0 po files that are out of date
writing output...
building [html]: targets for 0 source files that are out of date
updating environment: 0 added, 1 changed, 0 removed
reading sources... [100%] index
looking for now-outdated files... none found
pickling environment... done
checking consistency... done
preparing documents... done
copying assets... copying static files... done
copying extra files... done
done
writing output... [100%] index
generating indices... genindex mat-modindex done
writing additional pages... search done
dumping search index in Chinese (code: zh)... done
dumping object inventory... done
build succeeded.

The HTML pages are in build/html.
[I 231109 18:38:40 server:335] Serving on http://127.0.0.1:8000
[I 231109 18:38:40 handlers:62] Start watching changes
[I 231109 18:38:40 handlers:64] Start detecting changes
```

运行后通过浏览器打开网页`http://127.0.0.1:8000`即可查看生成的文档。需要注意的是，由于使用的是本地访问方式，所以针对latex等数学公式无法使用mathjax渲染。

为了能够在本地浏览公式，可以将Mathjax下载到本地。下载MathJax-v3.x版本软件包(https://github.com/mathjax/MathJax/archive/refs/tags/3.2.2.tar.gz)，将其中`es5`目录整体拷贝至`source/_static`目录内。随后修改conf.py文件，在其中加入如下定义：

```
 mathjax_path = "es5/tex-chtml.js"
```

注意不同版本MathJax目录内，`tex-chtml.js`文件的路径可能会有不同。更新conf.py文件后，再次运行sphinx-autobuild命令，可以看到在本地也可渲染数学公式了。

<img src="/notebook/assets/sphinx-matlab/example_StdCell.png" alt="StdCell说明文档"/>

## 项目配置修改

首先可以对sphinx主题进行修改，其中最常用的是sphinx_rtd_theme。打开`source/conf.py`文件，修改对应内容如下：

```
html_theme = 'sphinx_rtd_theme'
```

为了能够支持matlab文件，需要另外对配置文件进行如下修改：

```
import os
matlab_src_dir = os.path.abspath('../../NdgCell')
```

变量`matlab_src_dir`指明了matlab文档位置。由于`conf.py`文件位于`doc/source`目录下，因此NDG-FEM对应代码文件的相应路径应该为`../../`。为了能够对单个模块测试，所以仅生成`NdgCell`目录内文件注释内容。

```
extensions = [
    'sphinxcontrib.matlab', 
    'sphinx.ext.autodoc',
    'sphinx.ext.mathjax',
    'recommonmark'
    ]
primary_domain = "mat"
source_suffix = {
    '.rst': 'restructuredtext',
    '.txt': 'restructuredtext',
    '.md': 'markdown',
}
```

在加载模块部分，增加了`sphinxcontrib.matlab`、`sphinx.ext.autodoc`和`sphinx.ext.mathjax`模块支持。另外针对不同类型文件，通过`source_suffix`指明文件类型。

## 文档撰写

### 文档添加

在`NdgCell`目录内，包含如下文件和目录

```
$ ls NdgCell 
@StdCell  @StdLine  @StdPoint  @StdPrismQuad  @StdPrismTri  @StdQuad  @StdTri  UnitTest  enumStdCell.m
```

在`source/index.rst`文件内，以`@StdCell`为例增加对应文档内容。首先增加对应章节，内容如下

```
@StdCell
++++++++

This is the test class folder

.. module:: @StdCell

.. autoclass:: StdCell
   :show-inheritance:
   :members:
```

在模块具体内容中，采用`autoclass​`等命令自动读取对应.m文件中注释，并生成对应文档。查看html文件内对应内容如下所示，可以看出，类型对应属性和说明都自动生成。

<img src="/notebook/assets/sphinx-matlab/example_StdCell.png" alt="StdCell说明文档"/>

# reStructuredText语法介绍

## Matlab文件注释



# 参考文献

1. [Sphinx + Read the Docs 从懵逼到入门](https://zhuanlan.zhihu.com/p/264647009)