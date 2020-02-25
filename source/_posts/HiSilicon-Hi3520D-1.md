---
title: Hi3520D入门一（开发环境）
tags:
  - 海思
  - Hi3520D
  - Linux
categories:
  - HiSilicon
comments: true
date: 2019-10-06 20:09:16
---

> 学习环境
>
> OS：Deepin 15.10
>
> SDK版本：C01SPC040

自己还是第一次接触的Linux的平台，所以博客会记录自己的学习过程，由于学习环境不一样，所以以下的步骤可能会略有差异，整体应该是大同小异的。自己也是也是一个初学者，所以文章要是有错误的地方望大家指出。

<!-- more -->

### 解压SDK包

SDK包所在位置:`Hi3520D_V100R001C01SPC040/release/01.software/board/Hi3520D_SDK_V1.0.4.0.tgz`

通过tar命令解压压缩包

```bash
$ tar -zxvf Hi3520D_SDK_V1.0.4.0.tgz 
```
解压完成后应该有会以下的文件

```bash
.
├── package
│   ├── drv.tgz
│   ├── image_uclibc
│   │   ├── rootfs_hi3520d_256k.jffs2
│   │   ├── rootfs_hi3520d_64k.jffs2
│   │   ├── u-boot_hi3520d.bin
│   │   └── uImage_hi3520d_full
│   ├── mpp.tgz
│   ├── osdrv.tgz
│   └── rootfs_uclibc.tgz
├── scripts
│   └── common.sh
├── sdk.cleanup
└── sdk.unpack
```



在Hi3520D_SDK_V1.0.4.0目录下执行

```bash
$ ./sdk.unpack
```

若出现一下输出

```bash
./sdk.unpack: 2: ./sdk.unpack: source: not found
./sdk.unpack: 4: ./sdk.unpack: ECHO: not found
./sdk.unpack: 6: ./sdk.unpack: WARN: not found
./sdk.unpack: 7: ./sdk.unpack: WARN: not found
./sdk.unpack: 8: ./sdk.unpack: ECHO: not found
./sdk.unpack: 20: ./sdk.unpack: ECHO: not found
./sdk.unpack: 22: ./sdk.unpack: run_command_progress_float: not found
./sdk.unpack: 24: ./sdk.unpack: ECHO: not found
./sdk.unpack: 26: ./sdk.unpack: run_command_progress_float: not found
./sdk.unpack: 28: ./sdk.unpack: ECHO: not found
mkdir: 已创建目录 'mpp'
./sdk.unpack: 30: ./sdk.unpack: run_command_progress_float: not found
./sdk.unpack: 32: ./sdk.unpack: ECHO: not found
mkdir: 已创建目录 'drv'
./sdk.unpack: 34: ./sdk.unpack: run_command_progress_float: not found
```

说明您当前使用的系统的shell脚本解释器不是bash，所以我们需要稍微更改下文件`common.sh` `sdk.cleanup` `sdk.unpack`

将这个脚本的第一行由

```bash
#!/bin/sh
...
```

改成

```bash
#!/bin/bash
...
```

选择Shell解释器为bash即可，执行完毕后会有如下三个文件夹`drv` `mpp` `osdrv`：

```c
.
├── drv
├── mpp
├── osdrv		// 内核源码
├── package
├── scripts
├── sdk.cleanup
└── sdk.unpack

```



### 安装交叉编译工具

官方总共提供了两种工具链编译：`arm-hisiv100nptl-linux` 与`arm-hisiv200-linux`，其中arm-hisiv100nptl-linux是uclibc工具链，arm-hisiv200-linux是glibc工具链 。

* uclibc 是一种小型的C语言标准库，主要用于嵌入式；
* glibc  是Linux下的GUN C的函数库；

这里我使用的是uclibc，所以我需要安装的是arm-hisiv100nptl-linux，直接执行osdrv/toolchain/arm-hisiv100nptl-linux里的`cross.install`脚本文件：

```bash
$ sudo ./cross.install
```

若文件没有可执行权限表示需要使用chmod给文件增加可执行权限，大概的输出结果如下：

```bash
CROSS_COMPILER_PATH=/opt/hisi-linux-nptl/arm-hisiv100-linux/target/bin
Do not have version file
Delete exist directory...
Extract cross tools ...
'/opt/hisi-linux-nptl/arm-hisiv100-linux/target/bin/arm-hisiv100nptl-linux-addr2line' -> '/opt/hisi-linux-nptl/arm-hisiv100-linux/bin/arm-hisiv100-linux-uclibcgnueabi-addr2line'
...

```

使用命令gcc命令判断是否环境已经配置完成

```bash
$ arm-hisiv100-linux-gcc -v
```

若提示没有此命令，那么就说明环境变量没有配上。至此我们的开发环境搭建完毕。



