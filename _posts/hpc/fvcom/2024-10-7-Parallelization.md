---
layout: post
title: FVCOM并行区域划分与MPI通信
date: 2024-10-7
categories: hpc,fvcom
---

# 简介

FVCOM 采用非结构化网格计算，并且支持 MPI 并行。为实现多进程并行计算，需要将网格单元平均划分给各个进程，即所谓的并行区域划分过程。非结构化网格的并行区域划分是个复杂的问题，一般都使用第三方函数库实现，如Metis、Scotch、Kahip等。FVCOM使用Metis对并行区域进行划分，下面将首先对并行区域划分过程进行介绍，随后介绍FVCOM内各个并行区域间如何实现MPI数据通信。

# 并行区域划分

并行区域划分过程发生在 `SETUP_DOMAIN` 函数内，由FVCOM主函数调用。并行区域划分由函数 `DOMDEC` 实现，对应调用过程如下，其中传入和返回参数 `EL_PID` 储存了每个单元所属进程编号。

```fortran
     !
     !  DECOMPOSE DOMAIN BY ELEMENTS USING METIS
     !
     ALLOCATE(EL_PID(NGL))  ;  EL_PID = 1
     CALL DOMDEC(NGL,NVG,NPROCS,EL_PID,MSR)
```

下图显示了并行区域划分结果，每个区域用单个颜色表示。查看 `EL_PID` 划分结果可以看出，整个计算域被分为了 `NPROCS=16` 份。通过 Metis 划分保证各个并行区域间相邻单元较少，并且各区域内单元数大致相同。关于 Metis 函数可以参考 Metis 使用手册，因此不再赘述。

# halo 单元与格点

在 FVCOM 中对每个单元或格点计算时，需要利用周围单元或格点信息，从而计算出控制单元边界面通量。在每个进程内，需要构造 halo 单元从而保证进程内部单元或格点计算时全部模板数据包含在本进程内。
由于 FVCOM 中流速和标量值分别位于三角形单元形心和顶点，对应所需 halo 单元类型也有所不同。在 `GENMAP` 函数内，分别构造了 `EC`，`NC` 和 `BNC` 三类 halo 结构体，对应网格元素分别对应三角形单元、外部顶点和内部顶点，下面将对构造过程及对应变量进行介绍。

## EC 单元

EC 结构体传递的是 halo 区域内三角形单元，构造采用如下步骤执行：

1. 根据 `EL_PID` 划分结果，可以找到本地单元和本地格点：在 FVCOM 内本地单元数为 `N`，对应三角形单元全局编号储存在 `EGID` 内。
2. 所有包含在本地三角形单元内的格点，也都作为本地格点储存，对应格点数为 `M`，对应格点全局编号储存在 `NGID` 内。
3. EC 内包含的单元是那些包含本地格点，但是不属于本地单元的单元集合，即 halo 单元。

以上图中进程1为对象，包含的本地单元和halo单元如下图所示，对应分别是橙色与绿色单元。根据上面构造过程也可以推导出，与halo单元相邻的内部单元，也是其他进程的halo单元，所以进程1需要将这些内部单元数据发送给其他进程（进程2、3等）。

> 由于halo单元是由包含进程内部格点定义的，两个相邻进程对应的halo单元数不一定相同。

## NC 单元

NC 结构体传递 halo 区域外部格点，其筛选方法采用如下步骤执行：

1. 对所有halo单元和格点循环，当格点属于 halo 单元内时，标记 `INELEM` 为真，`ISMYN` 为假；
2. 对所有内部格点循环，当属于内部格点时标记 `ISMYN` （ISMYNode）为真；
3. 将halo格点全局编号保存在 `NTEMP` 内，并赋值给 `HN_LST`；

仍然以计算域内进程1为例，对应halo格点用蓝色点标出。通过上面构造过程可以看出，halo格点的个数与halo单元个数没有关系，可能出现多个halo单元共用格点情况。从几何拓扑来说，NC 包含顶点数量会远小于 EC 结构体内单元数量。

## BNC 单元



# 本地进程变量与通信

## 本地进程变量长度

在确定每个进程内包含本地与halo单元/节点后，后续单元与格点对应的物理量长度将从 `NGL`/`MGL` 变为 `N`/`M` 或 `N+NHE`/`M+NHN`，从而降低内存占用量。为了通信或结构存储，还需要建立本地变量与全局变量之间映射关系。此映射通过两个数组实现，分别为 `ELID_X(0:NGL)` / `NLID_X(0:MGL)` 和 `EGID_X(0:N+NHE)` / `NGID_X(0:M+NHN)`，其对应元素即为全局/本地单元或格点对应的本地/全局编号。

