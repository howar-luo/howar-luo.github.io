---
title: linux进程、线程和调度(三)
date: 2018-07-22 20:09:42
categories: Linux
tags: Process
---

## 吞吐率 vs. 响应
计算机系统的总体性能标准是响应时间和吞吐量。

响应时间是提交请求和返回该请求的响应之间使用的时间。<br />
示例包括：
+ 数据库查询花费的时间
+ 将字符回显到终端上花费的时间
+ 访问 Web 页面花费的时间


吞吐量是对单位时间内完成的工作量的量度。<br />
示例包括：
+ 每分钟的数据库事务
+ 每秒传送的文件千字节数
+ 每秒读或写的文件千字节数
+ 每分钟的 Web 服务器命中数
这些度量之间的关系很复杂。有时可能以响应时间为代价而得到较高的吞吐量，而有时候又要以吞吐量为代价得到较好的响应时间。在其他情况下，一个单独的更改可能对两者都有提高。可接受的性能基于合理的吞吐量与合理的响应时间相结合。

吞吐和响应之间的矛盾:<br />
+ 响应 : 最小化某个任务的响应时间，哪怕牺牲其他的任务为代价
+ 吞吐: 全局视野，整个系统的workload被最大化处理

#### I/O消耗型 vs. CPU消耗型
+ IO bound: CPU利用率低，进程的运行效率主要受限于I/O速度。主要关注是否能被及时调度到，不关注CPU是否强劲。
+ CPU bound: 多数时间花在CPU上面(做运算)。主要关注CPU是否强劲。
> ARM架构中的`big.LITTLE`就是基于这种考虑：将I/O消耗型任务放到小核上执行，将CPU消耗型任务放到大核上执行。这样可以最大限度的利用CPU的性能。

## Linux任务调度机制
通用Linux系统支持实时和非实时两种进程，实时进程相对于普通进程具有绝对的优先级。对应地，实时进程采用SCHED_FIFO或者SCHED_RR调度策略，普通的进程采用SCHED_OTHER调度策略。

### 早期2.6任务调度机制
#### 优先级数组和Bitmaps
首先看下面这幅图：<br />
<img src="https://github.com/howar-luo/image_repo/blob/master/20180801/task_bitmap.png?raw=true" width="60%" height="" /><br />
早起操作系统有一个140个元素的bitmap，其中0-99是给实时任务的，100-139是给普通任务的。

#### SCHED_FIFO、SCHED_RR
SCHED_FIFO：不同优先级按照优先级高的先跑到睡眠，优先级低的再跑；同等优先级先进先出。

SCHED_RR：不同优先级按照优先级高的先跑到睡眠，优先级低的再跑；同等优先级轮转。

在不同优先级的情况下，这两者的表现是一样的，即高优先级的任务先跑，直到高优先级的任务睡眠，低优先级的任务再跑。

在相同优先级的情况下，这两种任务调度策略是有区别的：SCHED_FIFO会才有先进先出的原则进行调度，即先ready的任务跑到休眠后，才调度后ready的任务；而SCHED_RR则会轮转调度所有ready的任务，你执行一下我执行一下。

#### SCHED_NORMAL
普通任务的调度策略采用SCHED_NORMAL，不同优先级的任务进行轮转。普通任务的优先级为100-139，但是普通任务的优先级高并不会相比低优先级的任务有绝对的优势，有两个优势：可以得到更高的时间片；醒来后可以抢占低优先级任务。普通任务调度将优先级转化为nice值（-20-19）。

在运行过程中，会根据任务的睡眠情况对nice进行奖励和惩罚。任务越喜欢睡觉，系统就会越奖励；任务越喜欢运行，系统就会越惩罚。这样做的目的是想让I/O消耗型的任务在和CPU消耗型的任务竞争的过程中，能够很好的竞争。

> 在Linux后期发展过程中，增加了补丁，从而限制了实时任务最大的运行时间占用率(95%)，即实时任务即使不主动放弃CPU，系统至少还会分配5%的CPU给普通进程。


### 当前Linux任务调度机制
Linux在发展过程中，SCHED_FIFO和SCHED_RR变化并不大，主要变化的是SCHED_NORMAL。

现在的SCHED_NORAML最著名的当数CFS了，CFS使用红黑树的数据结构，如下图：<br />
<img src="https://github.com/howar-luo/image_repo/blob/master/20180802/CFS.png?raw=true" width="60%" height="" /><br />
在上述红黑树中，左边节点小于右边节点的值，系统总是选择树的最左边的节点进行调度。

其中，节点上的数字代表是的是进程运行到目前为止的vruntime，即进程多的虚拟运行时间。其计算公式可以简单按下式理解：
```
vruntime += delta* NICE_0_LOAD/ se.weight
```
其中：<br />
`delta`为实际运行时间
`NICE_0_LOAD`为标准值1024
`se.weight`为权重，如下图所示:<br />
<img src="https://github.com/howar-luo/image_repo/blob/master/20180805/nice_weight.png?raw=true" width="60%" height="" /><br />

