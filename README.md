# I2C总线教学（基于控制温度传感器LM75A）
## ·整体概括
![I2C概括图](https://ts1.cn.mm.bing.net/th/id/R-C.8fb6b6ffc2abce3b5a489e36cb09ece1?rik=YfLDh//BGavDzA&riu=http://www.circuitbasics.com/wp-content/uploads/2016/01/Introduction-to-I2C-Data-Transmission-Diagram-Stop-Condition.png&ehk=eyuBLWTwXuKYXaO1FPbP81QG2Gjp8TNpWw1GJ5kZlbI=&risl=&pid=ImgRaw&r=0)
I2C整体分为两个部分，**第一个部分**是主设备（Master），也就是我们的单片机，**第二个部分**是从设备（Slave），主从设备之前连有3根先：SDA，SCL，GND（主从设备都共地）。从构成来看，I2C就是主、设备之间的通信。同一条I2C总线允许挂载127个设备。
## ·I2C使用特点

 1. I2C接口只用两条线：SDA,SCL
 2. 单片机在读取I2C总线的视乎，与I2C复用的I/O端口要设置为“复用开路模式”
 ```
void I2C_GPIO_Init(void){ //I2C接口初始化
	GPIO_InitTypeDef  GPIO_InitStructure; 	
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA|RCC_APB2Periph_GPIOB|RCC_APB2Periph_GPIOC,ENABLE);       
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE); //启动I2C功能 
    GPIO_InitStructure.GPIO_Pin = I2C_SCL | I2C_SDA; //选择端口号                      
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_OD; //选择IO接口工作方式       
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; //设置IO接口速度（2/10/50MHz）    
	GPIO_Init(I2CPORT, &GPIO_InitStructure);
}
```
 3. 重点：因为I2C主设备要和从设备之间进行通信，为了能够区分主、从设备，在使用I2C时都给它们分配了**器件地址**。但两者地址设置方法却略有不同，主设备地址是用户自己设定的（如本例中为0XC0），从设备地址是固定的（地址可以在自己要用的设备的数据手册中找到）。

```
#define HostAddress	0xc0	//总线主机的器件地址

#define LM75A_ADD	0x9E	//器件地址
```
## I2C程序分析
一个设备的开发分为5各部分：
**硬件** --->**I2C功能固件库**（官方给定）--->**I2C总线驱动程序**（在使用官方固件库的基础上进行编写）--->**I2C器件驱动驱动程序**（这个是基于I2C总线驱动程序中的函数结合设备所需要的程序进行编写） --->**用户应用程序**（main.c）

此处，我们来重点讲讲**I2C总线驱动程序**。不论使用哪款需要使用I2C进行通信的器件，编写器件驱动程序也都只需要这5个函数。
I2C总线驱动程序共包含五个部分：

 1. I2C_Configuration（配置I2C所需要用的串口和I2C的基本配置）
 2. I2C_SAND_BYTE（发送一个字节数据）
 3. I2C_SAND_BUFFER（发送一个数据串）
 4. I2C_READ_BYTE（接收一个字节数据）
 5. I2C_READ_BUFFER（接收数据串）
它们主要使用了I2C固件库中的I2C  ,  I2C_SendData  ,  I2C_ReceiveData这三个函数

```
void I2C_GPIO_Init(void){ //I2C接口初始化
	GPIO_InitTypeDef  GPIO_InitStructure; 	
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA|RCC_APB2Periph_GPIOB|RCC_APB2Periph_GPIOC,ENABLE);       
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE); //启动I2C功能 
    GPIO_InitStructure.GPIO_Pin = I2C_SCL | I2C_SDA; //选择端口号                      
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_OD; //选择IO接口工作方式       
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; //设置IO接口速度（2/10/50MHz）    
	GPIO_Init(I2CPORT, &GPIO_InitStructure);
}

void I2C_Configuration(void){ //I2C初始化
	I2C_InitTypeDef  I2C_InitStructure;
	I2C_GPIO_Init(); //先设置GPIO接口的状态
	I2C_InitStructure.I2C_Mode = I2C_Mode_I2C;//设置为I2C模式
	I2C_InitStructure.I2C_DutyCycle = I2C_DutyCycle_2;
	I2C_InitStructure.I2C_OwnAddress1 = HostAddress; //主机地址（从机不得用此地址）
	I2C_InitStructure.I2C_Ack = I2C_Ack_Enable;//允许应答
	I2C_InitStructure.I2C_AcknowledgedAddress = I2C_AcknowledgedAddress_7bit; //7位地址模式
	I2C_InitStructure.I2C_ClockSpeed = BusSpeed; //总线速度设置 	
	I2C_Init(I2C1,&I2C_InitStructure);
	I2C_Cmd(I2C1,ENABLE);//开启I2C					
}

void I2C_SAND_BUFFER(u8 SlaveAddr,u8 WriteAddr,u8* pBuffer,u16 NumByteToWrite){ //I2C发送数据串（器件地址，寄存器，内部地址，数量）
	I2C_GenerateSTART(I2C1,ENABLE);//产生起始位
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT)); //清除EV5
	I2C_Send7bitAddress(I2C1,SlaveAddr,I2C_Direction_Transmitter);//发送器件地址
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));//清除EV6
	I2C_SendData(I2C1,WriteAddr); //内部功能地址
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED));//移位寄存器非空，数据寄存器已空，产生EV8，发送数据到DR既清除该事件
	while(NumByteToWrite--){ //循环发送数据	
		I2C_SendData(I2C1,*pBuffer); //发送数据
		pBuffer++; //数据指针移位
		while (!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED));//清除EV8
	}
	I2C_GenerateSTOP(I2C1,ENABLE);//产生停止信号
}
void I2C_SAND_BYTE(u8 SlaveAddr,u8 writeAddr,u8 pBuffer){ //I2C发送一个字节（从地址，内部地址，内容）
	I2C_GenerateSTART(I2C1,ENABLE); //发送开始信号
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT)); //等待完成	
	I2C_Send7bitAddress(I2C1,SlaveAddr, I2C_Direction_Transmitter); //发送从器件地址及状态（写入）
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED)); //等待完成	
	I2C_SendData(I2C1,writeAddr); //发送从器件内部寄存器地址
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED)); //等待完成	
	I2C_SendData(I2C1,pBuffer); //发送要写入的内容
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED)); //等待完成	
	I2C_GenerateSTOP(I2C1,ENABLE); //发送结束信号
}
void I2C_READ_BUFFER(u8 SlaveAddr,u8 readAddr,u8* pBuffer,u16 NumByteToRead){ //I2C读取数据串（器件地址，寄存器，内部地址，数量）
	while(I2C_GetFlagStatus(I2C1,I2C_FLAG_BUSY));
	I2C_GenerateSTART(I2C1,ENABLE);//开启信号
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT));	//清除 EV5
	I2C_Send7bitAddress(I2C1,SlaveAddr, I2C_Direction_Transmitter); //写入器件地址
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));//清除 EV6
	I2C_Cmd(I2C1,ENABLE);
	I2C_SendData(I2C1,readAddr); //发送读的地址
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED)); //清除 EV8
	I2C_GenerateSTART(I2C1,ENABLE); //开启信号
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT)); //清除 EV5
	I2C_Send7bitAddress(I2C1,SlaveAddr,I2C_Direction_Receiver); //将器件地址传出，主机为读
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED)); //清除EV6
	while(NumByteToRead){
		if(NumByteToRead == 1){ //只剩下最后一个数据时进入 if 语句
			I2C_AcknowledgeConfig(I2C1,DISABLE); //最后有一个数据时关闭应答位
			I2C_GenerateSTOP(I2C1,ENABLE);	//最后一个数据时使能停止位
		}
		if(I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_RECEIVED)){ //读取数据
			*pBuffer = I2C_ReceiveData(I2C1);//调用库函数将数据取出到 pBuffer
			pBuffer++; //指针移位
			NumByteToRead--; //字节数减 1 
		}
	}
	I2C_AcknowledgeConfig(I2C1,ENABLE);
}
u8 I2C_READ_BYTE(u8 SlaveAddr,u8 readAddr){ //I2C读取一个字节
	u8 a;
	while(I2C_GetFlagStatus(I2C1,I2C_FLAG_BUSY));
	I2C_GenerateSTART(I2C1,ENABLE);
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT));
	I2C_Send7bitAddress(I2C1,SlaveAddr, I2C_Direction_Transmitter); 
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));
	I2C_Cmd(I2C1,ENABLE);
	I2C_SendData(I2C1,readAddr);
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED));
	I2C_GenerateSTART(I2C1,ENABLE);
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT));
	I2C_Send7bitAddress(I2C1,SlaveAddr, I2C_Direction_Receiver);
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED));
	I2C_AcknowledgeConfig(I2C1,DISABLE); //最后有一个数据时关闭应答位
	I2C_GenerateSTOP(I2C1,ENABLE);	//最后一个数据时使能停止位
	a = I2C_ReceiveData(I2C1);
	return a;
}
```

接下来，我们来看看LM75A是怎么使用I2C进行通信的。

```
#include "lm75a.h"



//读出LM75A的温度值（-55~125摄氏度）
//温度正负号（0正1负），温度整数，温度小数（点后2位）依次放入*Tempbuffer（十进制）
void LM75A_GetTemp(u8 *Tempbuffer){   
    u8 buf[2]; //温度值储存   
    u8 t=0,a=0;   
    I2C_READ_BUFFER(LM75A_ADD,0x00,buf,2); //读出温度值（器件地址，子地址，数据储存器，字节数）
	t = buf[0]; //处理温度整数部分，0~125度
	*Tempbuffer = 0; //温度值为正值
	if(t & 0x80){ //判断温度是否是负（MSB表示温度符号）
		*Tempbuffer = 1; //温度值为负值
		t = ~t; t++; //计算补码（原码取反后加1）
	}
	if(t & 0x01){ a=a+1; } //从高到低按位加入温度积加值（0~125）
	if(t & 0x02){ a=a+2; }
	if(t & 0x04){ a=a+4; }
	if(t & 0x08){ a=a+8; }
	if(t & 0x10){ a=a+16; }
	if(t & 0x20){ a=a+32; }
	if(t & 0x40){ a=a+64; }
	Tempbuffer++;
	*Tempbuffer = a;
	a = 0;
	t = buf[1]; //处理小数部分，取0.125精度的前2位（12、25、37、50、62、75、87）
	if(t & 0x20){ a=a+12; }
	if(t & 0x40){ a=a+25; }
	if(t & 0x80){ a=a+50; }
	Tempbuffer++;
	*Tempbuffer = a;   
}

//LM75进入掉电模式，再次调用LM75A_GetTemp();即可正常工作
//建议只在需要低功耗情况下使用
void LM75A_POWERDOWN(void){// 
    I2C_SAND_BYTE(LM75A_ADD,0x01,1); //
}
```
