+++
title = "ARM-learning"
date = 2022-04-05T12:13:31+08:00
author = "hansomxue99"
keywords = ["ARM", "STM32"]
cover = ""
summary = "参加蓝桥杯嵌入式的驱动代码总结"

+++

### 之前参加嵌入式比赛，总结了一些模块的代码书写思路。
#### 1. LED
1.1 LED控制函数：led是有PC8-15和PD2引脚控制亮灭，为了在控制led时不每次都调用 `HAL_GPIO_WritePin` 函数，可以将led控制程序进行封装，如下：
```c
#define led1 GPIO_PIN_8
#define led2 GPIO_PIN_9
#define led3 GPIO_PIN_10
#define led4 GPIO_PIN_11
#define led5 GPIO_PIN_12
#define led6 GPIO_PIN_13
#define led7 GPIO_PIN_14
#define led8 GPIO_PIN_15
#define led_all GPIO_PIN_All

void led_control(u16 led, u8 mode)
{
	if(mode)
	{
		HAL_GPIO_WritePin(GPIOC, led, GPIO_PIN_RESET);
		HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_SET);
		HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_RESET);
	}
	else
	{
		HAL_GPIO_WritePin(GPIOC, led, GPIO_PIN_SET);
		HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_SET);
		HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_RESET);
	}
}
```
1.2 LED与LCD引脚冲突：由于lcd和led占用相同的引脚，时常lcd的显示会将led显示“乱码”，尽管led配置了锁存器使能。虽然对lcd中的写函数进行类似出栈入栈的处理可以在很大程度上解决这类问题，但是最近在控制led1翻转时发现仍然存在led乱码问题。此时的解决方法是，先将所有灯熄灭，然后根据标志位进行判断翻转。代码如下：
```c
if(led_state)
{
    led_control(led_all, 0);
    led_control(led1, led_flag);
    led_flag = !led_flag;
}
```
#### 2. 按键KEY
2.1 按键读取：独立按键B1-B4分别对应端口PB0-2、PA0。端口处接有上拉电阻，无消抖电容。由于硬件无消抖，通过软件延时进行消抖。基本代码如下：
```c
#define u8 unsigned char
#define B1 HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_0)
#define B2 HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_1)
#define B3 HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_2)
#define B4 HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0)

u8 key_scan(void)
{
	static u8 key_up = 1;
	if(key_up && (B1==0 || B2==0 || B3==0 || B4==0))
	{
		HAL_Delay(10);		//延时消抖
		key_up = 0;
		if(B1==0)	return 1;
		else if(B2 == 0)	return 2;
		else if(B3 == 0)	return 3;
		else if(B4 == 0)	return 4;
	}
	else if(B1 && B2 && B3 && B4)
	{
		key_up = 1;
	}
	return 0;
}
```
2.2 按键操作：所谓按键操作，即根据读取的按键值，对相应的按键加入后续的操作。代码如下：
```c
#define u8 unsigned char

void key_proc(u8 key_re)
{
	switch(key_re)
	{
		case 1:
			//add your code
			break;
		case 2:
			//add your code
			break;
		case 3:
			//add your code
			break;
		case 4:
			//add your code
			break;
		default:
            //skip or add your code
			break;
	}
}
```
2.3 按键的长短按
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
	if(htim->Instance == htim2.Instance)
	{
		if(B4==0)
			key_flag++;
		else
			key_flag = 0;
		if(key_flag >= 800)
			key_up = 1;
	}
}
```
#### 3. 定时器
3.1 定时器修改重载值ARR
```c
//操作寄存器
TIM2->ARR = 1000-1；
```

3.2 定时器出处PWM波形
```c
//初始化
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_2);

//设置比较值
TIM2->CCR2 = 800;
//或者
__HAL_TIM_SetCompare(&htim2, TIM_CHANNEL_2, 800);


