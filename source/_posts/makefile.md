---
title: 初见Makefile
date: 2018-07-17 17:32:02
categories: Linux
tags: Makefile
---


## 1. makefile中包含什么
&emsp;&emsp;Makefile中主要包含了五部分内容：显示规则、隐晦规则、变量定义、文件指示、注释。

### 1.1 显示规则
&emsp;&emsp;显式规则说明了，如何生成一个或多个的目标文件。这是由Makefile的书写者明显指出，要生成的文件，文件的依赖文件，生成的命令。如下所示：

	main.o : main.c defs.h
		cc -c main.c

&emsp;&emsp;显示的指出目标、目标的依赖、已经生成目标的命令。

### 1.2 隐晦规则
&emsp;&emsp;GNU的make很强大，它可以自动推导文件以及文件依赖关系后面的命令，于是我们就没必要去在每一个[.o]文件后都写上类似的命令，因为，我们的make会自动识别，并自己推导命令。

&emsp;&emsp;只要make看到一个[.o]文件，它就会自动的把[.c]文件加在依赖关系中，如果make找到一个whatever.o，那么whatever.c，就会是whatever.o的依赖文件。并且 cc -c whatever.c 也会被推导出来，于是，我们的makefile再也不用写得这么复杂。我们的是新的makefile又出炉了。

&emsp;&emsp;所以，可以有一下makefile：

	objects = main.o kbd.o command.o display.o insert.o search.o files.o utils.o
	
	edit : $(objects)
		cc -o edit $(objects)

	main.o : defs.h
	kbd.o : defs.h command.h
	command.o : defs.h command.h
	display.o : defs.h buffer.h
	insert.o : defs.h buffer.h
	search.o : defs.h buffer.h
	files.o : defs.h buffer.h command.h
	utils.o : defs.h

### 1.3 变量定义
&emsp;&emsp;如上一小节中的“objects”即为定义的变量。

&emsp;&emsp;而Makefile中定义变量有四种方式：

&emsp;&emsp;**立即赋值**：a:=b。会立即计算b的值，并赋值给a。如：
	
		var1:=$(var2) bar
		var2:=foo
&emsp;&emsp;则var1的值就为"bar"。

&emsp;&emsp;**延迟赋值**：a=b。相当于C++和java的引用。如果后面b的值改变了，那么a的值也会改变。如：

		var1=$(var2) bar
		var2:=foo

&emsp;&emsp;var1的值就为"foo bar"。

&emsp;&emsp;**条件赋值**：a?=b。如果a没有定义，则相当于a=b ，否则不执行任何操作。

&emsp;&emsp;**附加赋值**：a+=b。将b的值添加到a原有的值后面，再赋值给a。如：

		var1=func1.o func2.0
		var2+=$(var1) func3.o

&emsp;&emsp;则var2的值为"func1.o func2.o func3.o"。
### 1.4 文件指示
&emsp;&emsp;主要分三部分：

&emsp;&emsp;a. 在makefile中引用另一个makefile，就像c语言中的include一样；

&emsp;&emsp;b. 根据某些情况指定Makefile中的有效部分，就像C语言中的预编译#if一样；

&emsp;&emsp;c. 定义一个多行的命令。

### 1.5 注释
&emsp;&emsp;Makefile中只有行注释，和UNIX的Shell脚本一样，其注释是用“#”字符，这个就像C/C++中的“//”一样。如果你要在你的Makefile中使用“#”字符，可以用反斜框进行转义，如：“/#”。
