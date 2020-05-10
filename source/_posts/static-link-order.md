---
title: Linux C/C++对静态链接库顺序的要求
date: 2019-05-15 20:09:47
categories:
tags:
---

## 前言
工作中出现两次使用第三方静态库，明明所有的symbol都在库中，却在链接时出错的问题。这和链接器工作原理有关，这里试着分析一下。

本文在x86 ubuntu上做实验。

## 基础知识介绍
### Linux下的静态链接库里有什么
一个静态库可以简单地看成一组目标文件的集合，即很多目标文件经过压缩打包后形成的一个文件。比如Linux中最常用的C语言静态库libc.a，它属于glibc项目的一部分。glibc本身是用C语言开发的，它由成百上千个C语言源代码组成，也就是说，编译完成以后有相同数量的目标文件，比如输入输出有printf.o/scanf.o；文件操作有fread.o/fwrite.o；时间日期有data.o/time.o；内存管理有malloc.o等。把这些零散的目标文件直接提供给库的使用者，很大程度上会造成文件传输、管理和组织方面的不便，于是通常是用“ar”压缩程序将这些目标文件压缩到一起，并且对其进行编号和索引，以便查找和检索，就形成了libc.a这个静态库文件。

在我的64为ubuntu系统中，libc.a位于`/usr/lib/x86_64-linux-gnu/`目录下，可以通过命令`ar -t /usr/lib/x86_64-linux-gnu/libc.a`查看lib.a中的内容如下（很多，只截取其中一部分）：

<img src="/images/static-link-order/lib-content.png" width="541" height="310" align=center>

### object文件里有哪些符号
下面分析链接器工作原理的时候需要知道目标文件里有哪些符号，这里简要说明一下。

以下面C程序为例，假设其保存在sample.c中：
```C
#include <stdio.h>

extern void callback(void);
static void func(void) {
    callback();
}

void start(void) {
    func();
}
```
使用命令`gcc -c sample.c`对其进行编译，得到目标文件`sample.o`。然后使用`nm`命令可以查看其中的符号，如下：

<img src="/images/static-link-order/object-content.png" width="691" height="71" align=center>

上图的含义是：  

* `start`函数是一个外部符号，即在本目标文件里定义，但会在外部访问。
* `func`函数是一个内部符号，即在本目标文件里定义，在本目标文件里访问。
* `callback`函数是一个未定义函数符号，链接器希望在别的地方找到它的定义。

## 链接器工作原理
如果想搞清楚为什么Linux C/C++在链接时对静态链接库的顺序有要求，必须要搞清楚静态链接器的工作原理。通过查阅"Linkers and Loaders"等资料，总结如下：</br>
首先要知道，链接器维护着一个符号表，这个符号表里记录了很多东西，不过这里我们只关心两个列表：
* 一个列表中记录着目前为止链接器解析到的所有外部符号（即可供外部访问的符号）
* 一个列表中记录着目前为止链接器解析到的所有未定义符号（即在此处访问，但未定义的符号）

而链接器更新上述两个列表的过程即其工作原理的重点，分两种情况进行分析：
1. ***当链接器载入一个目标文件***
* 当它解析到一个外部符号，会把它加入上述外部符号列表中；如果该符号存在于上述未定义符号列表中，则将其移除，因为它已经被找到了；但如果该符号已经存在于外部符号列表中，则会报“multiple definition”的错误，即两个不同的目标文件都引入了相同的符号，链接器就懵逼了
* 当它解析到一个未定义符号，会把它加入为定义符号列表中，除非这个符号可以在外部符号列表中找到
2. ***当链接器载入一个静态库***</br>
当链接器载入一个静态库时，其行为跟载入一个目标文件不太一样，它会遍历静态库中的每一个目标文件，对于每一个目标文件：
* 它首先会查看其外部符号，如果其中任一个外部符号在链接器的未定义符号列表中，那么这个目标文件就被加入链接，并且下面的步骤会被执行；否则这个目标文件整个都不会被加入链接，且下面的步骤也不会被执行，即这个目标文件的所有符号被舍弃
* 如果目标文件被加入链接，那么其会被上述***当链接器载入一个目标文件***一样处理

