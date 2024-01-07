---
abbrlink: ''
categories:
- - 单片机
date: '2024-01-07T16:51:22.062475+08:00'
tags:
- STM32
title: RTC配置
updated: '2024-01-07T16:54:03.996+08:00'
---
# RTC配置应用

## 日历配置流程

1. 使能PWR时钟

```c
RCC_APB1PeriphClockCmd()
```

2. 使能后备寄存器访问

```c
PWR_BackupAccessCmd()
```

3. 配置RTC时钟源，使能RTC时钟

```c
RCC_RTCCLKConfig()
RCC_RTCCLKCmd()
# 如果使用LSE    要打开LSE
# RCC_LSEConfig(RCC_LSE_ON)
```

4. 初始化RTC(同步/异步分频系数和时钟格式)

```c
RTC_Init ()
```
5. 设置时间

```c
RTC_SetTime ()
```

6. 设置日期

```c
RTC_SetDate()
```



