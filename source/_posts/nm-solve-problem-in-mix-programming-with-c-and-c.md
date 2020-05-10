---
title: 用nm工具定位C/C++混合编程中的链接问题
date: 2020-02-22 20:43:05
categories:
tags:
---

## 前言
在实际工作中，难免会用到第三方库，而有时候第三库所采用的编程语言可能跟我们项目中实际使用的语言不一致，比如C++项目使用了C的第三方库，或者C项目使用了C++的第三方库。这种情况就需要处理C/C++混合编程的问题。

## 什么是C/C++混合编程
### C++项目使用了C的第三方库
有C文件`foo.c`，其内容如下：
```C
#include <stdio.h>

void foo(void) {
    printf("This is foo function\n")
}
```
对应的头文件为`foo.h`，其内容为：
```C
#ifndef _FOO_H_
#define _FOO_H_

#ifdef __cplusplus
extern "C" {
#endif

void foo(void);

#ifdef __cplusplus
}
#endif

#endif /* _FOO_H_ */
```
先通过以下指令将其编译成静态库：
```shell
gcc -c foo.c
ar -r libfoo.a foo.o
```
`main.cpp`中调用`foo.c`内的函数，`main.cpp`的内容如下：
```C++
#include <iostream>
#include "foo.h"

int main(void) {
    foo();
    return 0;
}
```
使用以下命令编译`main.cpp`：
```shell
g++ main.cpp -L. -lfoo -o a.out
```
以上，实现C++对C的调用。

### C项目使用了C++的第三方库
有CPP文件`foo.cpp`，其内容如下：
```C++
#include <iostream>

#ifdef __cplusplus
extern "C" {
#endif

void foo(void) {
    std::cout << "This is foo function" << std::endl;
}

#ifdef __cplusplus
}
#endif
```
对应的头文件为`foo.h`，其内容为：
```C++
#ifndef _FOO_H_
#define _FOO_H_

void foo(void);

#endif /* _FOO_H_ */
```
先通过以下指令将其编译成静态库：
```shell
gcc -c foo.cpp
ar -r libfoo.a foo.o
```
`main.c`中调用`foo.cpp`内的函数，`main.c`的内容如下：
```C++
#include <stdio.h>
#include "foo.h"

int main(void) {
    foo();
    return 0;
}
```
使用以下命令编译`main.c`:
```shell
gcc main.c -L. -lfoo -o a.out -lstdc++
```
以上，实现C对C++的调用。
> 注意要加上`-lstdc++`，因为用gcc编译c++程序时，链接的库文件为libstdc++.so，而不是默认的libc.so，因此需要用-lstdc++参数指明，否则会在链接时发生错误。

## C/C++混合编程会遇到什么链接问题
通过上面的例子可以看到，都在合适的位置加上了如下代码：
```C++
#ifdef __cplusplus
extern "C" {
#endif

#ifdef __cplusplus
}
#endif
```
为什么呢？</br>
因为C只有单一的命名空间，不支持函数重载之类的特性，例如对于函数void fun(int a, int b)，经过编译后生成的符号为_fun，C链接器链接的时候就会去找_fun这样的函数符号；C++为了支持函数重载（即函数名字可以相同，参数类型或个数不同），允许存在同名的函数，这一点在C中是做不到的。其实，C++甚至可以存在相同的类型、变量等，因为在C++中命名空间的存在。在C++中，对于函数void fun(int a, int b)，经过编译后，生成的类似为_fun_int_int，新生成的符号名不仅带有函数名，还有参数类型。正因为他们两者编译函数的时候，生成的符号规则不一样，所以，在混合编程中，如果我们不进行任何处理，而相互效用的话，必然会出现在链接的时候，找不到符号链接的情况。

* 在C++调用C的情况中，可以在C对应的头文件中加入`extern "C"`，从而在编译C++代码时，可以告诉编译器按照C链接器的规则去查找相关函数。
* 在C调用C++的情况中，可以在C++的源文件中加入`extern "C"`，这样在编译C++对应接口的时候，就告诉编译器按照C链接器的规则去生成相关函数符号。此场景中对于一些复杂的情况，可能还需要增加一些包裹接口(wrapper)来封装相关C++接口，这里不作详细讨论。

由于C/C++编译器生成符号的规则不一样，从而导致C/C++混合编程时可能会出现"undefined reference to XXX"的问题。

## 使用nm工具定位上述问题
这里以上述*C项目使用了C++的第三方库*的代码为例，去掉文件`foo.cpp`中有关`extern "C"`的代码，然后用同样的方式进行编译，会出现以下问题：
<img src="/images/nm/nm-fault.png" width="842" height="117" align=center>
使用`nm`工具查看`foo.o`结果如下：
<img src="/images/nm/nm-result.png" width="704" height="225" align=center>
可以明显看出来函数`foo`对应的符号是C++形式的。

而出现上述问题，通常有两个原因：
1. 没有在合适的地方加上`extern "C"`
2. 函数定义和函数声明不一致（比如参数类型不一致）

这两种情况都可通过`nm`工具作排查。
