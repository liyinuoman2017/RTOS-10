# RTOS-10
**嵌入式实时操作系统10——系统时钟节拍**

## 1.系统节拍是什么

时间管理在操作系统内核中占有非常重要的地位，操作系统内核中有大量基于时间驱动的功能。有些任务是需要周期执行，比如一个软件定时器需要一秒钟周期性运行100次；有些功能任务需要延时一段时间后再运行，比如一个传感器读取操作需要延时2000ms；比如操作系统内核也需要对运行时间进行计算，统计不同的任务运行时间和处理器利用情况。

绝大多数操作系统内核需要一个周期性的时钟源，这个时钟源称之为**时钟节拍**。为了生成时钟节拍，往往需要使用一个硬件定时器，配置硬件定时器产生一个频率为10~1000Hz之间的中断，当时钟中断发生时，中断服务程序调用操作系统内核中一个特殊程序：**系统时钟节拍服务**。

**操作系统内核必须在硬件的帮助下才能计算和管理时间**。时钟节拍中断并不是必须由硬件定时器产生，也可以由其它周期性时钟来源，如电力系统种频率为50Hz的时钟源。

![请添加图片描述](https://img-blog.csdnimg.cn/63a726aa780a414db7337b333928af33.gif)

系统时钟节拍可以设置成10Hz,也可以设置成1000Hz。时钟节拍值越大意味着硬件定时器产生的中断越频繁，系统时钟节拍服务执行得就越频繁。

**高时钟节拍的优势：**

> 1、提高操作系统内核时间管理精度 
> 2、提高任务抢占准确度

比如10Hz的时钟节拍的执行粒度为100ms，系统中的周期性事件最快为100ms一次，不可能由更高的精度了。比如1000Hz的时钟节拍，此时的执行粒度就提高了100倍，此时系统中的周期性事件最快为1ms一次，时间精度可以达到1ms。

**高时钟节拍的优劣势：**
时钟节拍越高，意味着时钟中断越频繁，处理器需要花时间执行中断程序，如果时钟节拍大到一定程度比如1MHz，此时处理器将会一直周期性的执行时钟中断程序，而用户程序将几乎得不到执行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c54ee70cc2684f16befb98daf2eb1e41.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

## 2.系统时钟节拍服务实现

系统时钟节拍的实现依靠以下两个部分：

> 1、硬件定时器
>  2、系统时钟节拍服务

硬件定时器用于产生中断，配置硬件定时器产生一个频率为10~1000Hz之间的中断，硬件定时器周期性触发中断，当定时器中断发生时处理执行定时器中断程序。

系统时钟节拍服务用于执行系统时间管理相关操作，系统时钟节拍服务被定时器中断程序调用。系统时钟节拍服务完成了以下操作：
**1、更新系统节拍时间。
2、更新等待表和就绪表。
3、处理时间片轮询。
4、切换任务。**

![请添加图片描述](https://img-blog.csdnimg.cn/93e9be71ac3c4767a08b9683e692ca78.gif)

**更新系统节拍时间**
每次定时器中断调用系统时钟节拍服务时，完成系统节拍时间更新，将系统节拍时间计数值加1，通常情况下系统节拍时间计数值为一个32位的变量，时间计数值自加时需要考虑值溢出情况。

**更新等待表和就绪表**
每次进入系统时钟节拍服务更新系统节拍时间后，操作系统内核检查等待表是否有任务完成等待，如果有任务完成等待，操作系统内核会将任务从等待表中移除，并将该任务添加到就绪表中，完成更新等待表和就绪表。

**处理时间片轮询**
每次进入系统时钟节拍服务，操作系统内核会对**当前运行优先级中**的**多个任务**进行时间片轮询操作。

**切换任务**
每次进入系统时钟节拍服务等待表和就绪表更新后，若有更高优先级任务就绪，操作系统内核将启动任务切换。









## 3.FreeRTOS系统时钟节拍实现

分析对象为：cortex-m4硬件平台，FreeRTOS操作系统。
系统时钟节拍实现依靠cortex-m4中的SysTick硬件定时器产生时钟中断，时钟中断服务为：

> SysTick_Handler

FreeRTOS操作系统中的系统时钟节拍服务为

> xTaskIncrementTick


**SysTick定时器配置成1ms中断，配置函数实现如下：**

```c
/*配置定时器      1ms中断 */
HAL_SYSTICK_Config(HAL_RCC_GetHCLKFreq()/8000);
HAL_SYSTICK_CLKSourceConfig(SYSTICK_CLKSOURCE_HCLK_DIV8);
```


**定时器中断程序SysTick_Handler实现如下：**

```c
#define xPortSysTickHandler SysTick_Handler
	void xPortSysTickHandler( void )
	{
		/* 屏蔽相关中断 */
		vPortRaiseBASEPRI();
		{
			/* 调用系统时钟节拍服务 */
			if( xTaskIncrementTick() != pdFALSE )
			{
				/* 如果由高任务就绪，切换任务*/
				portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
			}
		}
		/* 开启相关中断 */
		vPortClearBASEPRIFromISR();
	}
```


**xTaskIncrementTick为系统时钟节拍服务，完成更新系统节拍时间，更新等待表和就绪表，处理时间片轮询，切换任务，代码实现如下：**

```c
BaseType_t xTaskIncrementTick( void )
{
	TCB_t * pxTCB;
	TickType_t xItemValue;
	BaseType_t xSwitchRequired = pdFALSE;

	traceTASK_INCREMENT_TICK( xTickCount );
	/* 调度器是否打开 */
	if( uxSchedulerSuspended == ( UBaseType_t ) pdFALSE )
	{
		/* 系统节拍计数器xTickCount 镜像 */
		const TickType_t xConstTickCount = xTickCount + ( TickType_t ) 1;
		/* 系统节拍计数器xTickCount 加1*/
		xTickCount = xConstTickCount;
		/* 系统节拍计数器xTickCount 是否溢出*/
		if( xConstTickCount == ( TickType_t ) 0U ) 
		{
			taskSWITCH_DELAYED_LISTS();
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}

		if( xConstTickCount >= xNextTaskUnblockTime )
		{
			
			for( ;; )/*  将完成延时等待的相关任务移除等待表  */
			{
				/* 等待表是否为空*/
				if( listLIST_IS_EMPTY( pxDelayedTaskList ) != pdFALSE )
				{
					/* 等待表为空 则将xNextTaskUnblockTime赋值成最大值   */
					xNextTaskUnblockTime = portMAX_DELAY; 
					break;
				}
				else/* 等待表不为空*/
				{					
					/*读取等待表中的第一个任务对象
					等待表队列时按照等待时间大小进行排序，等待表队列的第一个对象就是最小等待时间对象    */
					pxTCB = listGET_OWNER_OF_HEAD_ENTRY( pxDelayedTaskList );
					/* 读取任务的延时完成时间  */
					xItemValue = listGET_LIST_ITEM_VALUE( &( pxTCB->xStateListItem ) );
					/* 比较延时完成时间和当前时间  */
					if( xConstTickCount < xItemValue )
					{
						/* 比较延时完成时间未到 退出for循环 */
						xNextTaskUnblockTime = xItemValue;
						break;
					}
					else
					{
						mtCOVERAGE_TEST_MARKER();
					}

					/* 将任务从等待表中移除*/
					( void ) uxListRemove( &( pxTCB->xStateListItem ) );

					/* 将任务从等待事情的挂起表中移除*/
					if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
					{
						( void ) uxListRemove( &( pxTCB->xEventListItem ) );
					}
					else
					{
						mtCOVERAGE_TEST_MARKER();
					}

					/* 将任务添加到就绪表中*/
					prvAddTaskToReadyList( pxTCB );

					#if (  configUSE_PREEMPTION == 1 )
					{
						/* 如果等待中的任务的优先级高于当前任务
						触发任务切换*/
						if( pxTCB->uxPriority >= pxCurrentTCB->uxPriority )
						{
							xSwitchRequired = pdTRUE;/* 设置任务切换标志 */
						}
						else
						{
							mtCOVERAGE_TEST_MARKER();
						}
					}
					#endif 
				}
			}
		}

		/*  任务时间片轮询功能  */
		#if ( ( configUSE_PREEMPTION == 1 ) && ( configUSE_TIME_SLICING == 1 ) )
		{
			/*  每次节拍进行一次当前优先级任务轮询操作
			当前优先级下任务数量大于1时执行轮询操作 */
			if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ pxCurrentTCB->uxPriority ] ) ) > ( UBaseType_t ) 1 )
			{
				xSwitchRequired = pdTRUE;/* 设置任务切换标志 */
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}
		#endif 

		#if ( configUSE_TICK_HOOK == 1 )
		{
			/* tick 钩子函数 */
			if( uxPendedTicks == ( UBaseType_t ) 0U )
			{
				vApplicationTickHook();
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}
		#endif 
	}
	else
	{
		++uxPendedTicks;

		/* tick 钩子函数 */
		#if ( configUSE_TICK_HOOK == 1 )
		{
			vApplicationTickHook();
		}
		#endif
	}

	#if ( configUSE_PREEMPTION == 1 )
	{
		if( xYieldPending != pdFALSE )
		{
			xSwitchRequired = pdTRUE;
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}
	}
	#endif 

	return xSwitchRequired;
}
```

> 未完待续…
> 
> 实时操作系统系列将持续更新
> 
> 创作不易希望朋友们点赞，转发，评论，关注。
> 
> 您的点赞，转发，评论，关注将是我持续更新的动力
> 
> 作者：李巍
> 
> Github：liyinuoman2017

