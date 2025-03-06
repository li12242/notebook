---
layout: post
title: 使用 perf 工具启动 PMU 监控功能
date: 2025-1-21
categories: hpc
---

# 选择 PMU 性能事件

Perf 支持不同类型的事件，包括软件事件和硬件事件。 可以使用带有选项 `-e` 的各种 perf 命令来指定性能事件。

使用命令 `perf list` 列出事件列表。
```bash
$ perf list
List of pre-defined events (to be used in -e or -M):
 
branch-instructions OR branches                    [Hardware event]
...
 
alignment-faults                                   [Software event]
...
 
L1-dcache-load-misses                              [Hardware cache event]
L1-dcache-loads                                    [Hardware cache event]
...

br_immed_retired OR armv8_pmuv3_0/br_immed_retired/[Kernel PMU event]
br_mis_pred OR armv8_pmuv3_0/br_mis_pred/          [Kernel PMU event]
br_mis_pred OR armv8_pmuv3_1/br_mis_pred/          [Kernel PMU event]
...
```

如示例所示，每个 CPU 的 PMU 提供 Perf 定义的以下事件类型：

1. 硬件事件
2. 硬件缓存事件
3. 内核PMU事件

不过，并非 CPU 的所有 PMU 事件都以性能事件格式列出。 Perf 还支持使用原始硬件事件格式。对于 Armv8-A CPU，可以通过指定 CPU 的 PMU 驱动程序名称和 PMU 事件的事件编号来使用。 您可以通过以下步骤选择《CPU Technical Reference Manual》（TRM）中描述的任何 PMU 事件：

## 查看性能事件编号

以 Cortex-A53 为例。 如果要选择 Exception taken(IRQ) 和 Exception taken(FIQ)，TRM 中会列出这两个事件，但 perf 列表中没有。请参阅 Cortex-A53 TRM 获取它们的事件编号，如下表所示：

| Event Number | Event Mnemonic |
| :----------: | :------------: |
|     0x86     |    EXC_IRQ     |
|     0x87     |    EXC_FIQ     |

## 检查 CPU 对应 PMU 驱动

本博客中使用的 Juno r2 平台以 big.LITTLE 处理器为基础，引入了两个 PMU 驱动程序。它们分别以 armv8_pmuv3_0 和 armv8_pmuv3_1 命名。如果要选择 Cortex-A53 PMU 的事件，则必须查找 Cortex-A53 PMU 的正确 PMU 驱动程序名称。

```bash
$ cat /proc/cpuinfo 
processor : 0
BogoMIPS : 100.00
Features : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant : 0x0
CPU part : 0xd03
CPU revision : 0
...
processor : 4
BogoMIPS : 100.00
Features : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant : 0x0
CPU part : 0xd07
CPU revision : 0 
...
```

在上述日志中，CPU0 是部件号为 0xd03 的 Cortex-A53，CPU4 是部件号为 0xd07 的 Cortex-A72。 部件号在 MIDR_EL1 中定义。更多信息请参见 CPU TRM。

然后，获取 PMU 驱动程序支持的 CPU 信息，如下所示：

```bash
$ ls /sys/bus/event_source/devices
armv8_pmuv3_0  armv8_pmuv3_1  breakpoint  kprobe  software  tracepoint  uprobe
$ cat /sys/bus/event_source/devices/armv8_pmuv3_0/cpus
0-3
$ cat /sys/bus/event_source/devices/armv8_pmuv3_1/cpus
4-5
```

在上述日志中，以 armv8_pmuv3_0 命名的 PMU 驱动程序是 CPU0-3 PMU 的驱动程序，而 armv8_pmuv3_1 是 CPU4-5 PMU 的驱动程序。

最后，可以建立 PMU 驱动程序与相应 CPU 的关系，即

* armv8_pmuv3_0 for Cortex-A53
* armv8_pmuv3_1 for Cortex-A72

## 检查特定 PMU 的原始硬件事件编码格式

