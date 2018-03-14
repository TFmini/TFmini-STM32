# STM32_TFmini  

English version [README.md](/README.md)  


## Description
本文件夹为TFmini的STM32转接例程程序，使用STM32CubeMX、Keil作为开发工具。
其中：

包含2种TFmini通讯协议，分别为：PIX、Standard Data Format(89BYTE))。

包含3种转换方式：开关量、CAN、IIC。

## TFmini连线
各例程均使用USART1作为TFmini的通讯端口，接线如下： 
 
TFmini  | 开发板   
--------------|----------- 
红色线(+5V) | +5V  
绿色线(TX) | PA10(RX) 
黑色线(GND) | GND  

## TFmini通讯代码
各例程均使用 USART1用于连接TFmini，采用**空闲中断方式**，接收数据。

**具体实现步骤如下:**

**1.初始化USART1、PA8。**

**2.开启USART1的空闲中断。** 

 ` HAL_UART_Receive_DMA(&huart1, g_usart1_rx_buf, USART_BUF_SIZE);`

 ` __HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);    //UART_IT_IDLE`

**3.USART2_IRQHandler增加中断判断。** 代码如下：

`uint32_t tmp = 0;`

  `if((__HAL_UART_GET_FLAG(&huart2,UART_FLAG_IDLE) != RESET))`
  
  {

     __HAL_UART_CLEAR_IDLEFLAG(&huart2);

      tmp = huart2.Instance->SR;
      tmp = huart2.Instance->DR;

      HAL_UART_DMAStop(&huart2);

      tmp =  USART_BUF_SIZE - hdma_usart2_rx.Instance->CNDTR;
      HAL_UART_Receive_DMA(&huart2, g_usart2_rx_buf, USART_BUF_SIZE);

      USART1_RX_Proc(g_usart2_rx_buf, tmp);
  }

**4.中断处理函数，用于接收TFMINI数据。** 可支持2种TFmini协议，只需使用不同的USART1_RX_Proc函数，可以自行集成。


## USART_2_DOUT

USART_2_DOUT 为 USART转开关量例程。功能如下：

当 dist > TFMINI_ACTION_DIST,  PA8 置低; 否则， PA8 置高。

其中， TFMINI_ACTION_DIST可通过main.c中的Configuration Wizard设置。

PA8用于开关量输出。

- Standard Data Format(89BYTE)协议 处理函数如下：

`#define TFMINI_DATA_Len             9`

`#define TFMINT_DATA_HEAD            0x59`


`void USART1_RX_Proc(uint8_t *buf, uint32_t len)`

{

    uint32_t i = 0;
    uint8_t chk_cal = 0;
    uint16_t cordist = 0;

    if(TFMINI_DATA_Len == len)
    {
        if((TFMINT_DATA_HEAD == buf[0]) && (TFMINT_DATA_HEAD == buf[1]))
        {
            for(i = 0; i < (TFMINI_DATA_Len - 1); i++)
            {
                chk_cal += buf[i];
            }

            if(chk_cal == buf[TFMINI_DATA_Len - 1])
            {
                cordist = buf[2] | (buf[3] << 8);

                /*cordist > TFMINI_ACTION_DIST cm, PA8 set Low;
                  cordist <= TFMINI_ACTION_DIST cm, PA8 set High.*/
                if(HAL_GPIO_ReadPin(DOUT_GPIO_Port, DOUT_Pin) != GPIO_PIN_RESET)
                {
                    if(cordist > TFMINI_ACTION_DIST)
                    {
                        HAL_GPIO_WritePin(DOUT_GPIO_Port, DOUT_Pin, GPIO_PIN_RESET);
                    }
                }
                else
                {
                    if(cordist <= TFMINI_ACTION_DIST)
                    {
                        HAL_GPIO_WritePin(DOUT_GPIO_Port, DOUT_Pin, GPIO_PIN_SET);
                    }
                }
            }
        }
    }
}

- PIX协议 处理函数如下：

`#define TFMINI_PIX_FLAG_END         0x0D0A     /*\r\n*/`

`#define TFMINI_PIX_FLAG_NEG         0x2D       /* - */`

`#define TFMINI_PIX_FLAG_DECPIONT    0x2E       /* . */`

`#define ASCII_0                     0x30       /*ASCII:0*/`

`void USART1_RX_Proc(uint8_t *buf, uint32_t len)`

{

    uint16_t cordist = 0;

    /*xxx.xx\r\n*/
    if((TFMINI_PIX_FLAG_END == (buf[len - 1] | (buf[len - 2] << 8))) \
        && (TFMINI_PIX_FLAG_DECPIONT == buf[len - 5]))
    {

        if(buf[0] == TFMINI_PIX_FLAG_NEG)   /*Negative, the amplitude value too low.*/
        {
            cordist = 1200;
        }
        else
        {
            cordist = ((buf[len - 6] - ASCII_0) * 100) + ((buf[len - 4] - ASCII_0) * 10) + (buf[len - 3] - ASCII_0);
            cordist+= (len == 7) ? ((buf[len - 7] - ASCII_0) * 1000) : 0;
        }

        /*cordist > TFMINI_ACTION_DIST cm, PA8 set Low;
          cordist <= TFMINI_ACTION_DIST cm, PA8 set High.*/
        if(HAL_GPIO_ReadPin(DOUT_GPIO_Port, DOUT_Pin) != GPIO_PIN_RESET)
        {
            if(cordist > TFMINI_ACTION_DIST)
            {
                HAL_GPIO_WritePin(DOUT_GPIO_Port, DOUT_Pin, GPIO_PIN_RESET);
            }
        }
        else
        {
            if(cordist <= TFMINI_ACTION_DIST)
            {
                HAL_GPIO_WritePin(DOUT_GPIO_Port, DOUT_Pin, GPIO_PIN_SET);
            }
        }
    }
}
