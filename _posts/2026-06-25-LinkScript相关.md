---
layout:     post   				    # 使用的布局（不需要改）
title:      LinkScript相关				# 标题 
subtitle:   捣鼓记录 #副标题
date:       2026-06-25 				# 时间
author:     BY 	moment					# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 编程
    - 嵌入式
---
# 前言
在移植tinyusb的时候我灵机一动想使用RT-Thread式的静态注册机制，因此大概写了一个库出来。
但是这个机制是依赖于lds等脚本的，因此我打算专门学习一下怎么编写链接脚本。



# 链接脚本的类型
链接脚本是MCU官方提供的，一般有具体的设备寄存器地址和内存之类的信息。对于
GCC编译器来说，一般是ld文件。对于keil等来说一般是sct。
```
ENTRY(Reset_Handler)          // 入口

MEMORY                        // 内存区域定义
{
  RAM (xrw) : ORIGIN = ..., LENGTH = ...
  ROM (rx)  : ORIGIN = ..., LENGTH = ...
}

SECTIONS                      // 段布局
{
  .isr_vector : { ... } >ROM
  .text       : { ... } >ROM
  .rodata     : { ... } >ROM
  .data       : { ... } >RAM AT> ROM
  .bss        : { ... } >RAM
}
```