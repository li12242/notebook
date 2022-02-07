---
layout: post
title: Linux内核参数优化
date: 2022-2-7
categories: linux, hpc
---

# 优化

AMD 第二代 Zen 平台可以使用 Linux 内核参数进行优化，根据 AMD 手册[amd-epyc-7003-sb-hpc-openfoam-cfd(1).pdf]()，采用以下命令对内核进行优化：

```bash
# Turn off swap
# Turn off swap to prevent accidental swapping. Do not that disabling swap without sufficient memory can have undesired effects
swapoff -a
# Turn off NUMA balancing
# NUMA balancing can have undesired effects and since it is possible to bind the ranks and memory in HPC, this setting is not needed
# NUMA balancing 0
echo 0 >/proc/sys/kernel/numa_balancing
# Disable ASLR (Address Space Layout Ranomization) is a security feature used to prevent the exploitation of memory vulnerabilities
# randomize_va_space
echo 0 >/proc/sys/kernel/randomize_va_space
# Set CPU governor to performance and disable cc6. Setting the CPU perfomance to governor to perfomrnaces ensures max performances at all times. Disabling cc6 ensures that deeper CPU sleep states are not entered.
cpupower frequency-set -g performance
cpupower idle-set -d 2
```

经过测试，上述参数对于 Intel 平台一样适用，在大规模并行环境下可以大幅提升 HPC 应用性能。

# 参考

1. [AMD 2nd Gen EPYC CPU Tuning Guide for InfiniBand HPC](https://hpcadvisorycouncil.atlassian.net/wiki/spaces/HPCWORKS/pages/1280442391/AMD+2nd+Gen+EPYC+CPU+Tuning+Guide+for+InfiniBand+HPC)