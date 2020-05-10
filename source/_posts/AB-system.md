---
title: 基于u-boot+linux平台打造A/B双系统启动和升级
date: 2019-02-05 15:14:42
categories:
tags:
---

## 前言
对于嵌入式系统，通常使用u-boot引导Linux内核然后挂载文件系统的方式，一般都只有一个Linux内核和一个文件系统。这种平台如果考虑升级的话，一般只能对应用程序进行升级，如果想对系统进行升级，有可能会出现变砖的情况。对系统要求较高的系统，必须做完备的考虑，防止在刷写过程中变砖的出现。采用A/B系统备份更新的方案可以避免这种情况的出现；此外A/B系统备份更新的方案，还能防止因某个分区损坏等原因造成的系统无法启动的问题。

本方案以uboot(version:2018.07)和linux(version:4.19)为环境，在s32v234sbc硬件平台上实现。

## 总体方案介绍
### 系统分区
A/B系统升级需要对存储器进行分区设计，支持双系统升级需要两对boot/rootfs分区，如下表所示：
<table border="2" bordercolor="black" width="300" cellspacing="0" cellpadding="5">
   <tr>
      <td>bootloader</td>
      <td>裸扇区</td>
   </tr>
   <tr>
      <td>boot_a</td>
      <td rowspan="4"> 独立分区 </td>
   </tr>
   <tr>
      <td>rootfs_a</td>
   </tr>
   <tr>
      <td>boot_b</td>
   </tr>
   <tr>
      <td>rootfs_b</td>
   </tr>
</table>

其中：
* bootloader分区烧录bootloader（如u-boot）
* boot_a分区存放A系统的Image和dtb文件
* rootfs_a分区存放A系统的文件系统文件
* boot_b分区存放B系统的Image和dtb文件
* rootfs_b分区存放B系统的文件系统文件

### 启动流程
A/B系统的启动需要bootloader支持对双系统的引导，需要关注三个重要的场景：
* 正常引导启动
* 当前分区引导启动异常时，需要重试
* 当前分区引导启动异常且重试最大次数（3次）仍然失败，则切换分区引导启动

uboot在引导启动中时，实际上是使用"mmcpart"和"mmcroot"这两个环境变量来选择引导和加载的内核以及文件系统。为支持对双系统的引导且支持上述场景，需要增加环境变量：
* boot_attempt_count: 当前分区尝试引导次数
* boot_fail_flag: 某一分区引导异常且重试最大次数（3次）后仍然失败
* boot_bank: 当前启动分区标识

启动逻辑如下图所示：

<img src="/images/AB-system/uboot-boot.png" width="910" height="493" align=center>

### 升级流程
基于系统分区和启动流程对A/B双系统的支持，可进行A/B更新刷写，具体的流程如下图所示：

<img src="/images/AB-system/update-process.png" width="270" height="525" align=center>

## 方案实现
### 下载u-boot源码
由于本方案需要修改并重新编译u-boot，所以首先需要下载u-boot源码。本文基于硬件平台s32v234sbc，NXP官方已经有对应u-boot仓库，因此只需要执行一下命令即可下载u-boot源码：
```shell
git clone https://source.codeaurora.org/external/autobsps32/u-boot
cd u-boot
git checkout -b <branch_name> v2018.07_bsp22.0
```

### 编译u-boot
下载并修改（修改方案如下文）后，需要重新编译u-boot，使用一下命令：
```sh
 make ARCH=aarch64 CROSS_COMPILE=/opt/toolchain/aarch64/bin/aarch64-linux-gnu- s32v234sbc_defconfig
 make CROSS_COMPILE=/opt/toolchain/aarch64/bin/aarch64-linux-gnu-
```
其中`/opt/toolchain/aarch64`为我本机存放交叉编译工具链的目录。

### Image/rootfs下载
本实验中所使用的Image/dtb/rootfs均从NXP官网下载。当然也可以下载源码或通过yocto进行编译。

### u-boot对双系统启动的支持
#### 增加相关环境变量
如上文中提到，为支持对双系统的引导，需要增加环境变量：
* boot_attempt_count: 当前分区尝试引导次数
* boot_fail_flag: 某一分区引导异常且重试最大次数（3次）后仍然失败
* boot_bank: 当前启动分区标识

具体实现中，需对文件`include/configs/s32.h`进行修改：

修改宏 `CONFIG_EXTRA_ENV_SETTINGS` ，在其末尾增加上述变量，如下：
```c
	"boot_fail_flag=0\0" \
	"boot_attempt_count=0\0" \
	"boot_bank=0\0"
```

#### 修改引导逻辑
u-boot默认只支持对单系统的引导，为支持上述2.2节所述的启动流程，需对文件`include/configs/s32.h`进行修改：

