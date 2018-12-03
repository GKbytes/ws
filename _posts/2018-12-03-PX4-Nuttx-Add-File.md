---
layout: post
title:  "基于PX4的Nuttx实时操作系统编程"
categories: PX4
tags:  PX4 Nuttx  
author: Byte
---

* content
{:toc}

## 简介

上一篇介绍的是baro的uORB通信方法，现在同样介绍baro MS5611 加入Nuttx中编译的方法。

## Cmake基础
* 首先谈谈何为make
熟悉linux系统的朋友对make肯定不陌生，它就是用来读取Makefile文件然后执行，把源码编译链接为可执行程序的一个软件。我们只要把待编译的源码文件和这些源码所需要用到的库写到Makefile文件里面执行make命令就能得到可执行或者可烧录的二进制文件。

* 那何为Cmake呢
我们可以这样理解，早期人们直接使用gcc命令编译程序，当源码文件多起了之后，直接用gcc命令过长，链接的库过多，这样就太麻烦了。这时候Make就应运而生，解决了人们对于代码文件过多的困扰。但随着项目越来越大，文件越来越多，人们发现make也捉襟见肘，因为编写Makefile文件又会变得异常复杂。这个时候聪明的程序猿就想能不能有个程序可以帮我写Makefile文件呢？这样就引出了Cmake，当然Cmake不是智能的，它不可能自己去辨别那些是代码，那些是文件，需要什么库。这样就引入了另外一套规则，也引入了一个文件CMakeLists.txt，这个文件就是Cmake程序识别的文件，有这个文件，Cmake就能帮助程序猿自动生成Makefile文件。

小结：
* make是为了读取makefile(makefile写起来也很麻烦)
* cmake就是帮忙写makefile(cmake只认CMakeLists.txt)

## 添加baro驱动

首先编辑ms5611驱动，分别为(路径：/Firmware/src/drivers/barometer/ms5611)：
1. ms5611_i2c.cpp
2. ms5611_spi.cpp
3. ms5611.cpp
4. ms5611.h

然后编辑CMakeLists.txt文件
```js
set(srcs
	ms5611_spi.cpp
	ms5611_i2c.cpp
	ms5611.cpp
)

px4_add_module(
	MODULE drivers__ms5611
	MAIN ms5611
	STACK_MAIN 1200
	COMPILE_FLAGS
	SRCS ${srcs}
	DEPENDS
	)

```
 从字面上可以了解到每个程序在CMake里面都是一个模块，最后总的Cmake文件会去自动将这些模块添加到最后的生成的Makefile文件里面。所以我们要做的就是把我们写的模块的一些属性描写清楚，然后将其注册到nuttx_px4fmu-v2_default.cmake文件当中。
1. 首先是模块名MODULE：这个名字的命名规则是当前文件夹+__+主函数名。
2. 然后是MIAN ：
3. STACK_MAIN：
4. SRCS：所用到的源文件。
5. DEPENDS ：依赖。

"Firmware/cmake/configs/nuttx_px4fmu-v2_default.cmake"
最后在cmake文件中添加
>	drivers/barometer/ms5611
用#把不用的驱动注销掉：
>   #drivers/barometer/bmp280

在Firmware文件夹下面执行make px4fmu-v2_default 进行编译。