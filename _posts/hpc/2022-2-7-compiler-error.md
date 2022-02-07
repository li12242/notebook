---
layout: post
title: 编译器常见错误
date: 2022-2-7
categories: linux, compiler
---

# 编译常见错误

使用Intel编译器时，可能出现如下错误：
```bash
Catastrophic error: could not set locale "" to allow processing of multibyte characters 
```
定义环境变量即可

```bash
export LANG=en_US.UTF-8
```
