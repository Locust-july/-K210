# K210
亚博k210模块与STM32通信测试,使用模块(K210,stm32f103c8t6最小系统板)：

![模块](https://github.com/Locust-july/K210/blob/main/5491e66e0dda44859d9196735db9c15b.png)

## 硬件连接
硬件连接：K210的RX,TX分别与STM32的串口1(也即是PA9,PA10)连接，CH340连接STM32的串口2(也即是PA3,PA2)
连接目的：
1. PC端通过串口助手发送数据给STM32，STM32再把数据传给K210。
2. K210发送数据给STM32，STM32通过串口发送给串口助手显示出来。

K210端代码:(使用平台为**CANMV**)
```python
from fpioa_manager import fm
from machine import UART
import time
import os
from board import board_info
from fpioa_manager import fm
import ubinascii
from machine import UART
from machine import Timer

binding UART2 IO:6->RX, 8->TX
fm.register(6, fm.fpioa.UART2_RX)
fm.register(8, fm.fpioa.UART2_TX)

yb_uart = UART(UART.UART2, 115200, 8, 0, 0, timeout=1000, read_buf_len=4096)

write_bytes = b'$hello yahboom#\n' #模拟串口发送
last_time = time.ticks_ms()

try:
    while True:
        time.sleep_ms(1000)
        if time.ticks_ms() - last_time > 1000:  #验证发送，每1s发送一次
            last_time = time.ticks_ms()
            print("write_bytes = ", write_bytes) 
            yb_uart.write("Hello STM32\r\n")  
        #串口接收部分
        if yb_uart.any():      
            read_data = yb_uart.read()
            if read_data:   #接收到数据，打印并通过串口发送
                  print("read_data = ", read_data)
                  yb_uart.write(read_data)
except:
    pass

yb_uart.deinit()
del yb_uart
```
STM32端(main.c):
```C
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "Serial.h"

uint8_t RxData;
uint8_t RxData1;

int main(void)
{
	Serial_init();     //与K210串口连接
	Serial_init2();   //与PC端通过CH340连接,验证串口通信是否正常
	Serial_printf("测试串口通信\r\n");
	while (1)
	{
        /*串口助手发送的数据STM32通过串口2接收到，再通过串口1发送给K210.
        K210发送数据，STM32通过串口1接收到，再通过串口2发送给串口助手显示*/
		if (Serial_GetRXFlag2() == 1)  //串口2接收
		{
			RxData = Serial_GetRXData2();
			Serial_SendByte(RxData);  //数据通过串口1发送出去
		}
		else if (Serial_GetRXFlag() == 1)//串口1接收
		{
			RxData1 = Serial_GetRXData();  
			Serial_SendByte2(RxData1);  //数据通过串口2发送
		}
	}
}
```
演示结果：
①每隔1s终端以及串口助手收到消息并显示
![图片](https://img-blog.csdnimg.cn/direct/527d682129044e0282eaf44e816ceeb2.png)
②通过串口助手发送123，K210收到数据并显示
![图片]([https://img-blog.csdnimg.cn/direct/527d682129044e0282eaf44e816ceeb2.png](https://img-blog.csdnimg.cn/direct/78cb8333681148c5987ba890b20d6e72.png))

需要注意的点：
这里如果想要指定K210接收的数据，判断条件不能直接用==（原因参考亚博官网关于k210通信协议的规定）
示例：
![图片](​https://img-blog.csdnimg.cn/direct/14583012909f439d87bd1556c6b00364.png)
