---
layout:     post   				    # 使用的布局（不需要改）
title:      记录：STM32与RT-Thread				# 标题 
subtitle:   制作BSP中的笔记 #副标题
date:       2026-04-13 				# 时间
author:     BY 	moment					# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:	
    - 嵌入式							#标签
    - 编程
---
以下部分摘自RT-thread开发手册：  
# keil编译信息
Keil 工程在编译完之后，会有相应的程序所占用的空间提示信息，如下所示：
```
linking...
Program Size: Code=48008 RO-data=5660 RW-data=604 ZI-data=2124
After Build- User command \#1: fromelf--bin.\\build\\rtthread-stm32.axf--output
rtthread.bin
".\\build\\rtthread-stm32.axf"- 0 Error(s), 0 Warning(s).
Build Time Elapsed: 00:00:07
```
上面提到的ProgramSize包含以下几个部分：
- 1）Code：代码段，存放程序的代码部分；
- 2）RO-data：只读数据段，存放程序中定义的常量；
- 3）RW-data：读写数据段，存放初始化为非0值的全局变量；
- 4）ZI-data：0 数据段，存放未初始化的全局变量及初始化为0的变量；  

编译完工程会生成一个. map的文件，该文件说明了各个函数占用的尺寸和地址，在文件的最后几行
也说明了上面几个字段的关系：  
```
Total RO Size (Code + RO Data) 53668 ( 52.41kB)  
Total RW Size (RW Data + ZI Data) 2728 ( 2.66kB)  
Total ROM Size (Code + RO Data + RW Data) 53780 ( 52.52kB)   
```
- 1）ROSize 包含了Code及RO-data，表示程序占用Flash空间的大小；
- 2）RWSize包含了RW-data及ZI-data，表示运行时占用的RAM的大小；
- 3）ROMSize包含了Code、ROData以及RWData，表示烧写程序所占用的Flash空间的大小； 

程序运行之前，需要有文件实体被烧录到STM32的Flash中，一般是bin或者hex文件，该被烧录
文件称为可执行映像文件。如图3-3中左图所示，是可执行映像文件烧录到STM32后的内存分布，它包含
RO段和RW段两个部分：其中RO段中保存了Code、RO-data的数据，RW段保存了RW-data的数据，  
由于ZI-data 都是0，所以未包含在映像文件中。  
STM32在上电启动之后默认从Flash启动，启动之后会将RW段中的RW-data（初始化的全局变量）
搬运到RAM中，但不会搬运RO段，即CPU的执行代码从Flash中读取，另外根据编译器给出的ZI地址
和大小分配出ZI段，并将这块RAM区域清零。

以下是读自源码
## RT-Thread 隐式初始化源码解读
RT-T通过在编译阶段声明函数为初始化函数，以此来实现自动且有规律的初始化。具体宏定义应该在/rt-thread/blob/master/include/rtdef.h这个文件里，部分摘录如下
我一开始找错了，找到了MVSC的
```
#define INIT_EXPORT(fn, level)                                  \
    const char __rti_level_##fn[] = ".rti_fn." level;       \
    const char __rti_##fn##_name[] = #fn;                   \
    __declspec(allocate("rti_fn$f"))                        \
    rt_used const struct rt_init_desc __rt_init_msc_##fn =  \
    {__rti_level_##fn, fn, __rti_##fn##_name};

```
但是我们通常来说用的GCC的，也就是
```
    #ifdef RT_DEBUGING_AUTO_INIT
        struct rt_init_desc
        {
            const char* fn_name;
            const init_fn_t fn;
        };

        #define INIT_EXPORT(fn, level)                                                       \
            const char __rti_##fn##_name[] = #fn;                                            \
            rt_used const struct rt_init_desc __rt_init_desc_##fn rt_section(".rti_fn." level) = \
            { __rti_##fn##_name, fn};
    #else
        #define INIT_EXPORT(fn, level)                                                       \
            rt_used const init_fn_t __rt_init_##fn rt_section(".rti_fn." level) = fn
```

首先先看结构体rt_init_desc,可以看到其是这个注册后的init宏的返回类型，由初始化等级与初始化函数构成。在最前面规定了`typedef int (*init_fn_t)(void);`，是所有初始化函数的统一规范（返回int，参数void）
```
struct rt_init_desc
{
    const char* level;
    const init_fn_t fn;
};
```
然后逐行展开。`const char __rti_##fn##_name[] = #fn; `,#fn 是 C 宏里的“字符串化”,也就是说假设fn叫uart_init这个是开了一个叫做 __rti_uart_init_name[]的字符串，内容就是 uart_init这个函数名本身。  
`rt_used const struct rt_init_desc __rt_init_desc_##fn rt_section(".rti_fn." level)`这行是比较关键的，但是大部分也一样，
rt_used看起来就像是修饰符，实际上也确实如此。搜索获知，该宏的作用是向编译器 说明这段代码有用，即使函数中没有调用也要保留编译。
例如RT-Thread自动初始化 功能使用了自定义的段，使用RT_USED会将自定义的代码段保留。
所以，返回类型为有很多修饰符的结构体rt_init_desc,名字一样是包含了fn本身，然后后面是关键的 rt_section(".rti_fn." level),
这个语法很奇怪，通常不符合C语言规范，但是这是在宏定义里面，所以会出现：
rt_section(".rti_fn." level) 不是变量名的一部分，也不是函数调用，  
它是一个属性修饰，意思是：  

把这个变量放到 .rti_fn.x 这个特殊段里  
这样，这些就会在编译时候被归类到一个section里，日后链接就可以直接塞进初始化代码里。

也就是说展开来是这样的(假设fn是hw_usart_init)
```
const char __rti_rt_hw_usart_init_name[] = "rt_hw_usart_init";

rt_used const struct rt_init_desc __rt_init_desc_rt_hw_usart_init
    __attribute__((section(".rti_fn.1"))) =
{
    __rti_rt_hw_usart_init_name,
    rt_hw_usart_init
};
```
值得注意的是，这里存的是函数指针，尽管宏定义完全可以复制整个函数进去，但是这里并非如此。  