1. 首先增加以下宏：
```c
#define UPDATE_BOOT_PARAM_CMD "echo Switch boot bank ...;" \
	"if test ${boot_bank} = 0; then " \
		"setenv mmcpart 1; " \
		"setenv mmcroot /dev/mmcblk0p2 rootwait rw; " \
	"else " \
		"setenv mmcpart 3; " \
		"setenv mmcroot /dev/mmcblk0p4 rootwait rw; " \
	"fi; "

#define SWITCH_BOOT_BANK_CMD "echo Switch boot bank ...;" \
	"if test ${boot_bank} = 0; then " \
		"setenv boot_bank 1; " \
	"else " \
		"setenv boot_bank 0; " \
	"fi; "

#define UPDATE_BOOT_ATTEMPT_COUNT_CMD "echo Update boot attempt count ${boot_attempt_count} ...; " \
    "if test ${boot_attempt_count} = 0; then " \
	    "setenv boot_attempt_count 1; " \
	"else " \
	    "if test ${boot_attempt_count} = 1; then " \
		    "setenv boot_attempt_count 2; " \
		"else " \
		    "if test ${boot_attempt_count} = 2; then " \
		        "setenv boot_attempt_count 3; " \
			"else " \
                "if test ${boot_fail_flag} = 0; then " \
				    "setenv boot_fail_flag 1; " \
					"setenv boot_attempt_count 0; " \
					SWITCH_BOOT_BANK_CMD \
					"saveenv; " \
			    "fi; "	\
			"fi; "	\
		"fi; "	\
	"fi; "

#define SYSTEM_RESET_CMD \
	"if test ${boot_fail_flag} != 1 || test ${boot_attempt_count} != 3; then " \
		"saveenv; " \
		"reset; " \
	"fi; "
```
2. u-boot引导实际使用宏 `CONFIG_BOOTCOMMAND` ，需对其进行修改：
```c
#define CONFIG_BOOTCOMMAND \
	    "mmc dev ${mmcdev}; if mmc rescan; then " \
	       UPDATE_BOOT_ATTEMPT_COUNT_CMD \
		   UPDATE_BOOT_PARAM_CMD \
		   "if run loadimage; then " \
				"run mmcboot; " \
		   "else " \
		       SYSTEM_RESET_CMD \
		   "fi; " \
	   "else " \
			SYSTEM_RESET_CMD \
	   "fi"
```
以增加u-boot启动时对双系统引导、失败后重试、自动修改待引导系统分区等的支持。

### Linux上对修改u-boot环境变量的支持
在上文中提到，完成升级后，若要实现复位后从新分区启动的目的，需要在Linux上对u-boot的环境变量boot_bank进行修改。而u-boot的环境变量存储在裸分区，只有在u-boot下使用setenv命令对其进行修改。若在Linux上对其进行修改，需要使用到u-boot提供的工具`fw_setenv`。该工具的源码在u-boot目录tools/env下，需要对其进行修改，以符合本方案使用的硬件平台（s32v234sbc）。需对`tools/env/fw_env_private.h`文件进行修改，修改方案如下：
```c
//#define CONFIG_FILE     "/etc/fw_env.config"

#ifndef CONFIG_FILE
//#define HAVE_REDUND /* For systems with 2 env sectors */
#define DEVICE1_NAME      "/dev/mmcblk0"
#define DEVICE2_NAME      "/dev/mtd2"
#define DEVICE1_OFFSET    0xC0000
#define ENV1_SIZE         0x2000
#define DEVICE1_ESIZE     0x4000
#define DEVICE1_ENVSECTORS     2
#define DEVICE2_OFFSET    0x0000
#define ENV2_SIZE         0x4000
#define DEVICE2_ESIZE     0x4000
#define DEVICE2_ENVSECTORS     2
#endif
```
其中：
* 注释掉宏定义`CONFIG_FILE`，采用静态配置
* `DEVICE1_NAME`为实际的device名
* `DEVICE1_OFFSET`和`ENV1_SIZE`需要根据`include/configs/s32.h`中的相关配置进行修改，查找其相关配置如下：
```C
#define CONFIG_SYS_NO_FLASH
#define CONFIG_ENV_SIZE			(0x2000) /* 8 KB */
#define CONFIG_ENV_OFFSET		(0xC0000) /* 12 * 64 * 1024 */
#define CONFIG_SYS_MMC_ENV_DEV		0
#define CONFIG_MMC_PART			1
```
如上对`fw_setenv`源文件进行修改后执行编译命令：
```C
make CROSS_COMPILE=<your cross-compiler prefix> envtools 
```
可得到工具`fw_printenv`的可执行文件，然后执行：
```C
ln -s fw_printenv fw_setenv
```
即可得到u-boot环境变量修改工具，从而可实现在Linux下修改u-boot环境变量。






