---
abbrlink: ''
categories:
- - STM32
date: '2024-04-14T15:35:43.254787+08:00'
tags:
- STM32
title: ADC使用
updated: '2024-04-14T16:15:04.186+08:00'
---
# stm32的ADC单通道单次使用

## ADC通道与引脚对应关系

![https://github.com/username/repo/blob/main/Qexo/24/4/image_d766dd94b450b38b8d81a2db02a4d03c.png](https://github.com/username/repo/raw/master/Qexo/24/4/image_d766dd94b450b38b8d81a2db02a4d03c.png)
**stm32f1没有内部温度传感器**

## 单次ADC采集流程

> 1.开启对应时钟和ADC1时钟，设置PA1为模拟输入

```cpp
// 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE); // GPIOA的时钟定义错误，应就近挂在APB2上，而不能图省事直接挂在AHB上
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1, ENABLE);
    RCC_ADCCLKConfig(RCC_PCLK2_Div6); // 设置ADC时钟，设置ADC分频因子6 72M/6=12,ADC最大时间不能超过14M
    // 初始化GPIO
    GPIO_InitTypeDef ADC_PA1;
    ADC_PA1.GPIO_Pin = GPIO_Pin_1;
    ADC_PA1.GPIO_Mode = GPIO_Mode_AIN;// 模拟输入模式
    ADC_PA1.GPIO_Speed = GPIO_Speed_50MHz;// 50MHz
    GPIO_Init(GPIOA, &ADC_PA1);
```

> 2.复位ADC1，同时设置ADC1参数

```
ADC_DeInit(ADC1);
    adc1_1.ADC_Mode = ADC_Mode_Independent;// adc工作模式 独立模式
    adc1_1.ADC_ContinuousConvMode = DISABLE;// 连续转换是否开启 
    adc1_1.ADC_DataAlign = ADC_DataAlign_Right;// 数据对齐方式
    adc1_1.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;// 触发转换方式 软件触发
    adc1_1.ADC_ScanConvMode = DISABLE;// 扫描模式是否开启
    adc1_1.ADC_NbrOfChannel = 1;// 转换通道数目
    ADC_Init(ADC1, &adc1_1);
```

> 3.使能ADC

```cpp
ADC_Cmd(ADC1,ENABLE);
```

> 4.配置规则通道参数

```cpp
ADC_RegularChannelConfig(ADC1, channel, 1, ADC_SampleTime_7Cycles5); // 通道的转换顺序 如果设置Rank为1
                                                                     //那么这个通道将会首先被转换
```

> 5.开启软件转换

```cpp
ADC_SoftwareStartConvCmd(ADC1,ENABLE); // 软件触发转换
```

> 6.等待转换完成，读取ADC值

```cpp
ADC_GetConversionValue(ADC1);
```

