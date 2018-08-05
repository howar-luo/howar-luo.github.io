---
title: Linux shell标准输入/输出/错误重定向
date: 2018-07-16 19:32:11
categories: Shell
tags: Shell
---


## 1. 什么是文件描述符
&emsp;&emsp;文件描述符即file descriptor，是内核为了高效管理已经被打开的文件所创建的索引。它是一个非负整数，用于指被打开的文件。所有执行I/O操作的系统调用都是通过文件描述符。

&emsp;&emsp;程序刚刚启动时，默认分配文件描述符0为标准输入、1为标准输出、2为标准错误。

## 2. 文件重定向
### 2.1 标准输出重定向
&emsp;&emsp;标准输出重定向，使用符号“>”,它表示程序输出至文件。该符号右侧文件若存在，则清空文件中的内容，并将新的内容输入至该文件；该符号右侧文件若存在，则首先创建该文件，并将新的内容输入至该文件。

&emsp;&emsp;而重定向符号“>>”则表示追加的意思。

&emsp;&emsp;值得注意的是，“>file”其实是"1>file"的效果。
### 2.2 标准错误重定向
&emsp;&emsp;标准错误重定向跟标准输出重定向其实是类似的，只是重定向符号前面的文件描述符（2）不能省略，即：2>file
### 2.3 标准输入重定向
&emsp;&emsp;使用符号“<”。举个简单的例子，就知道怎么用了。

&emsp;&emsp;假设当前路径下有两个文件：file1、file2。其中，file1中为空，file2中有内容：stdin test。

&emsp;&emsp;在shell下输入命令：cat >file1，则shell会等待用户输入内容，该内容会输入file1中。但是如果在shell下输入命令：cat >file1 <file2,则shell将file2中的内容输入进file1中。
## 3. 高级用法
### 3.1 “>/dev/null 2>&1”是啥意思
&emsp;&emsp;将标准输出重定向到设备/dev/null中，并且标准错误“像标准输出一样”重定向到/dev/null中。
### 3.2 “>/dev/null 2>1”是啥意思
&emsp;&emsp;将标准输出重定向到设备/dev/null中，将标准错误重定向到文件名为1的文件中。
### 3.3 “>file 2>file1”和“>file 2>&1”有啥区别
&emsp;&emsp;前者会打开两次文件file，这样stdout和stderr会互相覆盖，这样相当于使用了FD1和FD2两个同时去抢占file的管道；

&emsp;&emsp;而后者中，stderr继承了FD1管道后，再被送往file，此时file只被打开一次，只使用了一个管道FD1，因此不存在覆盖的问题。
### 3.4 “>file 2>&1”和“2>&1 >file”有啥区别
&emsp;&emsp;如上分析，前一个命令表示：标准输入和标准错误通过同一个管道重定向到file中。

&emsp;&emsp;后一个命令无法达到这种效果，应该2>&1是标准错误拷贝了标准输出的行为，但此时标准输出还是在终端，>file后才被重定向到file，所以其标准错误最终仍然保持在终端。
