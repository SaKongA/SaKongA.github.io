---
title: MaixBit + STM32 + PCA9685 实现的双向舵机垃圾分类项目 - 代码调用部分解析
date: 2024-11-24 17:48:26
updated: 2024-11-25 00:08:16
tags: EE
---
## MaixBit - 图像识别处理部分
- `Path:` main.py
- 代码逻辑图
```plaintext
lcd_show_except
 ├── sys.print_exception  (格式化异常信息)
 └── lcd.display          (将异常信息绘制到屏幕)

Comm                      (处理串口通信)

init_uart
 ├── fm.register          (初始化 UART 引脚)
 └── UART(...)            (设置 UART 参数)

main
 ├── Sensor 初始化         (传感器初始化及配置)
 ├── LCD 初始化            (LCD 屏幕初始化及配置)
 ├── 加载模型与标签        (加载 YOLO 模型和标签文件)
 └── 循环检测              (持续捕获图像、运行检测、绘制结果并与 UART 通信)

异常处理
 ├── 捕获异常
 ├── 显示异常信息
 └── 清理资源
```

0. MaixBit是干什么的？
- 我们在MaixBit上进行目标检测，它加载一个 YOLO 模型，通过摄像头实时捕捉图像，并对图像中的物体进行分类和定位。检测到的物体信息会通过 UART 串口发送出去。  

1. 错误信息处理部分 (此部分可以跳过)
- 主要为格式化异常信息与将异常信息绘制到屏幕，故不解析。

2. 屏幕串口通信部分
- Comm 类的作用是负责处理串口通信，将检测到的目标信息通过 UART (串口) 发送给MaixBit屏幕，不涉及控制部分，故不解析。

3. UART 初始化
- UART1的TX和RX引脚被注册到MaixPy环境，并且波特率被设置为115200;

```py
def init_uart():
    fm.register(15, fm.fpioa.UART1_TX, force=True)
    fm.register(16, fm.fpioa.UART1_RX, force=True)
    uart = UART(UART.UART1, 115200, 8, 0, 0, timeout=1000, read_buf_len=256)
    return uart
```

4. 图像识别与 UART 物品标签信息发送
- 此部分对于MaixBit在舵机控制部分最重要，但是main函数其实大部分代码是图像识别部分，我们只解析物品类别是如何发送的，以下为绘制检测结果并发送数据部分：

```py
if objects:
    for obj in objects:
        pos = obj.rect()
        img.draw_rectangle(pos)
        img.draw_string(pos[0], pos[1], "%s : %.2f" % (labels[obj.classid()], obj.value()), scale=2, color=(255, 0, 0))
    if labels[obj.classid()] == '1':
        if count == 0:
            uart.write(labels[obj.classid()])
            print(labels[obj.classid()])
            count = 1
    # 类似的判断和发送操作对标签 '2'，'3' 和 '4' 也进行。
```
- 如果检测到物体，循环遍历每个物体，绘制物体的矩形框，并在矩形框上显示物体的标签和置信度。根据检测到的物体标签，向 UART 发送对应的标签（例如：'1', '2', '3', '4'）。每次发送后，count 被设置为不同的值以确保只发送一次。

## STM32 - 数据处理部分
- `Path:` SYSTEM/usart/usart.c
- 代码逻辑图
```plaintext
uart_init
 ├── 初始化 GPIO
 │    ├── GPIOA.9 (USART1_TX 配置为复用推挽输出)
 │    └── GPIOA.10 (USART1_RX 配置为浮空输入)
 ├── NVIC_Init
 │    └── 设置 USART1 中断优先级并使能
 └── USART_Init
      ├── 设置波特率、数据位、停止位、校验位等参数
      └── 开启 USART1 接收中断

USART1_IRQHandler
 ├── 检查 RXNE 标志位
 ├── 接收数据并存入 RxDate
 ├── 设置 RxFlag=1
 └── 清除 RXNE 中断标志位

Uart_dothing
 ├── 调用 Serial_GetRxFlag 检查是否有接收数据
 ├── 根据 Serial_GetRxDate 返回的值 ('1' ~ '4')
 │    ├── LED0 状态翻转
 │    ├── 更新 OLED 显示内容
 │    ├── 调用 setAngle 控制舵机转动
 │    │    └── 根据具体序列 ('1', '2', '3', '4') 调用 setAngle 设置不同角度
 │    └── 通过 USART1 发送确认数据
 └── 如果 time_flag=1，则显示初始化状态并关闭 TIM2
 ```

1. 串口初始化部分 (此部分可以跳过)

