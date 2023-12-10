---
abbrlink: ''
categories:
- - 单片机
date: '2023-12-10T15:43:59.244423+08:00'
tags:
- STM32
title: FSMC
updated: '2023-12-10T21:09:07.725+08:00'
---
# FSMC----灵活的静态存储控制器

> 能够与同步或异步存储器和16位PC存储器卡连接，STM32的FSMC接口支持包括 `SRAM`、`NANDFLASH`、`NORFLASH`和 `PSRAM`等存储器，支持8/16/32/位数据宽度。

![fsmc框图](https://raw.githubusercontent.com/leioukupo/img/main/fsmc.png)

fsmc驱动LCD的原理------->nor存储控制器把TFTLCD当成一个SRAM来用，有两个地址的SRAM

![fsmc存储块](https://raw.githubusercontent.com/leioukupo/img/main/FSMC%E5%AD%98%E5%82%A8%E5%9D%97.png)

！注意：

> 当Bank1接的是16位宽度存储器的时候：HADDR[25:1]->FSMC\_A[24:0]；
> 
> 当Bank1接的是8位宽度存储器的时候：HADDR[25:0]->FSMC\_A[25:0]；
> 
> 不论外部接8位/16位宽设备，FSMC\_A[0]永远接在外部设备地址A[0]。

**STM32F4仅写时序DATAST需要+1**

**SRAM模式A**

![模式A](https://raw.githubusercontent.com/leioukupo/img/main/FSMC-SRAM.png)

```c
//LCD地址结构体
typedef struct
{
    vu16 LCD_REG; // 0110 1100 0000 0000 0000 0111 1111 1110
    vu16 LCD_RAM; //0110 1100 0000 0000 0000 1000 0000 0000    16进制+1--->8进制加2
} LCD_TypeDef;

//使用NOR/SRAM的 Bank1.sector4,地址位HADDR[27,26]=11 A10作为数据命令区分线 
#define LCD_BASE        ((u32)(0x6C000000 | 0x000007FE))
//注意当Bank1接的是16位宽度存储器的时候：HADDR[25:1] ---> FSMC_A[24:0]
//右移一位对齐，对应到地址引脚即A10:A0 --- > 0011 1111 1110     A10是 0
//0x6C000000 ---> 0110 1100 0000 0000 0000 0000 0000 0000  
//0x000007FE ---> 0000 0000 0000 0000 0000 0111 1111 1110
//相与的结果
//                            0110 1100 0000 0000 0000 0111 1111 1110
#define LCD             ((LCD_TypeDef *) LCD_BASE)
```

```c
LCD_BASE，根据外部电路的连接来确定，如Bank1.sector4就是从地址0X6C000000开始，而0X000007FE，则是A10的偏移量
7FE ---> 0111 1111 1110
16位数据，地址右移一位对齐，对应到地址引脚  ---> 0011 1111 1111
#define LCD  ((LCD_TypeDef *) LCD_BASE)强制转换后，LCD_REG---> 0110 1100 0000 0000 0000 0111 1111 1110  
对应到A10是0，16位+1即8位加2  得到0110 1100 0000 0000 0000 1000 0000 0000  A10是1  
从而实现对RS的控制
```

