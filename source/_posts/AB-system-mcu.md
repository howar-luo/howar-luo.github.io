---
title: 基于MCU打造A/B双系统启动和升级
date: 2020-02-09 22:14:42
categories:
tags:
---

## 前言
上一篇文章提到，基于u-boot+Linux的平台，可利用其对MMU、虚拟内存、文件系统等的支持，通过修改u-boot相关源码可实现对A/B双系统的引导和升级。而对于单片机系统，通常无法支持上述功能。对于一些使用单片机，但对完备性要求较高的场景（比如汽车/物联网等），如果想实现双系统的引导和升级，需要另辟蹊径。本文尝试给出一种方案。

本方案在STM32f215RG硬件平台上实现。

## 总体方案介绍
### 系统分区
MCU不像Linux平台可以支持复杂文件系统，其地址为线性地址，从用户手册上可以查到其flash地址划分如下：

<img src="/images/AB-system-mcu/flash-organization.png" width="700" height="460" align=center>

可供应用程序使用的是Main memory对应的1M flash空间。MCU上电后，从0x08000000处取指令开始执行。为实现A/B系统引导和升级，可对flash作以下划分：

<table border="2" bordercolor="black" width="500" cellspacing="0" cellpadding="5">
   <tr>
      <td> bootloader_lite</td>
      <td> 0x08000000-0x08003FFF (sector 0) </td>
   </tr>
   <tr>
      <td> boot_flag </td>
      <td> 0x08004000-0x08007FFF (sector 1) </td>
   </tr>
   <tr>
      <td> bootloader </td>
      <td> 0x08010000-0x0801FFFF (sector 4) </td>
   </tr>
   <tr>
      <td> bank_a </td>
      <td> 0x08040000-0x0809FFFF (sector 6/7/8) </td>
   </tr>
   <tr>
      <td> bank_b </td>
      <td> 0x080A0000-0x080FFFFF (sector 9/10/11) </td>
   </tr>
</table>

其中：
* bootloader_lite分区存放第一级bootloader。第一级bootloader的作用是用来升级第二级bootloader，其逻辑应当尽量简单，因为其不可在线更新
* boot_flag分区存放程序分区引导相关标志：上次运行分区标志；升级状态。下文将详细介绍
* bootloader分区存放第二级bootloader。第二级bootloader的作用是用来选择并跳转执行相应分区的应用程序
* bank_a分区存放A分区应用程序。其中该分区最后四个字节存放该分区被刷写次数counter_a
* bank_b分区存放B分区应用程序。其中该分区最后四个字节存放该分区被刷写次数counter_b

上述分区方案并不唯一，只需遵循以下原则即可：
1. boatloader_lite必须放在sector 0（从0x08000000开始）
2. 每个分区不垮sector，即每个分区包含一个或多个完整的sector
3. 每个分区的大小根据实际的需要分配

### 启动流程
上文提到，MCU不支持MMU、虚拟内存、文件系统这些技术，因此也注定MCU的双系统启动流程与u-boot+Linux平台不同。

在介绍其启动流程流程之前，先介绍一下该MCU中断向量表前8个字节，MCU的启动与这8个字节息息相关：
```
前四个字节：存放栈地址，MCU启动后会取该四个字节到MSP寄存器
后四个字节：存放复位中断服务函数Reset_Handler的地址，MCU启动后会取该四个字节到PC寄存器，从而开始执行程序启动
```

启动逻辑如下图所示：

<img src="/images/AB-system-mcu/boot-process.png" width="410" height="550" align=center>

## 方案实现
### bootloader如何选择启动分区
上文启动流程中提到，bootloader会根据boot_flag区的相关标志，以及boot_a中的counter_a和boot_b中的counter_b来选择应该跳转的分区，这个过程至关重要。首先对该过程用到的各个标志作一些介绍：
1. last_bank: 存放在boot_flag分区的前四个字节，存放上次运行的分区首地址，即为0x0804000或者0x080A0000，其它值视为无效值；
2. update_mark: 存放在boot_flag分区last_bank后的四个字节，存放升级状态
```
0: 准备升级。在application中，如果已经接收到升级文件并且准备开始进行相关flash擦写升级操作，则先将update_mark标志置为0
1: 升级完成。在application中，如果flash擦写升级操作完成，则在复位重启前先将update_mark标志置为1
2: 升级完成后临时状态。当flash擦写升级操作完成，并将update_mark标志置1重启后进入bootlader，在bootloader中会将update_mark标志临时置为2
3: 新刷入的固件正确启动标志。当bootloader将update_mark标志置为2，正常启动新刷入的分区固件后，在application中会将此标志置为3
```
3. counter_a: 存放bank_a分区最后四个字节，存放刷写bank_a时系统总共被更新次数
4. counter_b: 存放bank_b分区最后四个字节，存放刷写bank_b时系统总共被更新次数

结合上述启动标志，下图展示bootloader选择启动分区流程：

<img src="/images/AB-system-mcu/choose-bank.png" width="695" height="540" align=center>

### 如何进行程序跳转
无论从bootloader_lite到bootloader，还是bootloader到application，都涉及到程序跳转。MCU程序跳转需要处理三个点：
* MSP寄存器；设置堆栈指针
```C
__asm void MSR_MSP(unsigned int addr)
{
	MSR MSP, r0
	BX r14
}
```
上文也有提到，传入的addr即栈指针的值，从对应分区固件的起始四字节取。
* PC寄存器；取`Reset_Handler`指针赋给PC寄存器后直接执行
```C
typedef void (*func)(void);

void load_app(unsigned int addr)
{
	func jump2app = (func)addr;
	jump2app();
}
```
同样上文也有提到，传入的addr即`Reset_Handler`函数指针值，从对应分区固件的byte4-byte7取。
* 中断向量表重定向
程序跳转之后，需要将中断向量表重定向到当前程序对应的地址。决定中断向量表地址的寄存器为`SCB->VTOR`，可以通过修改文件`system_stm32f2xx.c`中的宏`VECT_TAB_OFFSET`来使程序跳转后执行`SystemInit`函数来设置`SCB->VTOR`寄存器，从而实现中断向量表的重定向。
```C
对于bootloader_lite/booloader/bank_a/bank_b，这个宏分别设置为：
0x00
0x10000
0x40000
0xA0000
```

### 如何制作bank_a和bank_b对应的固件
由于MCU不支持虚拟内存，因此其物理地址必须和链接地址保持一致。这也就意味着bank_a和bank_b对应的固件有些许不同，只需要在同一工程中修改其ROM地址即可，即分别修改为0x08040000和0x080A0000。

### 升级流程
MCU双分区升级流程如下图所示：

<img src="/images/AB-system-mcu/update-process.png" width="413" height="547" align=center>





