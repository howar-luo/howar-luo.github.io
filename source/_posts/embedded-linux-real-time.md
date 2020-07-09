---
title: Linux内核针对实时性的调优
date: 2020-03-15 21:22:37
categories:
tags:
---
## 什么是操作系统的实时性
首先看看维基百科对实时操作系统的定义：
> **实时操作系统（Real-time operating system, RTOS）**，又称即时操作系统，它会按照排序运行、管理系统资源，并为开发应用程序提供一致的基础。实时操作系统与一般的操作系统相比，最大的特色就是“实时性”，如果有一个任务需要执行，实时操作系统会马上（在较短时间内）执行该任务，不会有较长的延时。这种特性保证了各个任务的及时执行。

**“实时性”** 是实时操作系统最大的特性。

而维基百科对“实时性”的定义如下：
> 实时运算（Real-time computing）是计算机科学中对受到“实时约束”的计算机硬件和计算机软件系统的研究，实时约束像是从事件发生到系统回应之间的最长时间限制。实时程序必须保证在严格的时间限制内响应。

对于实时性理解的重点是，实时并不是意味着速度快，其关键在于保证完成时间，而不是原始速度。

而针对用户对超出时间限制所造成的影响的可接受程度，实时又可分为软实时和硬实时：
* 软实时</br>
软实时的特点是如果超过了时间限制后操作还没有完成的话，体验的质量就会下降，但不会带来致命后果。结果可能不尽如人意，并导致体验的质量有所下降，但这并不是灾难性的。

* 硬实时</br>
硬实时的特点是错过时限会造成严重结果。在一个硬实时系统中，如果错过了时限，后果往往是灾难性的。当然，“灾难”是相对而言的。 

## Linux为什么不是硬实时操作系统
Linux系统一开始就被按照GPOS(通用操作系统)来设计的，它所追求的是尽量缩短系统的平均相应时间，提高吞吐量，达到更好的平均性能。在这个背景下，Linux无法达到强实时性的因素是多方面的，比如虚拟内存管理、共享资源互斥访问机制等等，但最重要的因素是进程调度以及内核抢占机制，这也是本文讨论的重点。

### Linux在哪些时机不可调度
想要搞清这个问题，首先需要介绍一下linux中的四类区间：
1. 中断
2. 软中断
3. 进程上下文中的spin_lock
4. 进程上下文中的其他区域

上述四类区间中，只有第四类区间支持抢占调度。当可以调度的事情发生在前3类区间中，即如果在这3类区间中唤醒了高优先级的可以抢占的task，实际上却不能抢占，直到这3类区间结束。

用一个例子说明如下：
<img src="/images/embedded-linux-real-time/preemptive.png" width="586" height="427" align=center>

如上图所示：
* T0时刻`Normal task`因为`syscall`陷入内核
* T1时刻`CPU`拿到`spin lock`，进入`Critical section`
* T2时刻系统产生中断`IRQ1`，进入`IRQ handler`
* T3时刻系统唤醒了高优先级的`RT task`，但由于此时系统处于不可调度区域，所以`RT task`无法立即运行
* T4时刻`IRQ1`结束，但接着产生中断`IRQ2`，进入`IRQ handler`
* T5时刻，中断都结束，但`spin lock`仍然没有释放，系统仍然处于不可调度区域
* T6时刻，`spin lock`释放，高优先级的`RT task`立马得到调度
* T7时刻`RT task`运行结束，`Normal task`再一次被调度到
* T8时刻从内核态返回

从T1到T6，这个区间的时间是不可预测的，因此通用的Linux系统无法达到硬实时的标准。

## 如何改进Linux内核的实时性
当前主要有两种方式对Linux内核的实时性进行优化：
1. 修改内核代码</br>
通过打布丁的方式，对内核的进程调度、中断服务程序等代码进行修改与优化，提高系统的实时性能，并且保证了系统的通用性。当前社区的维护的，最出名的就是`PREEMPT_RT`布丁。
2. 双内核法</br>
通过在Linux内核与硬件中断之间增加一个可抢先的实时内核，把标准的Linux内核作为该实时内核的一个优先级最低的进程来调度，它可以被实时进程抢断，正常的Linux进程仍可以在Linux内核上运行，这样既可以使用标准分时操作系统即Linux的各种服务，又能提供低延时的实时环境。RT-Linux是采用双内核法改造Linux实时性的典型代表。

本文主要讨论第一种方式的原理及效果。

## PREEMPT_RT补丁
### PREEMPT_RT补丁原理
`PREEMPT_RT`补丁可以通过以下方面对kernel进行源码级的改造：
* spinlock迁移为可调度的mutex
* 中断线程化
* 软中断线程化

从而将Linux内核中的1/2/3类区间都改造成4类区间，大大提高了系统的实时性。