```c    
void uart_init(u32 bound)
{
    //1. 开启 GPIO 和 USART1 时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA, ENABLE);
    
    //2. 配置 USART1 TX 引脚（GPIOA.9）
    GPIO_InitTypeDef GPIO_InitStructure;
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;               // A9 引脚
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;       // 高速 50MHz
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;         // 复用推挽输出
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    //3. 配置 USART1 RX 引脚（GPIOA.10）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;              // A10 引脚
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;   // 浮空输入模式
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    //4. 配置串口的基础信息
    USART_InitTypeDef USART_InitStructure;
    USART_InitStructure.USART_BaudRate = bound;             // 波特率
    USART_InitStructure.USART_WordLength = USART_WordLength_8b; // 字长 8 位
    USART_InitStructure.USART_StopBits = USART_StopBits_1;  // 1 个停止位
    USART_InitStructure.USART_Parity = USART_Parity_No;     // 无校验位
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None; // 无流控
    USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx; // 允许收发模式
    USART_Init(USART1, &USART_InitStructure);               // 初始化 USART1

    //5. 配置 NVIC 中断优先级，并使能中断
    NVIC_InitTypeDef NVIC_InitStructure;
    NVIC_InitStructure.NVIC_IRQChannel = USART1_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 3; // 抢占优先级
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 3;        // 子优先级
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    //6. 开启 USART 中断和串口
    USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);     // 开启接收数据中断
    USART_Cmd(USART1, ENABLE);                        // 使能串口
}
```

2. 中断服务程序 - 接收数据并设置标志位
- 在这段函数中，程序接收引脚的数据并存储到变量，即从USART_ReceiveData获取USART1的值，存储到RxDate，将RxDate封装到Serial_GetRxDate。

```c
u8 Serial_GetRxDate(void)//封装一个函数接受
{
	return	RxDate;
}
void USART1_IRQHandler(void)
{
    if (USART_GetITStatus(USART1, USART_IT_RXNE) == SET) // 判断是否收到数据
    {
        RxDate = USART_ReceiveData(USART1);             // 接收数据存储到全局变量
        state = 1;                                      // 设置状态标志位
        RxFlag = 1;                                     // 标志数据已接收
        USART_ClearITPendingBit(USART1, USART_IT_RXNE); // 清除中断标志
    }
}
```

3. 主逻辑处理函数 - 根据具体接收字符执行不同操作
- 根据Serial_GetRxFlag的值来给出相应的控制数据，我们这里以接收到数值‘1’为例，即可回收垃圾

```c
void Uart_dothing(void)
{
    if (Serial_GetRxFlag() == 1) // 判断是否接收到数据
    {
        if (Serial_GetRxDate() == '1') // 如果接收到字符 '1'
        {
            LED0 = !LED0;                          // 翻转 LED 指示灯
            OLED_Clear();                          // 清空 OLED 显示
            OLED_ShowString(1, 1, "serial:");      // OLED 显示串口输入
            OLED_ShowNum(1, 8, 1, 1);              // 数字 1
            OLED_ShowString(2, 1, "kind:");        // 显示垃圾种类
            OLED_ShowString(2, 6, "cycle");        // 类型：可回收
            OLED_ShowString(3, 1, "num:");         // 数量
            OLED_ShowNum(3, 5, 1, 1);              // 输入类别为“1”
            OLED_ShowString(4, 1, "OK");

            // 控制舵机动作
            setAngle(0, 90);                       // 通道 0 转动到 90 度
            delay_ms(1000);                        // 延时 1 秒
            setAngle(1, 120);                      // 通道 1 转动到 120 度
            delay_ms(1000);                        // 延时 1 秒
            setAngle(1, 70);                       // 通道 1 恢复到 70 度
            delay_ms(1000);

            state = 0;                             // 状态标志位清零
            TIM_Cmd(TIM2, ENABLE);                 // 启动定时器
            USART_SendData(USART1, '1');           // 回传字符 '1'（确认接收完成）
        }
        else if (Serial_GetRxDate() == '2') // 如果接收到字符 '2'
        // 继续‘2’、‘3’、‘4’的逻辑 ...
    }
    else if (time_flag == 1) // 定时器事件
    {
        time_flag = 0;                  // 清除定时器标志
        LED0 = !LED0;                   // 翻转 LED
        OLED_Clear();                   // 清空 OLED
        OLED_ShowString(1, 1, "To begin!!!"); // 显示字符串
        TIM_Cmd(TIM2, DISABLE);         // 停止定时器
    }
}
```
- 我们可以看到，当MaixBit通过窗口给STM32发送数据后，STM32根据不同的数值（垃圾类型）调用了多个舵机控制函数，并传入了多组数值，这边我们接着分析这些舵机控制函数是如何对舵机进行进一步控制的。