根据 `EGID_X(0:N+NHE)` 和 `NGID_X(0:M+NHN)` 内全局格点编号，在每个进程内部按照 ”内部“ + ”halo“ 单元/格点的顺序排列，如下代码为 `NLID_X` 和 `NGID_X` 映射数组赋值过程。

```fortran
      ALLOCATE(NLID(0:MGL)) ; NLID = 0
      ALLOCATE(NLID_X(0:MGL)) ; NLID_X = 0
      
      DO I=1,M
         NLID(NGID(I)) = I
      END DO
      NLID_X = NLID
      
      DO I=1,NHN
         NLID_X(HN_LST(I)) = I+M
      END DO

      ......

      ALLOCATE(NGID_X(0:M+NHN)); NGID_X = 0
      DO I =1,MGL
         NGID_X(NLID_X(I)) = I
      END DO
      NGID_X(0) = 0
```

上述代码中 `NLID(M)` 为递增编号序列，其初始化过程按照 1 到 `MGL` 顺序对每个单元对应局部编号进行分配。

```
      ALLOCATE(NTEMP(MGL))
      ......
      
      M = 0
      DO I=1,MGL
         IF(NODEMARK(I) == 1)THEN
            M = M + 1
            NTEMP(M) = I
         END IF
      END DO
!   DEALLOCATE(NODEMARK)

      
      ALLOCATE(NGID(M))
      NGID(1:M) = NTEMP(1:M)
      DEALLOCATE(NTEMP)
```

在每个进程内格点的局部编号也按照类似的 ”内部“ + ”halo“ 顺序进行。

## 进程间通信

在确认进程内局部单元/格点排序后，需要建立进程间通信映射关系。在 FVCOM 内通过 `COMM` 类型变量储存进程内通信使用的单元/格点/边界格点编号。`COMM` 类型变量定义如下，在 `MOD_PAR` 模块内共定义三个变量，分别是 `EC`，`NC`，`BNC`，分别负责 halo 单元/格点/边界点通讯。

```fortran
  TYPE COMM
     !----------------------------------------------------------
     ! SND: TRUE IF YOU ARE TO SEND TO PROCESSOR               |
     ! RCV: TRUE IF YOU ARE TO RECEIVE FROM PROCESSOR          |
     ! NSND: NUMBER OF DATA TO SEND TO PROCESSOR               |
     ! NRCV: NUMBER OF DATA TO RECEIVE FROM PROCESSOR          |
     ! SNDP: ARRAY POINTING TO LOCATIONS TO SEND TO PROCESSOR  | 
     ! RCVP: ARRAY POINTING TO LOCATIONS RECEIVED FROM PROCESS |
     ! RCPT: POINTER TO LOCATION IN RECEIVE BUFFER             |
     !----------------------------------------------------------

     !  LOGICAL :: SND,RCV
     INTEGER  NSND,NRCV,RCPT
     INTEGER, POINTER,  DIMENSION(:) :: SNDP => null(),RCVP => null()
     REAL(SP), POINTER,   DIMENSION(:) :: MLTP => null()
  END TYPE COMM
```


## 相关变量总结

|     变量名      |                        说明                        |
| :-------------: | :------------------------------------------------: |
|       NGL       |                     全局单元数                     |
|        N        |                     本地单元数                     |
|    EGID(1:N)    |                本地单元对应全局编号                |
|       MGL       |                     全局格点数                     |
|        M        |                     本地格点数                     |
|    NGID(1:M)    |                本地格点对应全局编号                |
|       NHE       |                   halo 单元个数                    |
| HE_LIST(1:NHE)  |               halo 单元对应全局编号                |
|  HE_OWN(1:NHE)  |                 halo 单元所属进程                  |
|       NHN       |                   halo 格点个数                    |
| HN_LIST(1:NHN)  |               halo 单元对应全局编号                |
|  HN_OWN(1:NHN)  |               halo 单元对应全局编号                |
|  ELID_X(0:NGL)  | 全局单元对应本地+halo单元编号（不存在单元编号为0） |
|  NLID_X(0:MGL)  | 全局格点对应本地+halo格点编号（不存在格点编号为0） |
| NGID_X(0:M+NHN) |           本地+halo单元对应全局单元编号            |
| EGID_X(0:N+NHE) |           本地+halo格点对应全局格点编号            |
|                 |

