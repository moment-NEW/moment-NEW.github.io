---
layout:     post
title:      WDFK IS DMA?
subtitle:   RT-Thread贡献记录
date:       2026-08-01
author:     moment
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 编程
    - 嵌入式
---
# 前言
做WS2812的时候要用到SPI6，然后发现SPI6驱动缺失，补完后被路过的程序员发现SPI6没有DMA，仔细一查只有BDMA（D3域），因此开始研究BDMA和相关的东西，着手写BDMA驱动。
## DMA是怎么开起来的？
RT-Thread使用Kconfig配置，然后通过Sconscript和env工具生成宏到rtconfig.h，这些我们已经知道了。DMA这类驱动也绕不开这套逻辑，需要开启DMA,我们就需要在BSP里面开启BSP_USING_DMA。当然，这里的驱动用了个小技巧：HAL库在开启DMA的时候会定义宏`HAL_DMA_MODULE_ENABLED`,因此不需要使用这个宏（当然我们BDMA是需要的，因为大部分BDMA逻辑和DMA混用，没有独立的宏）
开启之后整个ifdef会让drv_dma进入编译。但是DMA驱动，在RT-Thread里面是没有自动初始化的，实际上是通过SPI等实际外设提供配置调用其api实现的。
## DMA是如何初始化的？