## PCA9685 - 舵机实际控制部分
- `Path:` BSP/bsp_PCA9685.c
- 代码逻辑关系图：
```plaintext
PCA9685_Init
 ├── IIC_Init         (初始化 I2C 通信)
 ├── PCA9685_Write    (初始化模式寄存器)
 ├── PCA9685_setFreq  (设置频率)
 └── PCA9685_setPWM   (初始化各通道 PWM 占空比)

setAngle
 ├── 将角度映射为 PWM off 值
 └── 调用 PCA9685_setPWM 设置 PWM
```

1. 什么是PCA9685，这段代码是干什么的？
- 这段代码是用于驱动 PCA9685 的驱动程序实现，PCA9685 是一个 16 通道 PWM 控制器，经常用于舵机控制、电机驱动或 LED 调光等需要 PWM 信号的场景。代码通过I²C通信来与 PCA9685 芯片交互，可以实现以下功能：  
  - 初始化 PCA9685，包括设置工作频率和舵机初始位置。
  - 设置 PWM 信号频率，用于生成不同的脉宽。
  - 配置指定通道的 PWM 占空比，实现控制。
  - 通过控制角度，调整舵机的旋转位置。

2. 初始化 PCA9685
- 这个函数用于初始化 PCA9685 驱动板，设置 PWM 工作频率和舵机的初始角度

```c
void PCA9685_Init(float hz, u8 angle)
{
    u32 off = 0;    // 默认占空比为 0
    IIC_Init();     // 初始化 I²C 通信
    PCA9685_Write(PCA_Model, 0x00);  // 清除 MODE 寄存器，准备设置
    
    // 设置 PWM 频率（通常与舵机工作频率匹配）
    PCA9685_setFreq(hz);
    
    // 根据输入角度计算脉宽值，并初始化 16 个通道
    off = (u32)(145 + angle * 2.4); 
    PCA9685_setPWM(0, 0, off);
    PCA9685_setPWM(1, 0, off);
    PCA9685_setPWM(2, 0, off);
    ...
    PCA9685_setPWM(15, 0, off);    // 全部通道设置初始占空比（舵机默认位置）
    
    delay_ms(100);  // 延时 100 毫秒
}
```
- 这段函数可能涉及到了多个机械自动化专业的名词，这里我们可以不去深究，简单的解释它们的关系和用途：
  - **占空比/脉宽值:** 决定舵机角度。
  - **通道:** 指定控制哪个舵机。
  - **寄存器:** 通过配置寄存器设置每个通道的 PWM 信号参数（频率和脉宽）。

- 更加专业的解释就不再说明了，本人也非机械专业学生，知识有限，只知道这些对于这个项目就足够了，感兴趣的话可以Google一下。

3. 初始化模式寄存器
- 向 PCA9685 的寄存器写入或者读取一个字节的数据，通过 IIC 函数完成数据的发送与接收，**此处不设计舵机具体控制，可以跳过**；
- 写入函数参数含义：
  - addr: 目标寄存器地址。
  - data: 要写入的数据。
- 写入函数流程：
  - 发送器件地址 PCA_Addr。
  - 发送寄存器地址 addr。
  - 发送数据 data。

```c
void PCA9685_Write(u8 addr,u8 data)   //向PCA写数据，adrrd地址，data数据
{
	IIC_Start();
	
	IIC_Send_Byte(PCA_Addr);
	IIC_NAck();
	
	IIC_Send_Byte(addr);
	IIC_NAck();
	
	IIC_Send_Byte(data);
	IIC_NAck();
	
	IIC_Stop();	
}

u8 PCA9685_Read(u8 addr)   //从PCA读数据
{
	u8 data;
	
	IIC_Start();
	IIC_Send_Byte(PCA_Addr);
	IIC_NAck();
	IIC_Send_Byte(addr);
	IIC_NAck();
	IIC_Stop();
	
	delay_us(10);

	IIC_Start();
	IIC_Send_Byte(PCA_Addr|0x01);
	IIC_NAck();
	data = IIC_Read_Byte(0);
	IIC_Stop();
	
	return data;
}
```
4. 设置Freq与PWM
- 此部分为PCA9685对舵机的内部底层控制的接口，并非程序的外部接口，所以跳过此部分解析。

5. 舵机转动角度控制部分
- 这部分是整个PCA9685最重要的控制部分，先来看代码，很简单，将角度转化为PCA9685可以接受的PWM数值：

```c
void setAngle(u8 num, u8 angle)
{
    u32 off = 0;
    off = (u32)(158 + angle * 2.2);  // 根据角度计算对应的 PWM 高电平结束位置
    PCA9685_setPWM(num, 0, off);    // 调用函数设置 PWM 信号
}
```

- 输入参数
  - **num**：PCA9685 输出通道号（范围 0-15），对应驱动的舵机编号，结合项目实际，num=0为水平方向的舵机，num=1为竖直方向的舵机。
  - **angle**：目标角度，单位是度（例如 0°, 90°, 180°），需要通过公式转换为 PWM 脉宽值。
