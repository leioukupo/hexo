---
abbrlink: ''
categories:
- - 单片机
date: '2024-01-07T16:51:22.062475+08:00'
tags:
- STM32
title: RTC配置
updated: '2024-04-04T23:49:28.209+08:00'
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

## 闹钟配置流程

1. 初始化RTC相关参数
2. 关闭闹钟

```c
RTC_AlarmCmd(RTC_Alarm_A,DISABLE)
```

3. 配置闹钟参数

```c
RTC_SetAlarm()
```

4. 开启闹钟

```c
RTC_AlarmCmd(RTC_Alarm_A,EABLE)
```

5. 开启配置闹钟中断

```c
RTC_ITConfig()
EXTI_Init()
NVIC_Init()
```

6. 编写中断服务函数

```c
RTC_Alarm_IRQHandler()
```

## RTC周期性自动唤醒配置流程

1. 初始化RTC相关参数
2. 关闭WakeUp

```c
RTC_WakeUpCmd(DISABLE)
```

3. 配置WakeUp时钟分频系数/来源

```c
RTC_WakeUpClockConfig()
```

4. 设置WakeUp自动装载寄存器

```c
RTC_SetWakeUpCounter()
```

5. 使能WakeUp

```c
RTC_WakeUpCmd( ENABLE);
```

6. 开启配置闹钟中断

```c
RTC_ITConfig()
EXTI_Init()
NVIC_Init()
```

7. 编写中断服务函数

```c
RTC_WKUP_IRQHandler();
```