//注意！！！
//对寄存器操作完成后必须进行更新操作
TIM2->EGR = TIM_EGR_UG;
```
#### 4. UART串口
4.1 串口发送
```c
void us_SendString(u8 *tx_buf)
{
	while(*tx_buf)
	{
		HAL_UART_Transmit(&huart1, tx_buf, 1, 50);
		tx_buf++;
	}
}
```

4.2 串口接收
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
	if(huart->Instance == USART1)
	{
		if(rxdat == '?' || rxlen == 4)
		{
			rxbuf[rxlen++] = rxdat;
			rx_done_flag = 1;
			__HAL_UART_DISABLE_IT(huart,UART_IT_RXNE);
		}
		else 
		{
				rxbuf[rxlen++] = rxdat;
				HAL_UART_Receive_IT(&huart1, &rxdat, 1);
		}
	}
}
```
其中，串口收发最为关键的便是“协议”， 即怎么让串口或者“对方”知道这一帧数据已经发送完成，简单通信双方会约定一个结束符，比如“$”、“\n”等等。
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
	if(huart->Instance == USART1)
	{
		if(rxdat == '?' || rxlen == 4)
		{
			rxbuf[rxlen++] = rxdat;
			rx_done_flag = 1;
		}
		else 
		{
				rxbuf[rxlen++] = rxdat;
				HAL_UART_Receive_IT(&huart1, &rxdat, 1);
		}
	}
}

if(rx_done_flag)
{
    rx_done_flag = 0;
    rxlen = 0;
    us_SendString(rxbuf);
    HAL_UART_Receive_IT(&huart1, &rxdat, 1);
}
```
#### 5. ADC
开发板中的ADC的时钟源一般设为80MHz，ADC的转换位数位12bit。读取电压值，常用的ADC采集方式为：单通道、单次读取、软件触发，即如下的方式：
```c
#define u8 unsigned char
float get_vol(void)
{
	float usvol = 0.0f;
	HAL_ADC_Start(&hadc2);		//软件启动ADC
	HAL_ADC_PollForConversion(&hadc2, 50);		//等待转换完成
	usvol = HAL_ADC_GetValue(&hadc2)/4096.0*3.3;	//将读取的数字量0~4096转为对应电压值
	return usvol;
}

```
在进行ADC采样之前，常常需要对ADC进行校准，即 `HAL_ADCEx_Calibration_Start(&hadc2, ADC_SINGLE_ENDED);`
#### 6. EEPROM
6.1 通信方式——IIC
通用输出输入端口GPIO--PB6、7模拟iic时序，底层代码如下：
```c
void I2CStart(void);
void I2CStop(void);
unsigned char I2CWaitAck(void);
void I2CSendAck(void);
void I2CSendNotAck(void);
void I2CSendByte(unsigned char cSendByte);
unsigned char I2CReceiveByte(void);
void I2CInit(void);
```
6.2 eeprom单字节写：eeprom写地址为0xa0，实际代码如下：
```c
#define u8 unsigned char
void e2prom_write(u8 addr, u8 dat)
{
    	//发送写命令，告诉从机准备写入
	I2CStart();
	I2CSendByte(0xa0);
	I2CWaitAck();
	
    	//发送要写的地址
	I2CSendByte(addr);
	I2CWaitAck();
	
    	//发送要写的数据
	I2CSendByte(dat);
	I2CWaitAck();
	
    	//写入结束
	I2CStop();
}
```
6.3 eeprom单字节读：eeprom读地址为0xa1，实际代码如下：
```c
#define u8 unsigned char
u8 e2prom_read(u8 addr)
{
	u8 temp;
	
    	//发送写命令，告诉从机将要操作的地址
	I2CStart();
	I2CSendByte(0xa0);
	I2CWaitAck();
	I2CSendByte(addr);
	I2CWaitAck();
	
    	//发送读命令，准备读取数据
	I2CStart();
	I2CSendByte(0xa1);
	I2CWaitAck();
	
   	//读取数据
	temp = I2CReceiveByte();
	I2CWaitAck();
    
    	//读操作完成
	I2CStop();
	
	return temp;
}
```