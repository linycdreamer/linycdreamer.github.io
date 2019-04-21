---
title: 电机控制（一） stm32实现svpwm
date: 2019-03-11 18:01:06
tags: 电机控制
---

### 一、简介

FOC电机控制算法相比于方波控制能提高电机的运转性能，但是目前的困境在于大量的博客论坛都是理论上的泛泛而谈，只是谈到一些基本的FOC的原理没有落地到具体实现。另一方面，st公司开源了一套FOC的代码库，奈何为了推行F3系列芯片大部分都是基于stm32f3实现的，同时包含了大量的宏定义和GUI内容，把各种模式和配置都杂糅在一起，在我看来是不利于大家理解的并且从2.0的库到5.0的库封装性越好，越难以深入底层了解算法的本质。

所以我依托于st开源的2.0的库，深入的讲解如何用stm32f103来实现一整套无感FOC算法。
<!-- more -->
### 二、svpwm简介

svpwm是foc的正弦波调制的一种方式具体介绍很长，不是本文重点。请移步下面博客

https://blog.csdn.net/qlexcel/article/details/74787619

我们只需要知道svpwm波的波形如下图，而最重要的任务就是如何用stm32实现

 ![](assets\img\FOC-svpwm.jpg)

### 三、stm32 定时器配置

由于我采用的是三电阻采样的方式，在电机库的stm32x_svpwm_3shunt.c 文件中实现了该模式的具体配置。初始化定时器的代码如下

```TIM_DeInit(TIM1);
  TIM_TimeBaseStructInit(&TIM1_TimeBaseStructure);
  /* Time Base configuration */
  TIM1_TimeBaseStructure.TIM_Prescaler = 0x0;
  TIM1_TimeBaseStructure.TIM_CounterMode =      TIM_CounterMode_CenterAligned1;
  TIM1_TimeBaseStructure.TIM_Period = PWM_PERIOD;
  TIM1_TimeBaseStructure.TIM_ClockDivision = TIM_CKD_DIV2;
  // Initial condition is REP=0 to set the UPDATE only on the underflow
  TIM1_TimeBaseStructure.TIM_RepetitionCounter = REP_RATE;
  TIM_TimeBaseInit(TIM1, &TIM1_TimeBaseStructure);
```

配置方面需要对stm32的定时器有初步的了解,下面对每个参数进行介绍

**TIM_Prescaler : ** 时钟的分频系数，对于stm32 f103 的定时器Tim1时钟为72MHz，这里我们为了提高控制精度不分频则系数取（0+1），所以寄存器CNT每1/72M s加一。

**TIM_CounterMode： **定时器的模式也是配置的重点，不同于其他场合这里我们选择中心对齐模式（此外还有很多其他模式请参考官方参考手册），在中心模式下CNT寄存器的值从0一直递增直到等于ARR的数值从而产生上溢事件，然后从ARR开始递减，一直减到0产生下溢事件。具体如图

![](assets\img\TIM1中心对齐.png)

**PWM_PERIOD： **设置寄存器ARR的值，即CNT累加到PWM_PERIOD便会产生上溢事件，这里需要考虑到电机控制的频率问题，如果取的太低则会导致电机发出噪声。这里我取1800 - 1，也就是说频率为72M/1800 =  40 000HZ，同时由于中心对齐的特性两次溢出才是一个周期所以实际频率为20 000 Hz，T = 0.05ms

**TIM_CKD_DIV2：**分频系数，和TIM_Prescaler 不同，TIM_CKD_DIV2指的是定时器的一些高级功能的时钟分频如死区和刹车等事件处理的时钟，影响不大。

REP_RATE：重复计数值，对于中心模式有上溢事件和下溢事件，但是后面会说到我们只需要在上溢时才进行一些操作。将REP_RATE设置为1，意味着REP_RATE+1次 （上溢+下溢）事件发生后才会进入中断，减少了判断与打断次数。

紧接着我们配置输出，对于GPIO我们选择复用推挽便好

 `GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;  `

定时器的输出配置同样有一些需要注意的地方

