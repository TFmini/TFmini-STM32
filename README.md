# TFmini-STM32
TFmini's Example for STM32 Pi. 中文版参考 [README_CN.md](/README_CN.md).<br>  

## Description

This repository contains sample codes for TFmini-STM32 connection, using STM32Cube MX and Keil as development tools. It includes:

2 kinds of TFmini communication protocol: PIX and Standard Data Format(89BYTE).

3 kinds of output: switching value, CAN and I2C.

## TFmini Connection

Sample codes use USART1 as communication port to connect to TFmini. Connection is as follow:  
 
TFmini  | Board   
--------------|----------- 
Red cable(+5V) | +5V  
Green cable(TX) | PA10(RX) 
Black cable(GND) | GND  

## TFmini Communication Code

All the samples uses USART1 to connect to TFmini, and receives data using the **idle interrupt**. 

**Steps:**

**1.Initialize USART1, PA8.**

**2.Enable USART1 Idle Interrupt.** 

 ` HAL_UART_Receive_DMA(&huart1, g_usart1_rx_buf, USART_BUF_SIZE);`

 ` __HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);    //UART_IT_IDLE`

**3.Add Interrupt Checking to USART2_IRQHandler.** Code as follow:

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

**4.Add Interrupt Processing Function to Receive TFmini's Data.** It only needs to use different USART1_RX_Proc to support 2 kinds of TFmini communication Protocol, and users can integrate it by themselves.


## USART_2_DOUT

USART_2_DOUT is an USART-switching value sample. It does the work:

When dist > TFMINI_ACTION_DIST, PA8 is set to low level; otherwise, PA8 is set to high level.

The value of TFMINI_ACTION_DIST can be configured via Configuration Wizard of main.c.

PA8 is used to output switching value.

- Standard Data Format(89BYTE) protocol processing function:

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

- PIX protocol processing function:

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
