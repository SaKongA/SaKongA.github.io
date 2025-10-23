---
title: 快速入门在STM32标准库上的ToF激光测距传感器 - VL6180
date: 2025-2-19 15:12
updated: 2025-2-19 15:12
tags: EE
---

## 为什么是它？
这段时间参与了一个项目的控制部分的开发，需要一个STM32F103C8T6上的测距方案，要求精度高，测量的距离比较近，测量的目标点比较集中。常见的测距方案主要分为三类：超声波测距，红外测距，ToF激光测距。在最初的计划阶段，我们选择的是超声波测距，但是由于数据很不稳定，测量点不好把握，遂放弃；由于红外测距和ToF都是基于光学测距，为了节约时间，选择了成本较高，理论上效果更好的ToF激光测距，下面是对三种测距方案的简单比较：
| 测距方案 | 常见型号 | 应用场景 |
|----------|----------------------------|----------------------------|
| 超声波 | HC-SR04, US-100, MaxBotix MB1010, Parallax PING | 中短距离测量，个人认为超声波更适合小车避障 |
| 红外 | Sharp GP2Y0A21YK0F, TCRT5000, GP2Y0A02YK0F, GP2Y0A710K0F | 短距离测量，适用于简单的距离检测，比较容易受到外界环境的影响 |
| TOF激光 | VL53L0(X), VL6180(X), VL53L1(X), LIDAR-Lite v3, Garmin LIDAR-Lite v4 | 高精度和长距离测量，适用于复杂环境，比较适合单点测距，速度也比较快，价格高 |

## 初识 ToF 激光测距模块
- Time of Flight, ToF，全称激光测距模块，是一种基于飞行时间原理的测距设备，说白了，就是通过激光的传播的时间来计算距离。
- 其核心工作原理是模块发射一束激光脉冲，激光遇到目标物体后反射回来，模块接收反射光并计算发射与接收的时间差，结合光速计算出距离。

