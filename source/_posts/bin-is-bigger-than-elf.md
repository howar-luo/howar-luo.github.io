---
title: bin文件为何有时比elf文件大
date: 2020-02-13 08:04:40
categories:
tags:
---

## 背景
最近在研究一款多核芯片，其有MPU核（跑Linux/QNX）、MCU核（跑RTOS），还有DSP核以及跑深度学习的核。

其中MCU核启动主要有两种方式：
* MMCSD</br>
这种启动方式要求将MCU的elf格式的二进制文件放入SD卡中，然后通过SBL或者Uboot(SPL)来对elf文件进行解析，最后加载到SRAM进行执行。因为MCU并没有虚拟内存的支持，因此，这种方式实际上是SRAM的启动方式（需要将MCU的***代码段***链接到SRAM对应的地址，而其他段可以链接到DDR对应的地址）
* OSPI</br>
这种启动方式要求将MCU的bin格式的二进制文件通过串口烧入板载flash中，然后直接XIP(execute-in-place)，不需要将整个文件加载到SRAM中执行，这意味着需要将MCU的***代码段***链接到flash对应的地址。对于这款强大的片子而言，给MCU用的SRAM也就1M，通过OSPI的启动方式，可以突破代码段1M的限制。

上面是基本情况的介绍，为了引入本文的主题，还需要介绍一个背景：本项目是将对实时性要求较高的控制算法移植到MCU核上，而原算法使用C++编写，因此最后是一个C/C++/ASM共存的工程。

## bin比elf大
工程编译完成之后，代码段接近1M，起初采用上述***MMCSD***启动方式，这种方式下，程序的存储、启动都不会有什么问题，但是由于SRAM 1M的限制，不得不将数据段/BSS段/栈等段链接到DDR对应的地址，这就导致程序运行速度不及预期。

因此采用***OSPI***的方式启动，将代码段等只读段链接到flash对应的地址，其他可读可写段链接到SRAM对应的地址。由于***OSPI***启动方式需要生成bin格式的二进制，然后烧进flash，因此需要先生成elf文件，然后通过**objcopy**工具将elf文件转换成bin文件。

elf格式与bin格式区别如下：
> A Bin file is a pure binary file with no memory fix-ups or relocations, more than likely it has explicit instructions to be loaded at a specific memory address. Whereas....
>
> ELF files are Executable Linkable Format which consists of a symbol look-ups and relocatable table, that is, it can be loaded at any memory address by the kernel and automatically, all symbols used, are adjusted to the offset from that memory address where it was loaded into. Usually ELF files have a number of sections, such as 'data', 'text', 'bss', to name but a few...it is within those sections where the run-time can calculate where to adjust the symbol's memory references dynamically at run-time.

因此，理论上来讲，bin文件应该比elf文件更小才对，但在执行上述通过**objcopy**工具将elf文件转换成bin文件后，发现elf文件和bin文件的大小如下图：
<img src="/images/bin_bigger_than_elf/file-size.png" width="867" height="52" align=center>

可以看到，bin文件远远大于elf文件，为什么呢？从map文件入手，查看map文件中段的地址分布如下：
<img src="/images/bin_bigger_than_elf/map-file.png" width="720" height="562" align=center>

从中可以看到，莫名其妙的多出了一个`.init_array`段，结合上图中的`MEMORY CONFIGURATION`可以看出，链接器默认将其链接地址放到`SRAM`区域，但需要注意的是`.init_array`为只读权限，这是重点。至于为什么会出现一个`.init_array`段，上文提到该工程是一个C/C++/ASM共存的工程，这应该是编译器编译`cpp`代码的一个行为。

若修改`linker file`，指定`.init_array`到只读地址区域（flash），如下图所示，则恢复正常：
<img src="/images/bin_bigger_than_elf/linker-file.png" width="461" height="260" align=center>

从而引出了本文想说的重点：对于只读段，若被链接到不连续的区域，那么在生成`bin`时，**objcopy**工具会对这些段间的区域进行填充（默认使用0x00进行填充）。如在本例中，`.text`段、`.const`段和`.cinit`段是连续的，而这几个段与`.vecs`段以及`.init_array`之间都会被填充。正是`.init_array`段与其他段之间的填充，导致`bin`文件变得相当大。

***那为何`.data`段、`.stack`段等这些可读可写的段与只读段之间不会出现填充导致`bin`文件变得很大的问题呢？***

这是因为**objcopy**工具会对这些段进行特殊处理，即将他们单独拿出来，紧随着只读段放在一起，当程序烧写进flash上电启动后，再将这些段拷贝到`SRAM`对应的地址（链接器会记录要拷贝到哪些地址），具体过程如下汇编：
<img src="/images/bin_bigger_than_elf/startup-copy-data.png" width="678" height="772" align=center>

## 小结
将只读段散列在整个内存空间的不同地方时，会导致`bin`文件比`elf`文件大。