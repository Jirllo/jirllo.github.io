---
layout: post
title: "STMCubeIDE USE_HAL_***_REGISTER_CALLBACKS"
date: "2022-01-21 20:51:53 +0100"
categories: [Tutorials, STMCubeIDE]
tags: [STMCubeIDE, STM32, HAL, _REGISTER_CALLBACKS, peripheral]
math: false
mermaid: false
image:
  src: /assets/images/stm32cubeIDE.png
  width: 160
  height: 100
---
# How to using register callback in STM32 HAL
The simplest form of interrupt handling in the STM HAL library is to override __weak HAL interrupt handler function. (I assume that ```HAL_***_IRQHandler(...)``` means HAL drivers interrupt handler function with attribute __week, e.g. ```__weak void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)``` in stm32fxx_hal_tim.c file)
```c
// Example of handle output compare 1 event interrupt on timer htim1
void HAL_TIM_OC_DelayElapsedCallback(TIM_HandleTypeDef *htim)
{
  // check which timers
  if(htim == &htim1)
    // check which channel
    if(htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
    {
        //some code until output compare 1 on htim1 (TIM1) events occur
    }

  // there can be implement more timers interrupt service
}
```
This function cannot be written multiple times in this same types of peripheral. Therefore using this involved with two main problems
* is needed to check what triggered this callback: which peripheral
* must be sure that not implement ```HAL_***_IRQHandler(...)``` twice. This induced additional work when includes library using peripheral because functions ```HAL_***_IRQHandler(...)``` must be merged. Furthermore this makes reading the code difficult, because most often the interrupt service is located in a file other than the analyzed code.

Another option is abandon the use of HAL library to handle interrupt. Then interrupt must be handler by indirect from the peripheral interrupt vector. Its includes check and reset proper register. Example:
```c
//Example of handle TIM1 capture compare interrupt
void TIM1_CC_IRQHandler(void)
{
  // check Capture/compare 1 flag in Status Register
  if (TIM1->SR &TIM_SR_CC1IT == TIM_SR_CC1IF)
  {
    // check and reset Capture/compare 1 interrupt enable flag
  	if (TIM1->DIER & TIM_DIER_CC1IE == TIM_DIER_CC1IE)
  	{
  	  TIM1->DIER &= ~TIM_DIER_CC1IE;

      /* some code until TIM1 capture/compare 1 event occur */
  	}
  }
}
```
This enough good alternative because STM's MCU have overload peripheral interrupt vector. For example on STM32F756 have overload TIM2 global and TIM8 break event interrupt vector:
```c
//Example implementation of using simultaneously TIM12 global interrupt and TIM8 break event interrupt
// using  HAL macro and pure register code
void TIM8_BRK_TIM12_IRQHandler(void)
{
  if (__HAL_TIM_GET_FLAG(&htim8, TIM_FLAG_BREAK) != RESET)
  {
    if (__HAL_TIM_GET_IT_SOURCE(&htim8, TIM_IT_BREAK) != RESET)
    {
      __HAL_TIM_CLEAR_IT(&htim8, TIM_IT_BREAK);

      /* some code until TIM8 break events occur */
    }
  }
  if (TIM12->SR & TIM_SR_UIF == TIM_SR_UIF)
  {
  	if (TIM12->DIER & TIM_DIER_UIE == TIM_DIER_UIE)
  	{
  	  TIM12->DIER &= ~TIM_DIER_UIE;

      /* some code until TIM12 update events occur */
  	}
  }
}
```
Fortunately HAL library provides another option that is registered callback functions for each peripheral separately. First needed is change directive ```#define USE_HAL_TIM_REGISTER_CALLBACKS``` values from ```0U``` to ```1U```, then HAL drivers library allows use registered callback. It is best done in the STM32CubeMX code generator to avoid overwrite this until regenerate code. [STM32CubeMX RegisterCallback](/assets/Post_1/stm32mx.png)
(if you don't have this window install latest version of STM32CubeMX). First is needed to registered yours callback function using ```HAL_TIM_RegisterCallback()```(If doesnt register function dont worry, it will used standard function handler ```HAL_***_IRQHandler(...)```) Then using the peripheral look like:

```c
// Assume that timer 1 have configured interrupt until output compare 1 event, and callback register function is myCallback
void myCallback(TIM_HandleTypeDef *htim)
{
  // require only to check active channel
  if(htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
  {
      //some code until output compare 1 on htim1 (TIM1) events occur
  }
}
// example of initial TIM1, register callback function and start  timer with interrupt
void init_start_tim1()
{
  MX_USART3_UART_Init();
  HAL_StatusTypeDef status;
  status = HAL_TIM_RegisterCallback(
    &htim1,
    HAL_TIM_OC_DELAY_ELAPSED_CB_ID,
    myCallback
  );
  if(status == HAL_OK)
    HAL_TIM_OC_Start_IT(&htim1, TIM_CHANNEL_1);
}
```
# Sumary
Use register callback have some benefits and disadvantages:
* This is associated with a larger memory footprint, but its reduced the number of checked conditions when handling interrupts.  
* Using register_callbacks on STM32 HAL peripheral implementation could reduce development time and enhance readability of code.