![原理图示例](https://github.com/SaKongA/picx-images-hosting/raw/master/TOF原理示例.7zqk6okadv.webp)

- TOF激光测距模块通常由激光发射器、接收器、计时电路和处理器组成，具有高精度、快速响应、非接触测量和适应性强等优点，广泛应用于机器人、自动驾驶、工业自动化和消费电子等领域。然而，在使用时需要注意环境光干扰、目标表面特性以及功耗与散热等问题，以确保测量的准确性和稳定性。
## 我的模块 - VL6180
- 购入链接：
```text
【淘宝】7天无理由退货 https://e.tb.cn/h.TIV1eeWtEyhISCl?tk=jQVheiyVK3h HU006 「ToF激光测距传感器模块 TOF050C/050F/200C/200F/400F串口IIC模块」
点击链接直接打开 或者 淘宝搜索直接打开
```
- 店家提供了多款激光测距模块，具体区别可以看下图，主要关注一下测量距离和测量盲区，我们选用的是使用VL6180的TOF05F

![不同的ToF模块](https://github.com/SaKongA/picx-images-hosting/raw/master/不同的TOF模块.77dooycfax.webp)

- 我们可以看到三种模块都支持三种通讯方式，但是不同的芯片的寄存器有所区别，而IIC通讯主要就是通过读写寄存器达到通讯的目的，故这里不介绍IIC通信，不过需要吐槽的是，商家提供的例程里面，都是什么乱七八糟的代码，小白看了绝对一脸懵逼。。我们主要使用串口通信，至于Modbus其实就是串口通信的一个衍生，具工作模式介绍如下：

![三种工作模式](https://github.com/SaKongA/picx-images-hosting/raw/master/三种工作模式.86ts272ngl.webp)

## 如何连接？
1. 我们购买这家的模块的时候，尽量购买已经焊好排座的版本，商家也会附赠一个排线，排线一端为带有防反插的接口，另一端为普通的杜邦线母口，可以直连单片机；
2. 在开始之前，我们先使用TTL转USB工具简单的测试一下模块好坏，来看一下相关的引脚定义，如果你也是购买的这家的模块，可以直接按照颜色比对，否则需要按照相关标识和说明连接：
```
TTL转USB模块            ToF模块
    RX-------------------TX
    TX-------------------RX
    3V3------------------VIN
    GND------------------GND
```
![引脚定义](https://github.com/SaKongA/picx-images-hosting/raw/master/引脚定义.41y6q0udv9.webp)

> 我们使用串口通信，所以这里就只用接 TX RX GND VIN 即可，至于 SCL SDA 是 IIC 通信会用到的引脚，可以不接悬空。
  
3. 在连接好后，使用串口调试工具打开端口，设置以下串口信息：
```
默认波特率：115200
数据位：8
校验位：无
停止位：1
流控制：无
```
4. 我们直接显示 HEX 16 进制原始数据，开启显示时间戳，可以获取到数据格式如下的数据：
```
01 03 02 00 a2 39 jk
01 03 02 00 a2 39 fd
01 03 02 00 a2 39 as
01 03 02 00 a2 39 dd
01 03 02 00 a2 39 gg
01 03 02 00 a2 39 ds
01 03 02 00 a2 39 we
```
## 原始数据的解析
1. 上面我们获取到了源源不断的传感器返回的数据，这种数据就是Modbus格式的串口数据了，我们这里目的是快速入门ToF模块，所以就不详细介绍Modbuds格式是什么了，我们来直接拆解这些数据，看看都是什么意思，首先来看一下Modbus格式的数据最基本的构成：

![Modbus数据格式](https://github.com/SaKongA/picx-images-hosting/raw/master/modbus数据格式.5xarinpma1.webp)

2. 我们把数据对应上去就可以了，拿 01 03 02 00 a2 39 we 举例，从机地址就是 01 ，也就是 0x01，功能号为 0x03，数据字节个数为 0x02，所以后面要数两个格子的数据，分别是 00 和 a2 ,组成的16进制数据就是 0x00a2，这里就是我们的核心数据了，而后面的 39 we，是CRC校验位，主要是校验数据完整性，我们就不用看了。

![Modbus数据填入](https://github.com/SaKongA/picx-images-hosting/raw/master/modbus数据格式.m7bnk8v2.webp)
3. 我们将 0x00a2 从16进制转为10进制为 162，单位为毫米（mm），从而得到了我们的距离，知道了原理之后，我们就可以实战了！

## 在STM32标准库上的实战
0. 我写驱动代码的原则和目标就是：在代码尽可能的精简下，完成我们需要的目标，不需要的东西不要乱加，所以，需要其他定制功能，请优先在驱动代码以外的调用处解决，实在不行再修改驱动代码；
1. 本次实战使用STM32F103C8T6标准库进行开发，编译器为Arm Complier 5.06，我们这里使用的是江科大的0.96寸四针OLED屏幕驱动代码的示例工程源码，以此为底板进行开发，首先在Hardware下创建相关驱动文件 TOF.c 和 TOF.h；
2. 创建一个模块和串口通信初始化函数`TOF_Init();`，我们这里使用USART2为通信串口，优先级为3，使用的时候，注意不要与其他模块的串口冲突，不过冲突了也会编译报错的~ 我们这里定义的引脚为PA2与PA3，这两个引脚为USART2默认引脚：
```c
// Hardware/TOF.c
#include "TOF.h"

void TOF_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;
    
    // 使能时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART2, ENABLE);
    
    // 配置GPIO
    // PA2 - USART2_TX
    // PA3 - USART2_RX
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
    
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_3;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
    
    // 配置USART2
    USART_InitStructure.USART_BaudRate = 115200;
    USART_InitStructure.USART_WordLength = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits = USART_StopBits_1;
    USART_InitStructure.USART_Parity = USART_Parity_No;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
    USART_Init(USART2, &USART_InitStructure);
    
    // 配置NVIC
    NVIC_InitStructure.NVIC_IRQChannel = USART2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 3;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 3;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);
    
    // 使能USART2接收中断
    USART_ITConfig(USART2, USART_IT_RXNE, ENABLE);
    
    // 使能USART2
    USART_Cmd(USART2, ENABLE);

    TOF_RxIndex = 0;
}

// USART2中断服务函数
void USART2_IRQHandler(void)
{
    if(USART_GetITStatus(USART2, USART_IT_RXNE) != RESET)
    {
        uint8_t data = USART_ReceiveData(USART2);
        
        if(TOF_RxIndex < 7)  // Modbus帧长度为7字节
        {
            TOF_RxBuffer[TOF_RxIndex++] = data;
            
            if(TOF_RxIndex == 7)
            {
                TOF_DataReady = 1;
                TOF_RxIndex = 0;
            }
        }
        
        USART_ClearITPendingBit(USART2, USART_IT_RXNE);
    }
}
```
```c
// Hardware/TOF.h

#ifndef __TOF_H
#define __TOF_H

#include "stm32f10x.h"

// 函数声明
void TOF_Init(void);

#endif
```
3. 为我们的串口添加串口终端处理；新建一个串口终端标志为`TOF_RxIndex`;新建一个标志位`TOF_DataReady`，表示数据已经接收完毕了：
```c
// Hardware/TOF.c
#include "TOF.h"

static uint8_t TOF_RxIndex = 0;  // 定义串口中断标志位
static uint8_t TOF_DataReady = 0;  // 定义数据接收完毕标志位

void TOF_Init(void)
{
    // ······ 此处省略其他代码
    TOF_RxIndex = 0;   // 添加串口中断标志位
    TOF_DataReady = 0;
}

// USART2中断服务函数
void USART2_IRQHandler(void)
{
    if(USART_GetITStatus(USART2, USART_IT_RXNE) != RESET)
    {
        uint8_t data = USART_ReceiveData(USART2);
        
        if(TOF_RxIndex < 7)  // Modbus帧长度为7字节
        {
            TOF_RxBuffer[TOF_RxIndex++] = data;
            
            if(TOF_RxIndex == 7)
            {
                TOF_DataReady = 1;
                TOF_RxIndex = 0;
            }
        }
        
        USART_ClearITPendingBit(USART2, USART_IT_RXNE);
    }
}
```
4. 然后，我们来创建一个缓冲区，用来存放我们每次测量的数据，方便后面对数据进行转化；新建一个函数`TOF_DataIsValid();`，来检查我们接收到的缓冲区数据是否有效：
```c
// Hardware/TOF.c
#include "TOF.h"

// ······ 此处省略其他代码
static uint8_t TOF_RxBuffer[8];  // Modbus格式：01 03 02 XX XX CRC CRC

void TOF_Init(void)
{
    // ······ 此处省略其他代码
}

// 检查接收到的数据是否有效
uint8_t TOF_DataIsValid(void)
{
    if(!TOF_DataReady) return 0;
    
    // 检查帧格式
    if(TOF_RxBuffer[0] != 0x01 || TOF_RxBuffer[1] != 0x03 || TOF_RxBuffer[2] != 0x02)
        return 0;
    
    // 检查CRC
    uint16_t receivedCRC = (TOF_RxBuffer[6] << 8) | TOF_RxBuffer[5];
    uint16_t calculatedCRC = ModbusCRC16(TOF_RxBuffer, 5);
    
    return (receivedCRC == calculatedCRC);
}

// 计算Modbus CRC16
static uint16_t ModbusCRC16(uint8_t *data, uint8_t length)
{
    uint16_t crc = 0xFFFF;
    
    for(uint8_t i = 0; i < length; i++)
    {
        crc ^= data[i];
        for(uint8_t j = 0; j < 8; j++)
        {
            if(crc & 0x0001)
            {
                crc >>= 1;
                crc ^= 0xA001;
            }
            else
            {
                crc >>= 1;
            }
        }
    }
    return crc;
}

// USART2中断服务函数
void USART2_IRQHandler(void)
{
    // ······ 此处省略其他代码
}
```
```c
// Hardware/TOF.h

#ifndef __TOF_H
#define __TOF_H

#include "stm32f10x.h"

// 函数声明
void TOF_Init(void);
uint8_t TOF_DataIsValid(void);

#endif
```

5. 现在我们来完成获取距离的函数`TOF_GetDistance`，由于模块有的时候工作不稳定，会回复距离为0的时候，我们这里也使用正确标志位来处理数据不正常的情况：
```c
// Hardware/TOF.c

#include "TOF.h"
#include "Delay.h"

// ······ 此处省略其他代码
static uint16_t LastValidDistance = 0;  // 保存上一次的有效距离值

void TOF_Init(void)
{
    // ······ 此处省略其他代码
}

// 检查接收到的数据是否有效
uint8_t TOF_DataIsValid(void)
{
    // ······ 此处省略其他代码
}

// 计算Modbus CRC16
static uint16_t ModbusCRC16(uint8_t *data, uint8_t length)
{
    // ······ 此处省略其他代码
}

// 获取距离值
uint16_t TOF_GetDistance(void)
{
    uint16_t distance = 0;
    
    if(TOF_DataIsValid())
    {
        distance = (TOF_RxBuffer[3] << 8) | TOF_RxBuffer[4];
        
        // 只有当测量值大于0时才更新LastValidDistance
        if(distance > 0)
        {
            LastValidDistance = distance;
        }
        else
        {
            distance = LastValidDistance;  // 如果当前值为0，返回上一个有效值
        }
        
        TOF_DataReady = 0;  // 清除数据就绪标志
    }
    else
    {
        distance = LastValidDistance;  // 如果数据无效，返回上一个有效值
    }
    
    return distance;
}

// USART2中断服务函数
void USART2_IRQHandler(void)
{
    // ······ 此处省略其他代码
}
```
```c
// Hardware/TOF.h

#ifndef __TOF_H
#define __TOF_H

#include "stm32f10x.h"

// 函数声明
void TOF_Init(void);
uint8_t TOF_DataIsValid(void);
uint16_t TOF_GetDistance(void);

#endif
```
6. 到这里我们的驱动就完成了，本质上就是一个简单的串口接收数据的功能，相比`IIC`实现要简单很多，`IIC`还要涉及到寄存器地址等内容，以下是完整代码
```c
// Hardware/TOF.c

#include "TOF.h"
#include "Delay.h"

// 接收缓冲区
static uint8_t TOF_RxBuffer[8];  // Modbus格式：01 03 02 XX XX CRC CRC
static uint8_t TOF_RxIndex = 0;
static uint8_t TOF_DataReady = 0;
static uint16_t LastValidDistance = 0;  // 保存上一次的有效距离值

void TOF_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;
    
    // 使能时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART2, ENABLE);
    
    // 配置GPIO
    // PA2 - USART2_TX
    // PA3 - USART2_RX
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
    
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_3;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
    
    // 配置USART2
    USART_InitStructure.USART_BaudRate = 115200;
    USART_InitStructure.USART_WordLength = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits = USART_StopBits_1;
    USART_InitStructure.USART_Parity = USART_Parity_No;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
    USART_Init(USART2, &USART_InitStructure);
    
    // 配置NVIC
    NVIC_InitStructure.NVIC_IRQChannel = USART2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 3;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 3;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);
    
    // 使能USART2接收中断
    USART_ITConfig(USART2, USART_IT_RXNE, ENABLE);
    
    // 使能USART2
    USART_Cmd(USART2, ENABLE);
    
    TOF_RxIndex = 0;
    TOF_DataReady = 0;
}

// 检查接收到的数据是否有效
uint8_t TOF_DataIsValid(void)
{
    if(!TOF_DataReady) return 0;
    
    // 检查帧格式
    if(TOF_RxBuffer[0] != 0x01 || TOF_RxBuffer[1] != 0x03 || TOF_RxBuffer[2] != 0x02)
        return 0;
    
    // 检查CRC
    uint16_t receivedCRC = (TOF_RxBuffer[6] << 8) | TOF_RxBuffer[5];
    uint16_t calculatedCRC = ModbusCRC16(TOF_RxBuffer, 5);
    
    return (receivedCRC == calculatedCRC);
}

// 计算Modbus CRC16
static uint16_t ModbusCRC16(uint8_t *data, uint8_t length)
{
    uint16_t crc = 0xFFFF;
    
    for(uint8_t i = 0; i < length; i++)
    {
        crc ^= data[i];
        for(uint8_t j = 0; j < 8; j++)
        {
            if(crc & 0x0001)
            {
                crc >>= 1;
                crc ^= 0xA001;
            }
            else
            {
                crc >>= 1;
            }
        }
    }
    return crc;
}

// 获取距离值
uint16_t TOF_GetDistance(void)
{
    uint16_t distance = 0;
    
    if(TOF_DataIsValid())
    {
        distance = (TOF_RxBuffer[3] << 8) | TOF_RxBuffer[4];
        
        // 只有当测量值大于0时才更新LastValidDistance
        if(distance > 0)
        {
            LastValidDistance = distance;
        }
        else
        {
            distance = LastValidDistance;  // 如果当前值为0，返回上一个有效值
        }
        
        TOF_DataReady = 0;  // 清除数据就绪标志
    }
    else
    {
        distance = LastValidDistance;  // 如果数据无效，返回上一个有效值
    }
    
    return distance;
}

// USART2中断服务函数
void USART2_IRQHandler(void)
{
    if(USART_GetITStatus(USART2, USART_IT_RXNE) != RESET)
    {
        uint8_t data = USART_ReceiveData(USART2);
        
        if(TOF_RxIndex < 7)  // Modbus帧长度为7字节
        {
            TOF_RxBuffer[TOF_RxIndex++] = data;
            
            if(TOF_RxIndex == 7)
            {
                TOF_DataReady = 1;
                TOF_RxIndex = 0;
            }
        }
        
        USART_ClearITPendingBit(USART2, USART_IT_RXNE);
    }
}
```
```c
// Hardware/TOF.h

#ifndef __TOF_H
#define __TOF_H

#include "stm32f10x.h"

// 函数声明
void TOF_Init(void);
uint16_t TOF_GetDistance(void);
uint8_t TOF_DataIsValid(void);

#endif
```

7. 最后的最后，就是调用的问题，我们这里将数据实时显示到了OLED屏幕上，可以参考一下：
```c
// User/main.c

#include "stm32f10x.h" // Device header
#include "OLED.h"
#include "TOF.h"

int main(void)
{
	OLED_Init();  // 初始化OLED屏幕
	TOF_Init();  //初始化TOF测距模块

	while (1)
	{
		uint16_t distance = TOF_GetDistance();  // 获取数值
		sprintf(DistanceStr, "Distance:%4dmm", distance);  // 赋值
		OLED_ShowString(0, 0, DistanceStr, OLED_8X16);  // 写入显示屏幕
		OLED_Update();  // 更新屏幕
	}
}
```