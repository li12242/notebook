# pandoc 安装与使用

Makedown 是当前一种应用广泛的标记语言，其语法简单，但是功能强大，可以在文本中添加图片、超链接、引用、注释和公式等。本文档也是采用 Markdown 所撰写。

Markdown 文档结果的展示需要使用专用编译器，或者安装特定插件。
目前，PDF 文档是一种更通用的文档格式，因此若能够将撰写好的 Markdown 脚本转换为 PDF 文件，就能够结合 Markdown 编译的简便性和 PDF 文档的通用性，使得文档编写和修改过程效率更高。

本文介绍的 pandoc 就是一款功能强大的文档转换工具，其能够在多种文档格式之间进行转换，特别是将 Markdown 文档转换为符合要求的 PDF 文件。
本文将对 pandoc 的安装、配置和使用进行相应介绍。

## pandoc 安装

pandoc 可以使用 conda 安装，关于 conda 的安装和使用可以参考前一篇文档 [conda_pip_info](https://li12242.github.io/notebook/tools/conda_pip_info)。
使用如下命令安装 pandoc

```bash
$ conda install pandoc
```

安装完毕后，还需要安装 pandoc-xnos 插件，主要功能是图片、表格、公式等编号的索引，此软件包采用 python 编写，使用 pip 命令安装。

```
$ pip install pandoc-xnos
```

由于 pandoc 生成 PDF 文件需要使用 latex 工具，因此还需要安装 texlive 软件，推荐在 linux 平台直接使用 yum 或者 apt 命令安装。

## 参数设置

### 中文支持

pandoc 安装完成后即可使用如下命令将文档进行转换，但是当文档内有中文是会出现运行错误，需要指定 latex 编译器为 xelatex。

```
$ pandoc jorek_installation.md -o jorek_installation.pdf
Error producing PDF.
! Package inputenc Error: Unicode character 安 (U+5B89)
(inputenc)                not set up for use with LaTeX.

See the inputenc package documentation for explanation.
Type  H <return>  for immediate help.
 ...

l.94 ...��}\label{jorek-ux5b89ux88c5ux624bux518c}}

Try running pandoc with --pdf-engine=xelatex.
```

添加 `--pdf-engine=xelatex` 再次进行编译，虽然编译不再出现错误，但是会显示如下警告，并且编译好的 PDF 文件中并没有中文。

```
$ pandoc jorek_installation.md --pdf-engine=xelatex -o jorek_installation.pdf
[WARNING] Missing character: There is no 安 (U+5B89) in font [lmroman12-bold]:mapping=tex-text;!
[WARNING] Missing character: There is no 装 (U+88C5) in font [lmroman12-bold]:mapping=tex-text;!
......
```

![pandoc_xelatex_no_chinese.PNG](http://ww1.sinaimg.cn/large/7a1c18a8ly1gj1tqlofcdj20n10tomzb.jpg)

这是因为 xelatex 选择的默认字体不支持中文字符，修改 pandoc 命令，添加字体设置参数 `-V CJKmainfont="SimSun"`，并再次对文件进行编译。
编译结果文件如下，可以看出文档内中文文字已经可以正确显示，但是其代码并没有自动换行，此外文档题目和各小节标题也不够规范，需要进一步修改。

![pandoc_xelatex_unlimit_codeline.PNG](http://ww1.sinaimg.cn/large/7a1c18a8ly1gj1ttlbrxhj20n10toad6.jpg)

### 题目和标题设置

想要和标准文档一样正确设置标题，需要在 Markdown 文件头部添加 YAML 格式设置参数。例如

```yaml
---
title: "Jorek 软件性能分析"
author: 李龙翔
date: "2020-09-23"
subject: "Markdown"
keywords: [Jorek, APS, VTune]
---
```

向 Markdown 文件中添加如上设置后，重新使用 pandoc 对文档进行转换，得到 PDF 文件如下所示。可以看出，此时文档中包含了标题、作者和日期等信息，比原始文档规范很多。

![pandoc_add_title.PNG](http://ww1.sinaimg.cn/large/7a1c18a8ly1gj1u72ominj20n10todia.jpg)

### 代码块设置

除了题目和标题之外，在原始 PDF 文档中还存在代码无法自动换行的问题。为了解决此问题需要修改 pandoc 转换后的 latex 文件内参数，因此主要有两种方法：

1. 使用 `pandoc -D latex > ~/.pandoc/default.latex` 导出 pandoc 默认的 latex 配置模板并进行修改，添加 listing 模块相应参数。
2. 在 YAML 参数设置中直接添加 listing 模块设置参数。

相比较而言，第二种方法更为简单。在 Markdown 文件头中，添加如下配置，并且在编译时添加 `--listings` 参数

```YAML
---
header-includes:
 - \lstset{breaklines=true}
 - \lstset{keywordstyle=\bfseries}
 - \lstset{commentstyle=\rmfamily\itshape}
 - \lstset{flexiblecolumns}
 - \lstset{showspaces=false}
 - \lstset{frame=shadowbox}
---
```

编译完成后，对应 PDF 文件如下。可以看出，此时文档内代码已经能够自动换行，但是由于没有设置代码字体，代码和文档都为宋体，可以使用 lstset 命令进一步修改。

![pandoc_listing.PNG](http://ww1.sinaimg.cn/large/7a1c18a8ly1gj1urxx7xcj20n10towgy.jpg)

### 编号和索引

当文档内包含图片时，可能需要对图片进行索引。插件 pandoc-xnos 包含了图片、表格、公式等内容的索引功能。
以图片为例，在添加索引时需要在图片说明后添加花括号，括号内用 `#` 开头指定索引标记。
在引用此图片时，则直接使用 `@` + 索引标记的模式调用。

```markdown
![APS 监控144进程并行计算过程中进程间通信时间矩阵图](figure/aps-report-p114-r2r-comm-matrix.png){#fig:aps_report}

图 @fig:aps_report 显示了 144 进程计算时，进程间通信耗时，其中每个格点包含显示两个进程间通信情况。
......
```

## 总结

在上面内容中，对 pandoc 的安装和在转换 Markdown 文件为 PDF 文件过程的设置参数进行了介绍。
通过修改 pandoc 的一些默认参数，我们可以将 Markdown 文档直接转换为格式整洁的 PDF 文档。
这个功能能够大大简化文档撰写过程，并且使得最终输出文档满足特定需求。
