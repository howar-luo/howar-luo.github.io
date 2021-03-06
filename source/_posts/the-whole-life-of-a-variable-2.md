---
title: 一个变量的一生(下)
date: 2019-01-22 23:24:47
categories:
tags:
---
### 前言
本文承接上一篇博客，尝试分析变量在内存中被访问的机制。

### 地址空间
每一个进程都有自己的地址空间，其中包含运行的程序的所有内存状态。

操作系统将属于不同进程的地址空间进行隔离，从而防止互相伤害。通过内存隔离，操作系统页进一步去保运行程序不会影响底层操作系统的操作。

上述提到的地址空间为虚拟地址空间，CPU在访问类似上一篇文章中提到的variable变量都采用的是虚拟地址。实际上，作为用户级程序的程序员，可以看到的任何地址都是虚拟地址。只有操作系统，通过精妙的虚拟化内存技术，知道这些指令和数据所在的物理内存的位置，这也是本文尝试分析的点所在。

### 地址转换
当CPU发出对地址的访问时，终究需将虚拟地址转换成实际存储的物理地址，而这个完整的过程同时需要硬件和操作系统的参与。
1. 硬件(MMU)对每次内存访问进行处理（即指令获取、数据读取或写入），将指令中的虚拟地址转换成数据实际存储的物理地址。因此，在每次内存引用时，硬件都会进行地址转换，将应用程序的内存引用重定位到内存中实际的位置。
2. 仅仅依靠硬件不足以实现虚拟内存，因为它只是提供了底层机制来提高效率。操作系统必须在关键的位置介入，设置好硬件，以便完成正确的地址转换。因此它必须管理内存，记录被占用和空闲位置，并明智而谨慎地介入，保持对内存的控制。

### 分段与分页
操作系统在将进程的地址空间往物理内存中加载（即虚拟地址与物理地址的映射）方式有三种：
1. 完整的加载</br>
即将进程一次全部加载到内存中。这种加载方式会造成物理内存的浪费，比如通常栈和堆之间的空间并没有被进程使用，却依然占用了实际的物理内存。另外，如果剩余物理内存无法提供连续区域来放置完整的地址空间，那么进程便无法运行。

2. 分段加载</br>
将逻辑段（如代码段、堆、栈等）作为一个主体，在MMU中为其引入一对基址和界限寄存器对。这种加载方式可以解决上述加载方式所带来的问题，但也同时引入的新的问题，其中一个很重要的问题就是“外部碎片”。每个进程都有一些段，每个段的大小也可能不同，因此会出现物理内存很快充满了许多空闲空间的小洞，从而很难分配给新的段，或扩大已有的段。

3. 分页加载</br>
为了解决分段加载引入的“外部碎片”的问题，操作系统引入了分页的概念，即不是将进程的地址空间分割成几个不同长度的逻辑段，而是分割成固定大小的单元，每个单元成为一页。相应的把物理内存看成是定长槽块的阵列，叫作页帧，每个这样的页帧对应着一个虚拟内存页。

### 页表
#### 页表里有什么
以一个简单的例子来说明操作系统如何将物理内存页帧与虚拟地址页对应起来。下图左侧展示了一个只有64字节的小虚拟地址空间，有4个16字节的页：虚拟页0、1、2、3；右侧展示了由一组固定大小的槽块组成的物理内存，有8个页帧，128个字节。
<img src="/images/the-whole-life-of-a-variable/pte.png" align=center>

为了记录上图中地址空间的每个虚拟页放在物理内存中的位置，操作系统通常为每个进程保存一个数据结构，成为页表（page table），其主要作用是为地址空间的每个虚拟页保存地址转换，从而让CPU知道每个页在内存中的位置。对于上图中的例子而言，页表具有一下4个条目（PTE）：(VP0->PF3)、(VP1->PF7)、(VP2->PF5)、(VP3->PF2)。

当然，真实的PTE除了这种映射关系外，还有一些其他的内容，比如：
* 有效位：用于指示特定地址转换是否有效；
* 保护位：用于表明页是否可以读取、写入或执行；
* 存在位：用于表示该页是在物理存储器还是在磁盘上。

#### 如何进行地址转换
前面有提到过，CPU发出的地址都是虚拟地址。还是以上图的例子为例，虚拟地址空间是64字节，因此虚拟地址总共需要6位；而页的大小位16字节，所以虚拟地址会被划分如下：

<img src="/images/the-whole-life-of-a-variable/virtual-address.png" width="250" height="100" align=center>

通过虚拟页号（VPN）可以检索页表，找到其对应的物理帧号（PFN），从而可以通过用PFN替换VPN来转换此虚拟地址，然后将载入发送给物理内存。另外，偏移量保持不变，因为偏移量只是告诉我们页面中哪个字节是我们想要的。具体的地址转换过程如下图所示（以二进制虚拟地址"010101"为例）：

<img src="/images/the-whole-life-of-a-variable/address-translate.png" width="450" height="300" align=center>

#### 页表存在哪里
一个典型的32位地址空间，带有4KB的页，那么这个虚拟地址分成20位的VPN和12位的偏移量。一个20位的VPN意味着操作系统必须为每个进程管理2的20次方个转换地址（大约一百万）。假设每个页表条目需要4个字节来保存物理地址转换和其他信息，每个页表就需要巨大的4MB内存，假设有100个进程运行，则意味着操作系统需要400MB内存。而对于64位系统，这个数字更可怕。操作系统为了减小存放页表的内存，采用多级页表的手段，但仍然无法在MMU中利用任何特殊的片上硬件来存储当前正在运行的进程的页表，而是将每个进程的页表存储在内存中，甚至可以交换到磁盘上。

#### 快速地址转换
由上述描述可知，分页机制的引入，会导致CPU在进行虚拟地址转换时，需要一次额外的内存访问。每次指令获取、显式加载或保存，都要额外读一次内存以得到转换信息，这慢的无法接受。

要解决这个问题，操作系统需要硬件的支持，即地址转换旁路缓冲存储器（translation-lookaside buffer, TLB）。每次进行内存访问，硬件先检查TLB，看看其中是否有期望的转换映射，如果有，就很快完成转换，不用访问页表；如果没有，则会暂停当前的指令流，跳转至陷阱处理程序，从而检查页表中的映射，然后更新TLB。

### CPU访问变量全过程
在了解上述概念后，我们回到上一篇文章最开始的地方，再次贴出简单的代码如下：
```C
#include <stdio.h>

int variable = 100;
int main(void) {
    printf("This is variable example, variable=%d\n", variable);
    return 0;
}
```
编译后，通过`readelf -s a.out | grep variable`可以查到变量variable的虚拟地址为0x0804a01c，如下图所示：

<img src="/images/the-whole-life-of-a-variable/variable-virtual-address.png" align=center>

CPU访问该变量的全过程简化描述如下（此处忽略cache的参与）：

<img src="/images/the-whole-life-of-a-variable/variable-access-process.png" align=center>

### 小结
变量variable从源文件被编译成可执行文件，然后被加载到内存，最后被正确访问，大致过程如上述两篇文章中所概括。

