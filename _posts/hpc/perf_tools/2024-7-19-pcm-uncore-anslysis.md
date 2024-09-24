---
layout: post
title: PCM Uncore 监控分析
date: 2024-7-19
categories: hpc
---

# Self discovery Uncore PMON

自 4th Gen Intel Xeon CPU（Intel SPR）开始，软件可以通过读取一个MMIO页面，发现全局中所有PMON监控模块，然后找到PMON中所有寄存器。

# 工作流程

为找到MMIO页面，SW可以在所有PCI设备header查询具有PMON发现功能的DVSEC子结构。找到后即可读取MMIO页面BAR地址和页面大小，确定发现信息。

## MMIO页面

为寻找MMIO页面，需要对所有PCI设备进行轮询，并按照如下过程对PCI设备进行判断：

- h.read32(0, &value)，获取vendor_id与PCM_INTEL_PCI_VENDOR_ID（0x8086）比较；
- h.read32(6, &status)，获取capability list判断是否为0x10；
- h.read64(offset, &header.raw_value64[0]) 与 h.read64(offset + sizeof(uint64), &header.raw_value64[1])，判断 header.fields.cap_id == 0x23 和 header..fields.entryID == 1；
- h.read32(0x10 + header.fields.tBIR * 4, &bar) 获取 bar 地址为 bar &= ~0xFFF；

其中h为打开PCI设备文件，header为VSEC结构体，其结构体内容如下：

```C
union VSEC {
    struct {
        uint64 cap_id:16;
        uint64 cap_version:4;
        uint64 cap_next:12;
        uint64 vsec_id:16;
        uint64 vsec_version:4;
        uint64 vsec_length:12;
        uint64 entryID:16;
        uint64 NumEntries:8;
        uint64 EntrySize:8;
        uint64 tBIR:3;
        uint64 Address:29;
    } fields;
    uint64 raw_value64[2];
    uint32 raw_value32[4];
};
```

按照如上流程确认PCI设备后，最终读取bar变量即为MMIO页面地址。

## Discovery 寄存器内容读取

定义 GlobalPMU 和 BoxPMU 内容如下，两个结构体总长度为 $3 \times 64$ 位。

```C
    struct GlobalPMU
    {
        uint64 type:8;
        uint64 stride:8;
        uint64 maxUnits:10;
        uint64 __reserved1:36;
        uint64 accessType:2;
        uint64 globalCtrlAddr;
        uint64 statusOffset:8;
        uint64 numStatus:16;
        uint64 __reserved2:40;
    };
    struct BoxPMU
    {
        uint64 numRegs : 8;
        uint64 ctrlOffset : 8;
        uint64 bitWidth : 8;
        uint64 ctrOffset : 8;
        uint64 statusOffset : 8;
        uint64 __reserved1 : 22;
        uint64 accessType : 2;
        uint64 boxCtrlAddr;
        uint64 boxType : 16;
        uint64 boxID : 16;
        uint64 __reserved2 : 32;
    };
```

读取时可以通过 linux 内 mmap 命令将对应内存映射到设备空间进行访问。通过 PCI 设备内容获取 BAR 地址为 GlobalPMU 寄存器基地址。读取 GlobalPMU 寄存器全部数据后，maxUnits 表示包含最大 BoxPMU 数量，而 stride 为每个 BoxPMU 基地址偏移大小。

## PMON 寄存器访问

完成读取所有 BoxPMU 后，即可确认 PMON 对应寄存地址。以 SPR 平台为例，各 PMON 对应 BoxPMU 顺序如下：

```C
constexpr auto SPR_PCU_BOX_TYPE = 4U;
constexpr auto SPR_IMC_BOX_TYPE = 6U;
constexpr auto SPR_UPILL_BOX_TYPE = 8U;
constexpr auto SPR_MDF_BOX_TYPE = 11U;
constexpr auto SPR_CXLCM_BOX_TYPE = 12U;
constexpr auto SPR_CXLDP_BOX_TYPE = 13U;
```

### IMC PMON

IMC 寄存器为 MMIO 类型。以 SPR 平台为例，查看 BoxPMU 对应内容如下：

```C
PMU type 6 (8 boxes)
    unit PMU  of type 6 ID 0 box ctrl: 0xc8aa2800 (-) with access type MMIO width 48 numRegs 4 ctrlOffset 64 ctrOffset 8
    unit PMU  of type 6 ID 1 box ctrl: 0xc8aaa800 (-) with access type MMIO width 48 numRegs 4 ctrlOffset 64 ctrOffset 8
    unit PMU  of type 6 ID 2 box ctrl: 0xc8b22800 (-) with access type MMIO width 48 numRegs 4 ctrlOffset 64 ctrOffset 8
    unit PMU  of type 6 ID 3 box ctrl: 0xc8b2a800 (-) with access type MMIO width 48 numRegs 4 ctrlOffset 64 ctrOffset 8
    unit PMU  of type 6 ID 4 box ctrl: 0xc8ba2800 (-) with access type MMIO width 48 numRegs 4 ctrlOffset 64 ctrOffset 8
    unit PMU  of type 6 ID 5 box ctrl: 0xc8baa800 (-) with access type MMIO width 48 numRegs 4 ctrlOffset 64 ctrOffset 8
    unit PMU  of type 6 ID 6 box ctrl: 0xc8c22800 (-) with access type MMIO width 48 numRegs 4 ctrlOffset 64 ctrOffset 8
    unit PMU  of type 6 ID 7 box ctrl: 0xc8c2a800 (-) with access type MMIO width 48 numRegs 4 ctrlOffset 64 ctrOffset 8
```

可以看出，单个 socket 内共包含 8 个 IMC PMON 监控模块，每个 box 负责单条内存通道性能监控。对于第一个 PMU，其 box ctrl 寄存器地址为 0xc8aa2800（boxCtrlAddr），随后根据对应 box ctrl 地址，即可计算出其他寄存器偏移地址。
在 pcm 中定义的 IMC PMON 寄存器使用 MMIORegister32 和 MMIORegister64 类型，其对硬件寄存器读写通过 mmap 将设备内存映射到用户内存实现。