```
  TIM1_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1; 
  TIM1_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable; 
  TIM1_OCInitStructure.TIM_OutputNState = TIM_OutputNState_Enable;                  
  TIM1_OCInitStructure.TIM_Pulse = 0x505; //dummy value
  TIM1_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High; 
  TIM1_OCInitStructure.TIM_OCNPolarity = TIM_OCNPolarity_High;         
  TIM1_OCInitStructure.TIM_OCIdleState = TIM_OCIdleState_Reset;
  TIM1_OCInitStructure.TIM_OCNIdleState = LOW_SIDE_POLARITY;          
  
  TIM_OC1Init(TIM1, &TIM1_OCInitStructure); 
  TIM_OC2Init(TIM1, &TIM1_OCInitStructure);
  TIM_OC3Init(TIM1, &TIM1_OCInitStructure);
```

同样对每个参数都进行介绍

**TIM_OCMode： ** 有TIM_OCMode_PWM1和TIM_OCMode_PWM2两种模式可以选择，区别在于PWM1模式下当CNT寄存器小于CCR寄存器时状态为1，大于时则为0，而PWM2反之。如下图的效果

![](assets\img\PWM1VSPWM2.png)

PWM1和2的区别在于cnt小于Pulse时PWM1为高而pwm2为低，造成PWM1模式下高电平时产生跟新事件，而PWM2下低电平时产生跟新事件，跟新事件时进入中断可以开启adc采样，根据采样要求选择高电平时即PWM1模式		 

**TIM_OutputState/TIM_OutputNState：**使能CHx和CHNx输出，如果采用6个MOS驱动需要互补输出，如果直接采用集成芯片则不需要互补输出，芯片自带互补功能。

**TIM_OCPolarity/TIM_OCNPolarity：**  这两个配置和pwm1与pwm2模式共同决定最终的波形，如果极性设置为高则1==高电平，反之1==低电平

如下图效果，共同决定输出波形

![]()

**TIM_Pulse： ** 设置CCRx寄存器的值，但是我们后期需要不断改变这个值达到改变波形的目的，这里只是象征性的赋值

**TIM_OCIdleState/TIM_OCNIdleState：**  决定死区时CHx和CHNx的输出状态，这里推荐设置为CHx输出低，CHNx输出高，但是反过来理论上也可以。但是这两个一定要相反配置。



定时器部分最后一个配置便是死区配置，相信对有刷电机熟悉的同学对死区这个概念应该不陌生了，不了解的请自行百度。总之死区的目的在于避免上下MOS同时导通造成短路的现象，这也提醒我们一定要测好波形再上电上电机也一定要使用数字电源调试，盲目上电只会炸板子！！！不要问我是怎么知道的。

```
    TIM1_BDTRInitStructure.TIM_OSSRState = TIM_OSSRState_Enable;
	TIM1_BDTRInitStructure.TIM_OSSIState = TIM_OSSIState_Enable;
	TIM1_BDTRInitStructure.TIM_LOCKLevel = TIM_LOCKLevel_1; 
	TIM1_BDTRInitStructure.TIM_DeadTime = DEADTIME;
	TIM1_BDTRInitStructure.TIM_Break = TIM_Break_Disable;
	TIM1_BDTRInitStructure.TIM_BreakPolarity = TIM_BreakPolarity_High;
	TIM1_BDTRInitStructure.TIM_AutomaticOutput = TIM_AutomaticOutput_Enable;

	TIM_BDTRConfig(TIM1, &TIM1_BDTRInitStructure);  
```



这里需要注意的地方不多，具体介绍可参考数据手册。这里只介绍一个参数，其他的随便配置都可以。

**TIM_DeadTime：** 这个参数决定了死区时间，而我们后面需要对死区时间得到一个精确值。所以这里先分析得到理论上的死区时间计算公式。

由于我们上面配置分频系数为TIM_CKD_DIV2，则死区控制的时钟为 72/2 = 36MHz，

所以 死区时间： t = 72/2* DEADTIME/1000000000 

`#define DEADTIME  (u16)((unsigned long long)CKTIM/2 \
​          *(unsigned long long)DEADTIME_NS/1000000000uL) `

最后一步便是配置中断优先级，对于熟悉stm32的同学来说便是轻车熟路了。这里唯一值得注意的点在于我们选择

TIM_IT_Update 中断，也就是按照上面的配置只会在如图时间时产生中断，这一点对后面电流采样很重要。

