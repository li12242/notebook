---
layout: post
title: OpenMPI 进程绑定
date: 2022-7-6
categories: hpc
---

# 简介

在 OpenMPI 使用时，如何将进程绑定到 CPU 物理核心上对于运行性能至关重要。OpenMPI 提供了多种将进程映射到计算核心方式，如在 mpirun 运行参数中添加 `--map-by *` 和 `--bind-to *` 等参数。

# 进程绑定方式

使用 OpenMPI 映射过程中包括 mapping，ranking 和 binding 三个层次。

1. mapping 是为每个进程指定默认位置；
2. ranking 是为每个进程在MPI_COMM_WORLD中赋值；
3. binding 是将进程固定到特定核心上运行；

mapping 步骤使用的mapper为每个进程分配一个默认位置。默认按照slots和node的顺序将进程映射到各个计算节点,此外也可以通过 `--map-by *` 参数进行调整。例如 `--map-by ppr:1:socket:PE=8` 代表每1个进程映射到单个 Socket，并且使用 8 个核心计算。另外一种映射方法是使用 rankfile 提供更加细致的进程映射控制方法，将在后面详细介绍。

ranking 步骤侧重于 MPI_COMM_WORLD 中的进程的排序。OpenMPI 将此与 mapping 过程分开，以便在 MPI 进程的相对放置方面具有更大的灵活性。通过 `--rank-by *` 参数可以对进程排序编号进行修改。

binding 步骤将每个进程绑定到一组给定的处理器上，例如可以超额使用一些物理核心，避免其他应用争夺计算资源。绑定可以使用参数命令为 `--bind-to *`

# 绑定测试

使用下面代码对 OpenMPI 运行过程进行测试，在运行过程中会输出当前进程编号和所在节点信息等。同时在 OpenMPI 运行时，也需要添加 `--report-bindings` 运行参数，详细显示运行信息。

```c
#include <stdio.h>
#include <mpi.h>

int main(int argc, char **argv){
    int myrank, nprocs;
    char name[20];
    int name_len;
    MPI_Init(&argc, &argv);
    MPI_Comm_size(MPI_COMM_WORLD, &nprocs);
    MPI_Comm_rank(MPI_COMM_WORLD, &myrank);
    MPI_Get_processor_name(name, &name_len);
    printf("rank[%3d] of [%3d] in { %s } says hello.\n", myrank, nprocs, name);
    MPI_Finalize();

    return 0;
}
```

下面显示了使用 3 个节点，每个节点两进程时使用不同映射和绑定参数运行结果：
```sh
$ mpirun -n 6 -machinefile hostfile --map-by ppr:1:socket --rank-by node --report-bindings ./hello
[cu02:33104] MCW rank 0 bound to socket 0[core 0[hwt 0]]: [B/././././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu02:33104] MCW rank 3 bound to socket 1[core 32[hwt 0]]: [./././././././././././././././././././././././././././././././.][B/././././././././././././././././././././././././././././././.]
[cu04:29488] MCW rank 2 bound to socket 0[core 0[hwt 0]]: [B/././././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu04:29488] MCW rank 5 bound to socket 1[core 32[hwt 0]]: [./././././././././././././././././././././././././././././././.][B/././././././././././././././././././././././././././././././.]
[cu03:43945] MCW rank 1 bound to socket 0[core 0[hwt 0]]: [B/././././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu03:43945] MCW rank 4 bound to socket 1[core 32[hwt 0]]: [./././././././././././././././././././././././././././././././.][B/././././././././././././././././././././././././././././././.]
rank[  5] of [  6] in {cu04} says hello.
rank[  3] of [  6] in {cu02} says hello.
rank[  4] of [  6] in {cu03} says hello.
rank[  2] of [  6] in {cu04} says hello.
rank[  0] of [  6] in {cu02} says hello.
rank[  1] of [  6] in {cu03} says hello.


$ mpirun -n 6 -machinefile hostfile --map-by ppr:1:socket --rank-by core --report-bindings ./hello
[cu02:33185] MCW rank 0 bound to socket 0[core 0[hwt 0]]: [B/././././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu02:33185] MCW rank 1 bound to socket 1[core 32[hwt 0]]: [./././././././././././././././././././././././././././././././.][B/././././././././././././././././././././././././././././././.]
[cu04:29569] MCW rank 4 bound to socket 0[core 0[hwt 0]]: [B/././././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu04:29569] MCW rank 5 bound to socket 1[core 32[hwt 0]]: [./././././././././././././././././././././././././././././././.][B/././././././././././././././././././././././././././././././.]
[cu03:44074] MCW rank 2 bound to socket 0[core 0[hwt 0]]: [B/././././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu03:44074] MCW rank 3 bound to socket 1[core 32[hwt 0]]: [./././././././././././././././././././././././././././././././.][B/././././././././././././././././././././././././././././././.]
rank[  1] of [  6] in {cu02} says hello.
rank[  5] of [  6] in {cu04} says hello.
rank[  3] of [  6] in {cu03} says hello.
rank[  0] of [  6] in {cu02} says hello.
rank[  4] of [  6] in {cu04} says hello.
rank[  2] of [  6] in {cu03} says hello.
```

# rankfile

出了上面使用的绑定参数外，通过 rankfile 可以更灵活的调整各个进程在物理核心位置。rankfile文件格式为
```
rank <N>=<hostname> slot=<slot list>
```
其中 N 为进程编号，hostname为节点名，slot list为对应逻辑核心编号。

同样使用上面程序作为示例，当设置 rankfile 如下时，每个进程将按照rankfile内定义的规则将各个rank绑定到对应节点的核心上来。
```sh
(00:54:40) ○ [lilongx@mu] /mnt/lustre/home/lilongx/backup  cat rankfile 
rank 0=cu02 slot=0
rank 1=cu04 slot=1
rank 2=cu03 slot=0
rank 3=cu02 slot=1
rank 4=cu03 slot=1
rank 5=cu04 slot=1
```

```bash
[cu02:34070] MCW rank 0 bound to socket 0[core 0[hwt 0]]: [B/././././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu02:34070] MCW rank 3 bound to socket 0[core 1[hwt 0]]: [./B/./././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu03:46230] MCW rank 2 bound to socket 0[core 0[hwt 0]]: [B/././././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu03:46230] MCW rank 4 bound to socket 0[core 1[hwt 0]]: [./B/./././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu04:30470] MCW rank 1 bound to socket 0[core 1[hwt 0]]: [./B/./././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
[cu04:30470] MCW rank 5 bound to socket 0[core 1[hwt 0]]: [./B/./././././././././././././././././././././././././././././.][./././././././././././././././././././././././././././././././.]
rank[  1] of [  6] in {cu04} says hello.
rank[  2] of [  6] in {cu03} says hello.
rank[  5] of [  6] in {cu04} says hello.
rank[  4] of [  6] in {cu03} says hello.
rank[  3] of [  6] in {cu02} says hello.
rank[  0] of [  6] in {cu02} says hello.
```