- PWM 脉宽计算公式计算逻辑 (推广)
  - **158**：对应舵机的最小脉宽（约 1ms）。
  - **angle * 2.2**：将角度线性映射到 PWM 的增加值。
  - 例如：
    - 角度为 0° 时，off = 158。
    - 角度为 90° 时，off ≈ 158 + 90 * 2.2 ≈ 356。
    - 角度为 180° 时，off ≈ 158 + 180 * 2.2 ≈ 554。

## 流程总结与示例

**MaixBit部分：**  
开始 --> 识别到可回收垃圾 --> 通过UART窗口发送‘1’

**STM32部分：**  
--> 接收数据‘1’，存储到`RxDate`，方便`Serial_GetRxFlag`调用 --> 通过判断`Serial_GetRxFlag`值为‘1’（可回收垃圾）  --> 更新屏幕显示，调用`setAngle`控制舵机转动

**PCA9685部分：**  
--> 接收`setAngle`参数，控制舵机进行水平和竖直转动 --> 结束

## STM32上的垃圾桶满载检测部分
1. 先来看具体的代码实现  
`Path:` HARDWARE/threshold.c

- **Threshold_Init 函数：** 函数的目的是配置GPIOA的引脚4，也就是A04引脚，为垃圾满载检测的传感器；
```c
void Threshold_Init(void)
{
 GPIO_InitTypeDef  GPIO_InitStructure;
 	
 RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);	 //使能PC 端口时钟
	
 GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4;				 //LED0端口配置
 GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPD; 		 //推挽输出
 GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;		 //IO口速度为50MHz
 GPIO_Init(GPIOA, &GPIO_InitStructure);					 //根据设定参数初始化
}
```
- **Threshold_control 函数**
```c
void Threshold_control(void)
{
	if(state==0)
	{
		if(PAin(4)==0)
		{
			//OLED_Clear();
			OLED_ShowString(1,1,"harmful      ");
			OLED_ShowString(2,1,"fully loaded ");
			OLED_ShowString(3,1,"             ");
			OLED_ShowString(4,1,"             ");
			PBout(11)=0;
			delay_ms(500);
			PBout(11)=1;
		}
		else
		{
			OLED_ShowString(1,1,"To begin!!!  ");
			OLED_ShowString(2,1,"             ");
			OLED_ShowString(3,1,"             ");
			OLED_ShowString(4,1,"             ");
		}
	}

}
```
- 整体逻辑：
  - 如果系统状态变量 state 为 0：
    - 检查 GPIOA 引脚 4 的电平。
      - 如果是低电平：
        - 在 OLED 显示屏上显示特定的警告信息，例如 "harmful" 和 "fully loaded"。
        - 控制 GPIOB 引脚 11：先设置为低电平（激活某设备或触发某动作），延时 500 毫秒后恢复为高电平。
      - 如果不是低电平：
        - 显示提示信息 "To begin!!!"，表示准备开始。
2. 现在我们知道触发检测垃圾是否满载的条件了，即系统变量`state`的值，这个值是怎么变化的？我们来分析涉及到的`usart.c`，即我们前面分析的STM32接收信息与处理部分的代码，我们直接来分析关键部分，代码如下：

```c
······
u8 state=0;                                         // 此处定义了state的初始值
······
if (USART_GetITStatus(USART1, USART_IT_RXNE) == SET)
{
    RxDate = USART_ReceiveData(USART1);             // 读取接收到的数据
    state = 1;                                      // 设置 state 为 1
    RxFlag = 1;                                     // 设置接收完成标志为 1
    USART_ClearITPendingBit(USART1, USART_IT_RXNE); // 清除中断标志
}
······
void Uart_dothing(void)
{
	if(Serial_GetRxFlag()==1)                       // 如果接收完成标志为 1
		{
			if(Serial_GetRxDate()=='1')             // 如果垃圾类型为 1
			{ 
                ······
				setAngle(1,70);                     // 对应舵机的一系列控制
                ······
				state=0;                            // 完成舵机操作后，重新设置 state为 0
				TIM_Cmd(TIM2, ENABLE);
				USART_SendData(USART1,'1');	        // 串口返回数据 1
			}
			else if(Serial_GetRxDate()=='2')        // 其他垃圾种类逻辑一致
            ······
		}
        ······
}
```
3. 综上，state 是一个全局变量，初始值为 0，用于标识串口通信的状态或控制不同的逻辑流程。当STM32串口收到数据时，state 被设置为 1,当处理完接收到的数据后，state 被重新设置为 0。
4. 在实际使用中的逻辑就是，开机后会自动开始检测是否满载，如果识别到垃圾，停止检测，处理垃圾，然后处理完后，继续保持检测是否满载状态。