```
	TIM_ITConfig(TIM1, TIM_IT_Update, ENABLE);
	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	NVIC_InitStructure.NVIC_IRQChannel = TIM1_UP_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
	NVIC_Init(&NVIC_InitStructure);
```

四、 实现svpwm

终于，我们把定时器部分的内容配置完成了。我们还需要写一个函数来实现svpwm波的波形。当然这在st的电机库中也实现了，下面是为删减后的版本。

``` void SVPWM_3ShuntCalcDutyCycles(Volt_Components Stat_Volt_Input)
{
    int32_t wX,wY,wZ,wU;
    uint16_t hTimePhA = 0,hTimePhB = 0,hTimePhC = 0;
    uint16_t hDeltaDuty;
	/***************计算出相电压***************/
    wUAlpha = Stat_Volt_Input.qV_Component1 * T_SQRT3 ;
    wUBeta = -(Stat_Volt_Input.qV_Component2 * T);
    wX = wUBeta;
    wY = (wUBeta + wUAlpha)/2;
    wZ = (wUBeta - wUAlpha)/2;
	/***************判断电压矢量所处的扇区***************/
	if (wY<0)
    {
      if (wZ<0)
      {
        bSector = SECTOR_5;
      }
      else if (wX<=0)
      {
        bSector = SECTOR_4;
      }
      else // wX > 0
      {
        bSector = SECTOR_3;
      }
    }
    else // wY > 0
    {
     if (wZ>=0)
     {
       bSector = SECTOR_2;
     }
     else if (wX<=0)
     {  
       bSector = SECTOR_6;
     }
     else // wX > 0
     {
       bSector = SECTOR_1;
     }
    }
	/***************根据不同的扇区确定电压作用的时间***************/
   switch(bSector)
   {   
    case SECTOR_1:
                hTimePhA = (T/8) + ((((T + wX) - wZ)/2)/131072);
				hTimePhB = hTimePhA + wZ/131072;
				hTimePhC = hTimePhB - wX/131072;                                    
          break;
    case SECTOR_2:
                hTimePhA = (T/8) + ((((T + wY) - wZ)/2)/131072);
				hTimePhB = hTimePhA + wZ/131072;
				hTimePhC = hTimePhA - wY/131072;
          break;
    case SECTOR_3:
                hTimePhA = (T/8) + ((((T - wX) + wY)/2)/131072);
    			hTimePhC = hTimePhA - wY/131072;
    			hTimePhB = hTimePhC + wX/131072;
         break;
    
    case SECTOR_4:
                hTimePhA = (T/8) + ((((T + wX) - wZ)/2)/131072);
                hTimePhB = hTimePhA + wZ/131072;
                hTimePhC = hTimePhB - wX/131072;
         break;  
    
    case SECTOR_5:
                hTimePhA = (T/8) + ((((T + wY) - wZ)/2)/131072);
    			hTimePhB = hTimePhA + wZ/131072;
    			hTimePhC = hTimePhA - wY/131072;
    	 break;
                
    case SECTOR_6:
                hTimePhA = (T/8) + ((((T - wX) + wY)/2)/131072);
    			hTimePhC = hTimePhA - wY/131072;
    			hTimePhB = hTimePhC + wX/131072;
    default:
    	break;
   }
   /***************改变寄存器的值***************/
   CH1_ENABLE(1);
   CH2_ENABLE(1);
   CH3_ENABLE(1);
   set_PWM_1(hTimePhA);
   set_PWM_2(hTimePhB);
   set_PWM_3(hTimePhC);
} ```


```

这部分代码可以分为三个部分理解：

第一部分：

park逆变换，这部分会在另外一篇博客内详细介绍。

第二部分：

判断扇区，具体的理论请参考下面内容

![](assets\img\sector.png)

第三部分：

根据不同的扇区确定ABC三相的作用时间。具体同样参考下面图片中内容

![](assets\img\time.png)

### 小节

我们针对svpwm波形的特征，深入stm32定时器内部特性实现了svpwm波形的输出。这样我们便能试着控制直接调用上面的函数控制电机了。如果有传感器得到位置信息的话，很容易便实现正弦波驱动电机了。下一次我们会暂时将这个文件放一放。先学习clarke和park变换的内容，然后再回来继续学习剩下的内容。