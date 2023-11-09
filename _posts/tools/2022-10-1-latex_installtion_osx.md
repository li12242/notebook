---
layout: post
title: OSX 平台 VSCode + Latex 配置
date: 2022-10-1
categories: tools
---

## 下载

在 OSX 平台，安装 latex 可以通过命令行形式选择 texlive，但是更推荐是 MaxTex。后者可以在安装完成后，自动配置 latex 工具的环境，并且安装一些非常有用的 App 工具，如 latexit 等。

最新版 MacTex 下载可以从直接从官网下载（https://www.tug.org/mactex/），国内的话还可以从清华等镜像网站下载（https://mirrors.tuna.tsinghua.edu.cn/CTAN/）。
选择网页中 “MacTEX” 链接即可进入文件列表中进行下载。

<img src="/notebook/assets/latex-for-osx/CTAN_web.png" alt="CTAN网站列表" align="middle" width="320"/>

下载完 dmg 镜像文件后，直接点击安装即可。

## 配置

在 VSCode 内安装 “Latex Workshop” 模块。为了能够支持中文，在配置文件中进行如下修改，使用 xelatex 代替默认的 pdflatex。

```json
  "latex-workshop.latexindent.path": "/Library/TeX/texbin/latexindent",
  "latex-workshop.latex.tools": [
    {
      "name": "latexmk",
      "command": "latexmk",
      "args": [
        "-synctex=1",
        "-interaction=nonstopmode",
        "-file-line-error",
        "-pdf",
        "%DOC%"
      ]
    },
    {
      "name": "xelatex",
      "command": "xelatex",
      "args": [
        "-synctex=1",
        "-interaction=nonstopmode",
        "-file-line-error",
        "%DOC%"
      ]
    },
    {
      "name": "pdflatex",
      "command": "pdflatex",
      "args": [
        "-synctex=1",
        "-interaction=nonstopmode",
        "-file-line-error",
        "%DOC%"
      ]
    },
    {
      "name": "bibtex",
      "command": "bibtex",
      "args": [
        "%DOCFILE%"
      ]
    }
  ],
  "latex-workshop.latex.recipes": [
    {
      "name": "xelatex",
      "tools": [
        "xelatex"
      ]
    },
    {
      "name": "latexmk",
      "tools": [
        "latexmk"
      ]
    },
    {
      "name": "xelatex -> bibtex -> xelatex*2",
      "tools": [
        "xelatex",
        "bibtex",
        "xelatex",
        "xelatex"
      ]
    }
  ],
  "latex-workshop.view.pdf.viewer": "tab",
  "latex-workshop.latex.clean.fileTypes": [
    "*.aux",
    "*.bbl",
    "*.blg",
    "*.idx",
    "*.ind",
    "*.lof",
    "*.lot",
    "*.out",
    "*.toc",
    "*.acn",
    "*.acr",
    "*.alg",
    "*.glg",
    "*.glo",
    "*.gls",
    "*.ist",
    "*.fls",
    "*.log",
    "*.fdb_latexmk"
  ],
  "latex-workshop.bibtex-format.tab": "4 spaces",
  "latex-workshop.latex.autoBuild.run": "never",
  "[latex]": {
    "editor.defaultFormatter": "James-Yu.latex-workshop"
  },
```

## Latex 格式化

在 VSCode 中，使用的是 latexindent 命令对 tex 文本进行格式化。latexindent 本质是个 perl 脚本，首次运行可能出现 Perl 模块缺失问题。

```
Can't locate Log/Log4perl.pm in @INC (you may need to install the Log::Log4perl module) (@INC contains: /usr/local/texlive/2018/texmf-dist/scripts/latexindent /Library/Perl/5.18/darwin-thread-multi-2level /Library/Perl/5.18 /Network/Library/Perl/5.18/darwin-thread-multi-2level /Network/Library/Perl/5.18 /Library/Perl/Updates/5.18.2 /System/Library/Perl/5.18/darwin-thread-multi-2level /System/Library/Perl/5.18 /System/Library/Perl/Extras/5.18/darwin-thread-multi-2level /System/Library/Perl/Extras/5.18 .) at /usr/local/texlive/2018/texmf-dist/scripts/latexindent/LatexIndent/LogFile.pm line 22. BEGIN failed--compilation aborted at /usr/local/texlive/2018/texmf-dist/scripts/latexindent/LatexIndent/LogFile.pm line 22. Compilation failed in require at /usr/local/texlive/2018/texmf-dist/scripts/latexindent/LatexIndent/Document.pm line 25. BEGIN failed--compilation aborted at /usr/local/texlive/2018/texmf-dist/scripts/latexindent/LatexIndent/Document.pm line 25. Compilation failed in require at /Library/TeX/texbin/latexindent line 27. BEGIN failed--compilation aborted at /Library/TeX/texbin/latexindent line 27.
```

此时需要使用 cpan 安装缺失模块。在命令行中运行 `sudo cpan install Log::Log4perl`。必须使用 sudo 的原因是部分模块需要安装在系统目录下，如 `/Library/Perl/Updates` 等。将全部模块安装完毕后即可对 latex 文件格式化。
