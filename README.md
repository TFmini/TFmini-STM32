# STM32_TFmini  

http://benewake.com

## Description
本文件夹为TFmini的STM32转接例程程序，使用STM32CubeMX、Keil作为开发工具。
其中：

包含2种TFmini通讯协议，分别为：PIX、Standard Data Format(89BYTE))。

包含4种接口协议：IIC、CAN、SPI、开关量。



## USART_2_DOUT

USART_2_DOUT 为 USART转开关量例程。功能如下： 当 dist > TFMINI_ACTION_DIST,  PA8 置低; 否则， PA8 置高。

其中， TFMINI_ACTION_DIST可通过main.c中的Configuration Wizard设置。

USART1用于连接TFmini，采用空闲中断方式，接收数据。

PA8用于开关量输出。

具体实现步骤如下
1.初始化USART1、PA8。
2.开启USART1的空闲中断。
3.USART2_IRQHandler增加中断判断。代码如下：

`
uint32_t tmp = 0;

  if((__HAL_UART_GET_FLAG(&huart2,UART_FLAG_IDLE) != RESET))
  {
      __HAL_UART_CLEAR_IDLEFLAG(&huart2);

      tmp = huart2.Instance->SR;
      tmp = huart2.Instance->DR;

      HAL_UART_DMAStop(&huart2);

      tmp =  USART_BUF_SIZE - hdma_usart2_rx.Instance->CNDTR;
      HAL_UART_Receive_DMA(&huart2, g_usart2_rx_buf, USART_BUF_SIZE);

      USART1_RX_Proc(g_usart2_rx_buf, tmp);
  }

/*************************************************
Function: USART1_RX_Proc
Description: USART1_RX_Proc
Input:
    buf, data buffer
    len, data len
Output: None
Return: None
Others: TFMini output dist unit = cm;
*************************************************/
void USART1_RX_Proc(uint8_t *buf, uint32_t len)
{
    uint32_t i = 0;
    uint8_t chk_cal = 0;
    uint16_t cordist = 0;

    if(TFMINI_DATA_Len == len)
    {
        if((TFMINT_DATA_HEAD == buf[0])&&(TFMINT_DATA_HEAD == buf[1]))
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
`