最终，当链接器完成上述工作后，它会查看上述符号表，如果其中未定义符号列表中仍然有内容，那么链接器将会抛出一个“undefined reference”的错误。

## 例子
本文中的编译基于cmake。
1. exampe 1
首先展示各模块/文件/函数的依赖/调用关系如下图：

<img src="/images/static-link-order/example1-func-call.png" width="406" height="446" align=center>

对于这种情况，编译`main.c`时链接liba.a和libb.b的顺序是如何的呢？当理解了上述链接器工作原理，可以清楚的知道，只需要顺序链接liba和libb就行了。

这里将文件结构展示如下：
<img src="/images/static-link-order/example1-file.png" width="704" height="208" align=center>

其中最外层的`CMakeLists.txt`内容如下：
```cmake
cmake_minimum_required(VERSION 3.5)

project(static-lib-link C CXX)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/lib-a
    ${CMAKE_CURRENT_SOURCE_DIR}/lib-b
)

add_subdirectory(lib-a)
add_subdirectory(lib-b)

set(BIN_FILES ${CMAKE_CURRENT_SOURCE_DIR}/main.c)

add_executable(static-lib-link ${BIN_FILES})
target_link_libraries(static-lib-link
    PRIVATE
        liba
        libb
)
```

2. example2
同样，首先展示各模块/文件/函数的依赖/调用关系如下图：

<img src="/images/static-link-order/example2-func-call.png" width="540" height="455" align=center>

对于这种情况，编译`main.c`时链接liba.a和libb.b的顺序与上述`example1`会有何不同？

这里将文件结构展示如下：
<img src="/images/static-link-order/example2-file.png" width="703" height="278" align=center>

如果最外层`CMakeLists.txt`与`example1`保持一致的话，会出现链接失败，结果如下：
<img src="/images/static-link-order/example2-build-fail.png" width="800" height="342" align=center>

但将其内容改为如下：
```cmake
cmake_minimum_required(VERSION 3.5)

project(static-lib-link C CXX)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/lib-a
    ${CMAKE_CURRENT_SOURCE_DIR}/lib-b
)

add_subdirectory(lib-a)
add_subdirectory(lib-b)

set(BIN_FILES ${CMAKE_CURRENT_SOURCE_DIR}/main.c)

add_executable(static-lib-link ${BIN_FILES})
target_link_libraries(static-lib-link
    PRIVATE
        liba
        libb
        liba
)
```
则链接成功。

3. 为什么
对比`example1`和`example2`的文件组织结构可以看到：
* 在`example1`中，函数`func_a1`和`func_a2`同在文件`a.c`中，那么链接器在载入`liba.a`并遍历目标文件`a.o`时，由于函数`func_a1`在未定义符号列表中，因此会将`a.o`加入链接，从而函数`func_a2`被加入了外部符号列表中；因此链接器在遍历`b.o`时便能找到`func_a2`，从而链接成功
* 而在`example2`中，函数`func_a1`和`func_a2`分别处在两个文件中。链接器在载入`liba.a`并遍历目标文件`a1.o`时，由于函数`func_a1`在未定义符号列表中，因此会将`a1.o`加入链接，而在遍历`a2.o`时，其函数`func_a2`并未在未定义列表中，因此不会将`a2.o`加入链接。那么在载入`libb.a`进行链接时会出现符号未定义的错误，需要在对`libb.a`的链接之后再增加一次对`liba.a`的链接。

## 小结
1. 链接器为何要这么做？
以`C library`为例，这里面提供了相当多的接口，有一些（或者说绝大部分）在我们当前的开发中会用不到，那么链接器的这个特性就会帮助我们仅仅将我们需要的接口链接进我们的可执行文件，从而将我们的可执行文件减小到最小。

2. 要避免循环调用的情况
上面展示了简单的循环调用情况，当然对于更复杂的循环调用，只要按照上述分析就可以梳理出链接顺序。当然，更重要的是，尽量避免出现循环调用的情况。