现在您有了要选择的事件的事件编号和相应 CPU 的 PMU 驱动程序。 接下来，您需要检查 PMU 事件的编码格式。 使用如下命令：

```bash
$ cat /sys/bus/event_source/devices/armv8_pmuv3_0/format/event
config:0-15
```

在上述日志中，config:0-15 表示 0-15 位字段（两个字节）用于向 Perf 传递事件编号。

## 在 Perf 中选择原始硬件事件

在 Perf 中选择原始硬件事件格式的 PMU 事件。 以下命令列出了三种可能的用法。

```bash
# Single event
$ perf <command> -e armv8_pmuv3_0/event=0x86/ 
# Multiple events, selected in comma-separated with no space
$ perf <command> -e armv8_pmuv3_0/event=0x86/,armv8_pmuv3_0/event=0x87/ 
# Multiple events, both symbolic format and raw hardware events format
$ perf <command> -e armv8_pmuv3_0/event=0x86/,armv8_pmuv3_0/event=0x87/,armv8_pmuv3_0/br_immed_retired/
```

# 收集 PMU 统计数据

知道要选择的 PMU 事件后，就可以使用 Perf 收集所选 PMU 事件的统计数据。

perf stat：使用该命令运行指定应用程序，并收集整个过程的 PMU 计数器统计数据。 该命令可帮助您了解应用程序的性能特征，并确定需要进一步调查的潜在领域。

## perf stat 使用

perf stat 的基本用法如下：
```bash
$ perf stat -e <event> -- <command to run the application>
```

使用 perf stat 从 PMU 收集统计数据时，应考虑以下因素：

* 每次要选择的 PMU 事件数；
* 每次要选择的 PMU 事件数；
* 指定要计数的用户空间和内核空间事件。

## 每次选择的 PMU 事件数量

在对应用程序进行初步性能分析时，通常会选择一系列 PMU 事件，以全面了解应用程序的行为和性能。

但是，对于 Armv8-A CPU，每个 CPU 的 PMU 可用计数器是有限的。 如果在 Perf 命令中选择的事件超过了可用计数器的数量，内核会使用时间多路复用让每个事件都有机会计数。 在运行结束时，Perf 会根据启用的总时间和运行时间来缩放计数值。 换句话说，当多路复用和缩放发生时，所选 PMU 事件的计数值为估计值。

A typical example on the Juno r2 platform is as follows:

```bash
$ taskset -c 0-3 perf stat -e armv8_pmuv3_0/inst_retired/,armv8_pmuv3_0/l1d_cache_refill/,armv8_pmuv3_0/l1d_cache/,armv8_pmuv3_0/l2d_cache_refill/,armv8_pmuv3_0/l2d_cache/,armv8_pmuv3_0/br_immed_retired/,armv8_pmuv3_0/br_mis_pred/,armv8_pmuv3_0/br_pred/ -- ls

Performance counter stats for 'ls':

         1,690,222      armv8_pmuv3_0/inst_retired/     (45.02%)
            23,170      armv8_pmuv3_0/l1d_cache_refill/
           726,296      armv8_pmuv3_0/l1d_cache/
            13,997      armv8_pmuv3_0/l2d_cache_refill/
           125,787      armv8_pmuv3_0/l2d_cache/
           351,946      armv8_pmuv3_0/br_immed_retired/
            28,695      armv8_pmuv3_0/br_mis_pred/      (54.98%)
     <not counted>      armv8_pmuv3_0/br_pred/          (0.00%)

       0.007883225 seconds time elapsed

       0.004223000 seconds user
       0.004223000 seconds sys
```

在本例中，选择了 Cortex-A53 PMU 的八个 PMU 事件。 然而，对于 Cortex-A53，只有六个事件计数器可用。 这就造成了 inst_retired、br_mis_pred 和 br_pred 事件之间的复用。

