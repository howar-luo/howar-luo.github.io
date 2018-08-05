---
title: linux进程、线程和调度(二)
date: 2018-07-20 17:02:26
categories: Linux
tags: Process
---

## fork、vfork
#### fork
fork的结果，是创建出新的task_struct，当子进程的task_struct刚被创建出来是，其成员(即资源)完全是从父进程处对拷过来的，即执行一个copy。此时，任何资源的修改都造成分裂。<br>
其中，除了一种资源外其他资源的分裂都比较好理解，这种资源就是内存资源。考虑以下代码：
```
#include <sched.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int data = 10;

int child_process()
{
	printf("Child process %d, data %d\n",getpid(),data);
	data = 20;
	printf("Child process %d, data %d\n",getpid(),data);
	_exit(0);
}

int main(int argc, char* argv[])
{
	int pid;
	pid = fork();

	if(pid==0) {
		child_process();
	}
	else{
		sleep(1);
		printf("Parent process %d, data %d\n",getpid(), data);
		exit(0);
	}
}
```
这三次打印的`data`的值为多少？答案是`10 20 10`。这其中经历了写时拷贝，即copy-on-write，从微观上看，子进程中的`data = 20`这句话，执行时间是很长的。

#### 写时拷贝技术
要想理解写时拷贝，先看下面一幅图：<br/>
<img src="https://github.com/howar-luo/image_repo/blob/master/cow.PNG?raw=true" width="60%" height="" />

父进程P1中的某个内存变量有一个虚拟地址和一个物理地址，通过MMU进行映射，并且原则上将权限设置为RW。当子进程P2被创建出来后，它所看到的对应的内存变量与P1具有相同的虚拟地址和物理地址。此时，操作系统会将MMU中该页表的权限修改为RD-ONLY，这样无论P1还是P2哪个进程去写该内存变量，都会产生一个page fault，然后为写该变量的进程申请一页新的物理内存，然后把老的那一页的内容拷贝到新的物理页上，并修改进程页表。从此，虽然父子进程看到的该内存变量的虚拟地址一样，其物理地址已经不同了。之后Linux会将两个物理地址所对应的页表权限都改成RW。


***要注意的是，COW是严格要求硬件有MMU的***
#### vfork
如上可知，fork会用到COW技术。而vfork不会用到COW技术。

使用vfork时，父进程会阻塞，知道子进程执行`exit`或者`exec`。

上面介绍到，fork后，父子进程的task_struct中的资源都会对拷分裂；而vfork与fork相比，有些不同。vfork时，其他资源还是会对拷分裂，但是内存资源不会，子进程的mm_struct直接指向父进程的mm_struct，即父子进程的内存资源完全一样。如果将上述代码改成：
```
#include <sched.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int data = 10;

int child_process()
{
	printf("Child process %d, data %d\n",getpid(),data);
	data = 20;
	printf("Child process %d, data %d\n",getpid(),data);
	_exit(0);
}

int main(int argc, char* argv[])
{
	int pid;
	pid = vfork();

	if(pid==0) {
		child_process();
	}
	else{
		sleep(1);
		printf("Parent process %d, data %d\n",getpid(), data);
		exit(0);
	}
}
```
则三次打印的`data`的值会变成`10 20 20`。

## Linux线程的实现本质(clone)
上面介绍了vfork和fork的区别，那么，如果在创建进程的时候，task_struct中的所有资源都不再分裂，即父子进程中的所有资源都指向同一份(完全执行clone)，即资源共享，则这就是线程。创建线程用pthread_create，每个线程也有一个task_struct，所以线程是调度的最小单位。

## PID和TGID
首先看一幅图：<br />
<img src="https://github.com/howar-luo/image_repo/blob/master/20180730/pid_tgid.PNG?raw=true" width="60%" height="" />

可以看出，同一个进程下的多个线程具有相同的TGID(thread group id)，而通过getpid获取到的其实是TGID。实际上，每个线程在内核中都有一个独一无二的pid。

那么，如何获取线程的pid呢？可以通过系统调用获取。首先要知道获取线程的pid的系统调用号为：
```
#define __NR_gettid 224
```
那么可以通过`syscall(__NR_gettid)`来获取线程的pid。

## 进程0和进程1
每当cpu上电，bios或bootloder之类的引导程序会在一个cpu上创建一个0进程，该进程进而在其它cpu复制一个子进程，并把这些子进程的pid设置为0.0进程是通过全局静态的数据结构来初始化的，特别是主内核页全局目录放在swapper_pg_dir中。

0进程，进一步调用init函数创建子进程1号进程。

0进程也叫idle进程，实际上只有当系统中没有其它进程处于running态时，才会执行idle进程。

1进程也叫init进程，其生命周期直到系统结束，因为init进程负创建及监视其它进程。

## 孤儿进程
一个父进程退出，而它的一个或多个子进程还在运行，那么那些子进程将成为孤儿进程。孤儿进程一定会将被init进程(进程号为1)所收养吗？先看下面一张图：<br />
<img src="https://github.com/howar-luo/image_repo/blob/master/20180730/subreaper.PNG?raw=true" width="60%" height="" />

当P2和P4被杀死以后，P3被init收养，而P5被subreaper收养。所以，孤儿进程并不一定都是被init收养。

那么，如何可以成为具有subreaper属性的进程呢？答案是，使用prctl接口:`prctl(PR_SET_CHILD_SUBREAPER, 1)`。当一个进程具有subreaper属性后，就可以收养进程在托孤过程中经过该进程的孤儿进程。***注：该特性是在linux3.4中才加入的。***
