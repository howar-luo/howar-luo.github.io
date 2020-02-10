---
layout: pages
title: 单片机重启时可能发生的异常 ---- flash被异常改写
date: 2018-11-09 22:21:05
tags:
---
### 背景
STM32F215RG MCU在多次断电重启的场景中，概率性出现无法启动的现象。

### 问题定位
重新线刷MCU完整固件，现象消失。

通过jlink读出芯片整片flash二进制文件进行分析，发现出现无法启动的问题时，从0x08000000地址开始的48Bytes或者64Bytes被异常清0，由于该区域存放中断向量表，从而导致芯片启动时找不到对应的复位中断向量而无法正常启动。

#### 上下电电源波形异常验证
由于这种情况只出现在MCU断电重启的场景，因此怀疑跟上下电电压波形有关，因为STM32F215RG单片机内核工作电压范围为1.8v-3.6v，而电压低至1.8v附近时，flash及其他外设很有可能无法正常工作，如果该情况持续时间较长，可能会引起芯片内部逻辑信号紊乱，导致程序跑飞及flash异常改写的情况。

为验证上述猜想，分别拿容易复现问题的控制板和正常的控制板量取下电电压波形如下：
![](/images/mcu-power-on-erase-flash/fault-vol.png)
![](/images/mcu-power-on-erase-flash/normal-vol.png)
从上述波形可以看出，异常MCU下电时，电压降的较缓(从2.4v降到1.8v需要13ms左右)，从而可能出现CM3核在工作，flash及其他外设无法正常工作的场景，从而可能出现芯片内部逻辑信号异常的情况。

至此，猜测具体原因可能有两个：
1. 在电源异常的情况下，芯片可能有自保护策略，由自身硬件逻辑清除掉相应flash，从而导致芯片无法启动；
2. 在电源异常的情况下，由于芯片内部逻辑信号异常，导致程序跑飞，由程序中flash驱动擦出相应flash，从而导致芯片无法启动。

遗憾的是，第一种猜想在相关官方手册中没有找到明确的证据。针对第二种猜想，继续做以下实验。

#### Flash异常操作验证
由于固件中有操作flash的代码：擦(FLASH_Erase_Sector)和写(HAL_FLASH_Program)。
在遵守尽量用相同二进制文件的原则下，做以下试验。
通过反汇编，分别找到擦和写flash函数对应的地址和二进制内容，并将其清掉。
写(HAL_FLASH_Program)函数二进制范围为0x080433a8-0x0804341c，将其清除，如下图：
![](/images/mcu-power-on-erase-flash/disable-flash-write.png)
多次实验结果发现，MCU无法启动的问题依然存在。

擦(FLASH_Erase_Sector)函数二进制范围为0x08041780-0x080417d4，将其清除，如下图：
![](/images/mcu-power-on-erase-flash/disable-flash-erase.png)
多次实验结果发现，系统无法启动的问题消失。说明MCU在上下电的时候可能出现程序跑飞最终执行该函数。

进一步实验，来验证程序是直接跑飞到该函数，还是跑飞到调用到该函数的地方。
调用FLASH_Erase_Sector的函数为HAL_FLASHEx_Erase_IT，通过反汇编，找到FLASH_Erase_Sector被调用的机器码，将其清掉，如下图：
![](/images/mcu-power-on-erase-flash/disable-flash-erase-called.png)
多次实验结果发现，MCU无法启动的问题消失。说明有可能FLASH_Erase_Sector的执行是通过HAL_FLASHEx_Erase_IT的调用完成的。

通过上述相同的方法，继续向上查找，通过多次实验，发现最顶层的调用点在Internal_Erase_Flash_Bank调用HAL_FLASHEx_Erase_IT的地方，即清掉Internal_Erase_Flash_Bank调用HAL_FLASHEx_Erase_IT的机器码，问题消失。

从上述实验现象可以推测，MCU在上下电时，程序跑飞到HAL_FLASHEx_Erase_IT到FLASH_Erase_Sector的调用链上，导致flash异常，向量表被清除。

#### 解决措施
基于上述推测及实验，单片机在上下电过程中可能出现PC指针跑飞，为避免这种情况的发生，可以考虑提高单片机复位电压，即在上下电过程中，只让单片机内核及外设在较稳定的电压范围内工作，否则就让其处于复位状态。
STM32F215RG芯片支持BOR(brown-out reset)，且支持修改BOR reset Level，如下图所示：
![](/images/mcu-power-on-erase-flash/BOR-reset-level.png)
本项目中，将BOR reset Level设置为Level2，即当电压下降到2.4v以下，就将CM3核拉至复位状态。
开启该功能后，用同样的二进制文件，经过上千次的实验，问题不复现。

