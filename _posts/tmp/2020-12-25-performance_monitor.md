---
layout: post
title: Intel CPU 性能监控
date: 2020-12-25
categories: tools
---
# 简介

在 Intel Pentium 处理器开始，通过引入性能监控计数器 MSRs（Model-Specific Register）实现了性能监控功能。
性能监控分为两类，第一类使用计数器对事件进行监控并且取样，这些事件不属于架构系统并且会随着处理器架构变化。第二类为架构级别的性能监控能力，其支持一小部分的计数和取样功能，而支持的架构性能事件在不同 CPU 架构中是相同的。

# Uncore Performance Monitor Units

PMU（performance monitoring units）使用分布式设计，其中计数器在不同 uncore 单元中，不同单元中计数器无法监控其他单元的事件。
不同 uncore 单元如图所示，其中 CBo（C-box）、ARB（arbitration）和 IMC（integrated memory controller）都是 uncore 单元中一部分。

![6th Generation Intel^reg Core Processor Uncore Block Diagram](http://ww1.sinaimg.cn/large/7a1c18a8ly1get6yufscrj20ka09r0tb.jpg)

CBox 是负责 LLC 上多个切片一致性的引擎，每个 CBo 提供了 MSRs 来选取 uncore 性能监控事件。MSRs 选取的每个事件与一个计数寄存器相联系。ARB 提供了局部性能计数器和事件，在 ARB 中包含了固定或不可编程的计数器。IMC 包含 5 个模块，其中固定计数器运行监控一些列 DRAM 请求。

## Uncore PMU MSR

在 PMU 中，包含了许多 MSR，并且在Intel 官方手册中，提供了对应寄存器地址[^12121]。
每个寄存器都有不同名字和功能，其不同功能实现主要通过修改对应寄存器中 bit 位上的值来实现，因此寄存器的每个位都有对应的 field 名。

[^12121]: 6th Generation Intel® Core™ Processor Family Uncore Performance Monitoring Reference Manual n.d.:20.

## Uncore PMU Events

在 CBo 和 ARB 单元中，不同监控内容有相应的监控事件（Events）。在 Intel 官方手册中对每个事件介绍了对应的事件名，ID，掩码（umask）以及描述。
其中代码写入相应寄存器的 EVT_SEL 位，而掩码则写入寄存器 UMASK 位中。

在 IMC 中，

### UBox

UBox 负责 Skylake 平台中系统配置管理者的角色，作为中央控制单元负责以下内容：

1. 通过使用 Message Channel 管理读取和写入物理分配的寄存器；
2. 中断消息的中间人，从系统接收中断信息随后发送到对应核心上；
3. 系统锁管理角色

# pcm 代码解析

## cpucounters

### PCM 对象

在 `cpucounters.cpp` 文件中，最重要的对象就是 PCM 对象。
PCM 为 CPU 性能监控对象，在每次进行性能监控时都必须首先对此对象进行初始化，生成唯一的对象后捕捉 CPU 性能参数。

PCM 初始化时需要采用如下函数

* `detectModel` 测试是否采用虚拟化模式运行，提示相应警告，通过 cpuid 提出 core_gen_counter_num_max 相应信息；
* `checkModel` 判断当前平台是否支持；
* `initCStateSupportTables` 获取与 CPU 功耗相关的 C-States 表格并进行初始化；
* `discoverSystemTopology` 获取 cpu 相关信息；
* `initMSR` 初始化 MSR 指针；
* `readCoreCounterConfig` 通过 cpuid 并利用汇编语句读取 cpu 性能；
* `printSystemTopology` 
* `detectNominalFrequency` 获取 CPU 基准频率；
* `showSpecControlMSRs` 检查内核中是否安装 IBRS 和 STIBP 补丁；
* `initEnergyMonitoring` 初始化功耗监控
* `initUncoreObjects` 
* `initRMID`
* `readCPUMicrocodeLevel`