这三个事件后面列出的百分比数字表示启用时间占整个运行时间的百分比。事件 inst_retired 和 br_mis_pred 前面列出的计数值是估计值，根据百分比进行缩放。选择 ls 命令作为应用程序。 因为它的运行时间很短。事件 br_pred 没有机会启用计数值。 因此，它的计数值被标记为<未计入>。对于其他事件，其值都是真实的计数值。

因此，在确定每次选择的 PMU 事件数量时，建议您考虑应用程序的特性。 如果应用程序的执行时间太短或不均匀，则复用和缩放会带来误差。

# 选择 PMU 事件

为了通过 PMU 事件值提取更准确、更有意义的指标，建议您每次都选择相关或可比事件。

当每次选择的 PMU 事件数超过可用计数器时，可以选择将相关 PMU 事件归入一个组。这样，组内事件之间不会发生复用，而只会在组间发生复用。为此，可以在要选择为一组的事件之间使用修饰符 \{ 和 \}。 基本用法如下：

```bash
$ perf stat -e \{<event 1>,<event 2>,…,<event m>\},\{…\} -- <command to run the application>
```

注：每组包含的事件数不得超过每个 CPU PMU 支持的最大数量。 对于 Cortex-A53 和 Cortex-A72，即六个事件计数器和一个周期计数器。 对于其他 CPU，您可以查阅相应 TRM 中的 PMCFGR.N。

Juno r2 平台的示例如下：

```bash
$ taskset -c 0-3 perf stat -e \{armv8_pmuv3_0/inst_retired/,armv8_pmuv3_0/l1d_cache_refill/,armv8_pmuv3_0/l1d_cache/,armv8_pmuv3_0/l2d_cache_refill/,armv8_pmuv3_0/l2d_cache/\},\{armv8_pmuv3_0/br_immed_retired/,armv8_pmuv3_0/br_mis_pred/,armv8_pmuv3_0/br_pred/\} -- ls

Performance counter stats for 'ls':

         1,568,362      armv8_pmuv3_0/inst_retired/     (25.03%)
            18,195      armv8_pmuv3_0/l1d_cache_refill/ (25.03%)
           623,266      armv8_pmuv3_0/l1d_cache/        (25.03%)
            15,510      armv8_pmuv3_0/l2d_cache_refill/ (25.03%)
           146,055      armv8_pmuv3_0/l2d_cache/        (25.03%)
           372,014      armv8_pmuv3_0/br_immed_retired/ (74.97%)
            31,153      armv8_pmuv3_0/br_mis_pred/      (74.97%)
           424,800      armv8_pmuv3_0/br_pred/          (74.97%)

       0.007646814 seconds time elapsed
 
       0.000000000 seconds user
       0.008331000 seconds sys
```

上例将八个 PMU 事件分为两组：

* 一组与高速缓存有关
* 另一组与分支机构有关

PMU 事件后面列出的百分比表明，多路复用仅在组间进行。 这确保了组内 PMU 事件的计数值具有同源性和可比性。

## 指定用户态和内核态事件监控

Perf 分别支持用户空间和内核空间计数。 您可以通过添加修饰符 u（用户空间计数）和 k（内核空间计数）来实现这一功能，如下所示：

```bash
$ perf stat -e armv8_pmuv3_0/event=0x86/u,armv8_pmuv3_0/br_immed_retired/u -- <command to run the application> 
$ perf stat -e cpu-cycles:u -- <command to run the application>
```

注意： 要在 Linux 中为非 root 用户计算内核空间中的 PMU 事件，还必须使用以下设置：

```bash
$ echo -1 > /proc/sys/kernel/perf_event_paranoid
```

我们可以将 perf stat 的可能用途总结如下：

1. 将要选择的所有 PMU 事件分成若干组。
2. 根据您要配置的应用程序，您可以选择：
   * 多次运行，每次选择一组进行测量，重复这一过程，直到测量完所有事件。 这样就不会发生多路复用和缩放。
   * 运行一次，并添加组修改器。 这样，组间就不会发生多路复用和缩放。


