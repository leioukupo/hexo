---
abbrlink: ''
categories:
- - 单片机
date: '2023-12-10T15:43:59.244423+08:00'
tags:
- STM32
title: FSMC
updated: '2023-12-10T16:53:28.248+08:00'
---
# FSMC----灵活的静态存储控制器

> 能够与同步或异步存储器和16位PC存储器卡连接，[STM32](https://so.csdn.net/so/search?q=STM32&spm=1001.2101.3001.7020)的FSMC接口支持包括 `SRAM`、`NANDFLASH`、`NORFLASH`和 `PSRAM`等存储器，支持8/16/32/位数据宽度。

![fsmc框图](https://raw.githubusercontent.com/leioukupo/img/main/fsmc.png))

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