### 基于Raspberry Pi 3B的PREEMPT_RT补丁效果测试
#### 获取获取实时补丁的`kernel`源码
现在直接可以在`github`上获取带有实时补丁的`kernel`源码，使用以下`git`指令(假设都在`~/raspberrypi-kernel$`目录下进行)：
```shell
~/raspberrypi-kernel$ git clone https://github.com/raspberrypi/linux.git -b rpi-4.14.y-rt
```
编译过程在ubuntu 14.04上，因此需要进行交叉编译，获取相应的交叉编译工具链如下：
```shell
~/raspberrypi-kernel$ git clone https://github.com/raspberrypi/tools.git
```

#### 工具链相关配置
可以执行以下命令，对交叉编译工具链进行相关配置(首先在`~/raspberrypi-kernel`目录下新建目录`rt-kernel`)：
```shell
~/raspberrypi-kernel$ export ARCH=arm
~/raspberrypi-kernel$ export CROSS_COMPILE=~/raspberrypi-kernel/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf-
~/raspberrypi-kernel$ export INSTALL_MOD_PATH=~/raspberrypi-kernel/rt-kernel
~/raspberrypi-kernel$ export INSTALL_DTBS_PATH=~/raspberrypi-kernel/rt-kernel
```

#### 内核编译配置
使用以下指令可以对内核编译进行相关配置：
```shell
~/raspberrypi-kernel$ export KERNEL=kernel7
~/raspberrypi-kernel$ cd ~/raspberrypi-kernel/linux/
~/raspberrypi-kernel/linux/$ make bcm2709_defconfig
```
执行`make bcm2709_defconfig`的时候可以看到，如下图所示，打了实时补丁的内核，在`Preemption Model`中，多了一个选项`Fully Preemptible Kernel (RT)`，需要选中这个选项。

<img src="/images/embedded-linux-real-time/menuconfig.png" width="713" height="415" align=center>

#### 内核编译
使用以下命令，编译内核/模块/dtb，并进行相应目录的安装：
```shell
~/raspberrypi-kernel/linux$ make -j4 zImage
~/raspberrypi-kernel/linux$ make -j4 modules
~/raspberrypi-kernel/linux$ make -j4 dtbs
~/raspberrypi-kernel/linux$ make -j4 modules_install
~/raspberrypi-kernel/linux$ make -j4 dtbs_install
```
> 其中，`-j4`是想利用编译机的多个核心，进行多线程的编译，这样编译起来比较快

编译完成后，需要进行相应格式内核镜像生成命令：
```shell
~/raspberrypi-kernel/linux$ mkdir $INSTALL_MOD_PATH/boot
~/raspberrypi-kernel/linux$ ./scripts/mkknlimg ./arch/arm/boot/zImage $INSTALL_MOD_PATH/boot/$KERNEL.img
```

#### 新内核安装
首先，将新内核相关文件传输到树莓派上：
```shell
~/raspberrypi-kernel/linux$ cd $INSTALL_MOD_PATH
~/raspberrypi-kernel/rt-kernel$ tar czf ../rt-kernel.tgz *
~/raspberrypi-kernel/rt-kernel$ cd ..
~/raspberrypi-kernel$ scp rt-kernel.tgz pi@<ipaddress>:/tmp
```
然后进行`Kernel Image, Modules & Device Tree Overlays`的安装，在树莓派上执行以下命令：
> 注意，在安装之前，要测试以下原内核的实时性能，测试方法下文中有介绍
```shell
~$ cd /tmp
/tmp$ tar xzf rt-kernel.tgz
/tmp$ cd boot
/tmp/boot$ sudo cp -rd * /boot/
/tmp/boot$ cd ../lib
/tmp/lib$ sudo cp -dr * /lib/
/tmp/lib$ cd ../overlays
/tmp/overlays$ sudo cp -d * /boot/overlays
/tmp/overlays$ cd ..
/tmp$ sudo cp -d bcm* /boot/
```
并且需要修改`/boot/config.txt`中的信息，添加被引导的kernel版本：
```shell
kernel=vmlinuz-4.14.91-rt49-v7+
```
这个版本号不能弄错，可以到`linux`源码根目录下的`Makefile`文件和`localversion-rt`文件分别查看`kernel`和`rt`对应的版本号。

#### 实时性能测试
可以通过工具`cyclictest`工具来测试Linux实时性能。该工具的使用方法已经安装方法这里不做过多介绍。
使用以下命令
```
sudo cyclictest -p 90 - m -c 0 -i 200 -n -h 100 -l 1000000
```
分别测试打实时补丁前后Linux实时性能如下图:
<img src="/images/embedded-linux-real-time/kernel-latency-test.png" width="460" height="123" align=center> <img src="/images/embedded-linux-real-time/kernel-latency-test-rt.png" width="460" height="123" align=center>

重点关注`Max Latencies`这一项，单位为微秒。由上图可知，kernel打上实时补丁后，实时性能有一定的提升。

