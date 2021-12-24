# RTOS_Study
RTOS自学笔记

Link：https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1864y1T7Z7%3Ffrom%3Dsearch%26seid%3D16364197990203970120

# 1. 物联网操作系统多任务基础
## 1.1 任务
操作系统的入口是中断

任务控制块就是描述任务属性的结构体：
- 任务堆栈信息
- 任务调度信息
- 任务创建信息
- 任务通知信息
- 任务互斥信息
- 任务调试信息

任务堆栈：
- 保存现场：CPU寄存器局部变量
- 堆栈单位：分配为4个字节为单位
- 堆栈大小：每个任务都需要自己的栈空间，应用不同，每个任务需要的栈大小也是不同的

堆栈大小确定（需要保存的内容）：
- 函数嵌套：
  - 函数局部变量
  - 函数形参
  - 函数返回地址
  - 函数内部状态值
- 任务切换：
  - 所有的寄存器都需要入栈
  - 进入中断以后其余通用寄存器和浮点寄存器入栈以及发生中断嵌套都是用的系统栈
- 堆栈打印：
  - 测试每个任务的堆栈大小

### 1.1.1 任务相关API
xTaskCreate()
- 功能概述：动态创建一个新的任务，每个任务都需要RAM来保存任务状态（任务控制块+任务栈），此接口采用动态分配内存资源。
- 参数：
  - 1.任务实现函数指针（函数名）
  - 2.任务名称（字符串）
  - 3.任务堆栈大小，单位为字
  - 4.任务传入参数
  - 5.任务优先级
  - 6.任务句柄
- 返回值：
  - pdPASS：创建成功
  - errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY:堆空间不足，失败

xTaskCreateStatic()
- 功能概述：静态创建一个新的任务，每个任务都需要RAM来保存任务状态（任务控制块+任务栈），此接口采用静态分配内存资源。
- 参数：
  - 1.任务实现函数指针（函数名）
  - 2.任务名称（字符串）
  - 3.任务堆栈大小，单位为字
  - 4.任务传入参数
  - 5.任务优先级
  - 6.任务堆栈指针
  - 7.任务控制块指针
- 返回值：
  - NULL：创建失败
  - ！=NULL：任务句柄

任务句柄：也是一个任务指针，指向的是任务控制块与任务堆栈

什么时候用静态内存，什么时候用动态内存？
- 想要自己掌控资源的时候使用静态，一般都使用动态，在系统的稳定性可以掌控在自己手中的时候，可以使用静态分配

为什么需要去做重定向？
- https://zhuanlan.zhihu.com/p/133460085
- 简单解释：
  - 在自己的原文件中直接实现标准库函数
- 好处：
  - 别人可以不需要给你源码
  - 即使没有源码，也能间接地达到修改源码的目的
  - 即使有源码，通过该属性设置，也不需要删除别人的代码去重新实现，可以保留原来的代码
  - 不需要使用回调函数的方式进行注册，可以直接重新实现该函数，非常简单
  - 存在一个默认函数实现，如果说你不想重新实现那么编译器就会使用该函数进行编译、链接，而不会在编译时出现错误或警告

## 1.2 任务挂起和恢复
### 1.2.1 接口
vTaskSuspend()
- 功能概述：将任务至于挂起状态
- 参数：任务句柄，当传入NULL时，则挂起任务自己
- 返回值：none

vTaskResume()
- 功能概述：将任务从挂起状态恢复到就绪态
- 参数：任务句柄
- 返回值：none

### 1.2.2 业务实现与流程
功能需求：
- 1.创建按键检测任务
- 2.当按键按下时，挂起LED闪烁任务
- 3.当按键松开时，恢复LED闪烁任务

业务流程：
- 按键初始化
- 按键检测任务创建
- 按键中断函数实现
- 任务内检测按键状态，根据状态进行挂起/恢复LED任务

# 2. 物联网操作系统多任务调度原理
## 2.1 任务创建和删除实现原理
### 2.1.1 任务控制块
- FreeRTOS的每个任务都有一些属性需要存储
- 把这些属性集合到一起用一个结构体表示
- 这个结构体叫做任务控制块(tcb_t)

TCB中的变量：
- *pxTopOfStack：任务堆栈栈顶
- xStateListItem：状态列表项
- xEventListItem：事件列表项
- uxPriority：任务优先级
- *pxStack：任务栈起始地址
- pcTaskName[]：任务名称
- *pxEndOfStack：任务堆栈栈底

### 2.1.2 任务创建原理深入
任务动态创建业务流程：
- 分配任务控制块内存，分配任务堆栈空间
- 初始化任务控制块，初始化任务堆栈
- 添加任务到就绪列表中

### 2.1.3 任务删除原理深入
任务删除业务流程：
- 从就绪列表删除
- 从事件表中删除
- 释放任务控制块，释放任务堆栈内存
- 开始任务调度

## 2.2 任务的挂起和恢复实现原理
### 2.2.1 任务挂起原理深入
任务挂起业务流程：
- 从就绪表中删除
- 从事件表中删除
- 添加任务到挂起列表中
- 开始任务调度

### 2.2.2 任务恢复原理深入
任务恢复业务流程：
- 从挂起表中删除
- 添加任务到就绪列表中
- 开始任务调度

## 2.3 多任务调度器基础知识
### 2.3.1 systick的重要性
systick是调度器的核心

### 2.3.2 中断管理
每一个外部中断都可以被使能或者禁止，并且可以被设置为挂起状态或者清除状态。处理器的中断可以电平的形式，也可以是脉冲形式的，这样中断控制器就可以处理任何中断源。

中断和异常向量表：
- 中断时外部的，异常是内部的
- 当异常或中断发生时，处理器会把PC设置为一个特定地址，这一地址就称为异常向量。每一类异常源都对应一个特定的入口地址，这些地址按照优先级排列以后就组成一张异常向量表。

向量化处理中断的好处：
- 统一处理方式需要软件去完成，采用向量表处理异常，M0处理器会从存储器的向量表中，自动定位异常的程序入口，从发生异常到异常的处理中间的时间被缩减。

中断和异常的区别：
- 中断是微处理器外部发送的，通过中断通道送入处理器内部，一般是硬件引起的，比如串口接收中断，而异常通常是微处理器内部复燃生的，大多是软件引起的，比如除法出错异常，特权调用异常等待。不管是中断还是异常，微处理器通常都有相应的中断/异常服务程序。

嵌套中断：
STM32F0中断的优先级：
- 3个固定的优先级，都是负值，不能改变
- 四个可编程优先级，用两个bit位表示，00，01，10，11
- 优先级越小，优先级越高
- 不同优先级的终端同时发生，优先处理优先级编号小的那个。同样优先级的中断同时发生，中断向量号较小的那个优先响应。

Cortex-M0寄存器组————特殊寄存器
- xPSR：组合程序状态寄存器，该寄存器由三个程序状态寄存器组成
  - APSR：包含前一条指令执行后的条件标志
  - IPSR：包含当前ISR的异常编号
  - EPSR：包含Thumb状态位
- PRIMSK：中断屏蔽特殊寄存器
- CONTROL：控制寄存器
  - \[PRIV\]: 为0，处理器处于线程模式的特权级，为1为非特权级
  - \[SPSEL\]:为0时，线程模式使用MSP，为1时使用PSP
  - 处理器模式时，固定使用MSP

Cortex-M0工作模式
- 线程模式：芯片复位后，即进入线程模式，执行用户程序
- 处理模式：当处理器发生了异常或者中断，则进入处理模式进行处理、处理完成后返回线程模式
- Thumb状态：正常运行时处理器的状态
- 调试状态：调试程序时处理器的状态

Cortex-M0寄存器组————通用寄存器
- R0-R12：通用寄存器
- R13：堆栈指针SP，Cortex-M0在不同物理位置上存在两个栈指针，主栈指针MSP，进程栈指针PSP。在处理模式下，只能使用主堆栈，在线程模式下，可以使用主堆栈也可以使用进程堆栈，这主要是由CONTROL寄存器控制完成。系统上电的默认栈指针是MSP。
- R14：连接寄存器(LR)
- R15：程序计数器(PC)

PendSV异常：
- 为什么需要使用PendSV
  - 在RTOS中，不单单是时间片调度处理，若时间分片为1ms，则systick能使用的中断抢占处理最小为1ms（systick只能做时间片的调度），若需要更小时间维度的中断处理，需要使用PendSV.
  - 除了systick可以去触发PendSV异常，用户代码也可以去触发PendSV异常

Cortex-M0上下文切换
- 进入PendSV时存储器：自动入栈
- 进入PendSV时存储器：保存PSP的数值
- 指向任务B的上下文：加载下一个任务的PSP值
- 任务B的上下文恢复

## 2.4 多任务调度原理
### 2.4.1 多任务启动
多任务启动流程：
- 创建空闲任务
- 配置SysTick，PendSV为最低优先级：为了保证实时性，只用高优先级的中断处理完之后才可以执行调度器
- 配置SysTick寄存器
- 启动第一个任务

初始化systick代码（配置寄存器）
```
void vportsetupTimerInterrupt(void)
{
    portNVIC_SYSTICK_LOAD = (configCPU_CLOCK_HZ / configTICK_RATE_HZ) - 1UL;
    portNVIC_SYSTICK_CTRL = portNVIC_SYSTICK_CLK | portNVIC_SYSTICK_INT | portNVIC_SYSTICK_ENABLE;
}
```

### 2.4.2 启动第一个任务
任务启动的业务流程：
- 获取当前任务栈顶
- 手动出栈R4-R11 R14
- 更新栈顶到PSP
- 使能全局中断，调用异常返回指令

### 2.4.3 PendSV业务流程
PendSV业务流程：
- 读取当前PSP值，获取当前任务栈顶
- 保存R4-R11 R14到当前栈中
- 更新栈顶到当前任务控制块中，保存R3到栈中关闭中断
- 查找优先级最高的任务，更新当前任务控制块，开启中断，出栈R3值
- 出栈R4-R11 R14到当前栈中，更新栈顶到PSP，调用移除返回指令

## 2.5 系统时钟节拍详解
### 2.5.1 SysTick初始化
SysTick初始化：
- 配置SysTick装载值
- 使能SysTick时钟源，使能SysTick中断，使能SysTick

### 2.5.2 SysTick中断服务函数
SysTick中断服务函数：
- 关闭中断
- Tick值增加，SysTick任务调度，启动PendSV
- 开启中断

### 2.5.3 SysTick任务调度
SysTick任务调度：
- 系统节拍数加1，判断是否溢出，溢出更新任务锁定时间
- 判断是否有任务需要解除阻塞，获取延时列表第一个任务控制块（时间排序），获取状态列表值，判断时间是否到达，未到达退出
- 任务阻塞事件到达，从延时列表中删除，从事件列表中删除，添加到就绪列表
- 如果使用抢占内核，判断任务优先级是否大于当前任务，开启任务调度
- 如果使用时间片调度，判断当前优先级下是否还有其他任务，开启任务调度器

## 2.6 系统延时函数应用
### 2.6.1 系统延时API详解
vTaskDelay()
- 功能概述：使任务进入阻塞态，根据传入的参数延时多少个tick（系统节拍）
- 参数：延时周期，宏pdMS_TO_TICKS()可以把ms转换为tick
- 返回值：none

vTaskDelayUntil()
- 功能概述：使任务进入阻塞态，直到一个绝对延时时间到达
- 参数：
  - 1.记录任务上一次唤醒系统节拍值
  - 2.系统延时周期(tick)
- 返回值：none

vTaskGetTickCount()
- 功能概述：获取系统节拍值，从调度器开启开始计数
- 参数：none
- 返回值：当前系统节拍值

### 2.6.2 相对延时与绝对延时的区别
相对延时：任务执行时间与延时时间是两个步骤执行的，如果中间发生中断，则实际感知上的任务延时时间将发生变化；指每次延时都是从执行函数vTaskDelay()开始，直到延时指定的时间（参数：滴答值）结束。
<br>
绝对延时：任务执行时间包含在延时时间当中，若在延时时间内任务已执行完成，任务将在延时时间到达后才会执行下一次任务；指每隔指定的时间（参数：滴答值），执行一次调用vTaskDelayUntil()函数的任务。

区别解释：https://blog.csdn.net/ba_wang_mao/article/details/105946369

### 2.6.3 相对延时与绝对延时应用
功能需求：
- 创建一个任务使用vTaskDelayUntil()
- 分别在两个任务里定时5s打印一次任务运行状态
- 观察两种延时接口的区别

# 3. 物联网操作系统消息队列
## 3.1 系统延时函数实现原理
### 3.1.1 vTaskDelay
- 挂起调度器
- 添加任务到延时列表
- 恢复调度器，进行上下文切换

### 3.1.2 vTaskDelayUntil
- 挂起调度器
- 判断记录的系统节拍值是否溢出；如果溢出，并且大于当前滴答值；把当前任务添加到延时列表。
- 判断记录的系统节拍值是否溢出；没有溢出，当前定时间隔小于记录值，或者大于系统节拍值；把当前任务添加到延时列表。
- 更新记录值；恢复调度器，进行上下文切换。

### 3.1.3 vTaskSuspendAll/xTaskResumeAll（调度锁）
vTaskSuspendAll：
- ++uxSchedulerSuspended
- 上下文切换中判断uxSchedulerSuspended

xTaskResumeAll：
- 进入临界区，挂起记录减一
- 判断挂起就绪列表是否为空，不为空，添加任务到就绪列表中；如果优先级高于当前任务，则进行上下文切换
- 判断调度器挂起后的SysTick值，重新遍历阻塞列表进行上下文切换

## 3.2 消息队列概念及其应用
### 3.2.1 消息队列定义
消息队列的作用：
- 消息队列可以在任务与任务间、中断和任务间传递消息；实现任务接收来自其他任务或中断的不固定长度的消息。

### 3.2.2 FreeRTOS消息队列介绍
在操作系统中，队列的实现与裸机之间的区别：
- 在操作系统中，队列就变成了消息队列；
- 在消息队列中，操作系统会使任务进入阻塞态；
- 可以保证任务的实时性

### 3.2.3 FreeRTOS消息队列工作原理
- 消息队列本身就是一块内存
- 消息队列中包含消息队列的控制块和队列的内存空间，其中队列的内存空间分为两块，一块是队头，一块是队尾

## 3.3 消息队列函数应用
### 3.3.1 功能需求
1. 使用消息队列检测串口输入
2. 通过串口发送字符串openled2，openled3，openled4分别打开板载led2，led3，led4
3. 通过串口发送字符串closeled2，closeled3，closeled4，分别关闭板载led2，led3，led4

### 3.3.2 API
xQueueCreate()
- 功能概述：创建一个消息队列，并返回消息队列句柄
- 参数：
  - 1.队列一次可容纳消息的最大长度
  - 2.队列中每个消息体大小
- 返回值：
  - NULL：创建失败
  - Any other value：创建成功，返回消息队列句柄（指向队列控制块的指针）

xQueueSend()
- 功能概述：在任务中往队列中传入消息，xQueueSend等价于xQueueSendToBack入到队尾，xQueueSendToFront入到队头
- 参数：
  - 1.消息队列句柄
  - 2.要发送的消息的地址
  - 3.阻塞等待时间
- 返回值：
  - pdPASS：发送成功
  - errQUEUE_FULL：队列已经满，发送失败

xQueueSendFromISR()
- 功能概述：在中断中往队列中传入消息xQueueSendFromISR等价于xQueueSendToBackFromISR入到队尾，xQueueSendToFrontFromISR入到队头
- 参数：
  - 1.消息队列句柄
  - 2.要发送的消息的地址
  - 3.NULL
- 返回值：
  - pdTRUE：发送成功
  - errQUEUE_FULL:队列已经满，发送失败
- 注意事项：调用此函数，会触发上下文切换（当前被中断的任务优先级低于解除阻塞的任务），在启动调度器之前不能调用此函数
  - 中断不能被阻塞

xQueueReceive()
- 功能概述：在任务中读取消息队列消息
- 参数：
  - 1.消息队列句柄
  - 2.接收消息的缓冲区
  - 3.阻塞等待时间
- 返回值：
  - pdPASS：读取成功
  - errQUEUE_EMPTY：消息队列为空

### 3.3.3 消息队列接收和发送功能开发
消息队列接收流程：
- 串口中断使能，串口中断服务函数入队操作
- 从消息队列出队一直等待，当接收到第一个消息循环从消息队列出队，阻塞等待50ms完成消息接收
- 消息解析控制功能


## 3.4 消息队列实现原理
### 3.4.1 消息队列控制块
从头到尾（起始地址到结束地址）：
- pcHead
- pcTail
- pcWriteTo
- pcReadFrom
- xTasksWaitingToSend
- xTasksWaitingToReceive
- uxMessagesWaiting
- uxLength
- uxItemSize
- cRxLock
- cTxLock
- queue
- queue
- ...

### 3.4.2 消息队列创建
填充消息队列控制块中的内容

消息队列可以当做信号量使用

### 3.4.3 消息队列删除
使用free释放所有内存空间

### 3.4.4 消息队列在任务中发送
![消息队列在任务中发送](queue_send_in_task.PNG)

### 3.4.5 消息队列在中断中发送
![消息队列在中断中发送](queue_send_in_ISR.PNG)

### 3.4.6 消息队列在任务中接收
![消息队列在任务中接收](queue_receive_in_task.PNG)

### 3.4.7 消息队列在中断中接收
![消息队列在中断中接收](queue_receive_in_ISR.PNG)

# 4. 物联网操作系统软件定时器
## 4.1 软件定时器概念及其应用
### 4.1.1 软件定时器定义
控制硬件实现软件，使资源缺少的硬件定时器上可以实现多个软件定时器

### 4.1.2 FreeRTOS软件定时器介绍
软件定时器的组成：
- 任务或中断（设置时间）
- 软件定时器本身
- 回调函数

## 4.2 软件定时器函数应用
### 4.2.1 功能需求
- 使用软件定时器功能完成闹钟功能设计
- 当闹钟到达时，可根据执行动作，触发相关的LED亮灭

### 4.2.2 API
xTimerCreate()
- 功能概述：创建一个软件定时器，并返回一个软件定时器句柄；创建软件定时器，并没有启动软件定时器。
- 参数：
  - 1.定时器名称：调试使用
  - 2.定时周期值：单位为多少tick值
  - 自动装载标志：pdTRUE/pdFALSE（用于判断定时器是否重复使用）
  - 软件定时器标识符
  - 软件定时器回调函数：void cCallbackFunctionExample(TimerHandle_t xTimer)
- 返回值：
  - NULL：创建失败
  - any other value：创建成功，返回事件标志组句柄

xTimerStart()
- 功能概述：启动软件定时器
- 参数：
  - 1.软件定时器句柄
  - 2.发送软件定时器命令，阻塞等待时间，其实内部调用的消息队列
- 返回值：
  - pdPASS：启动成功
  - pdFAIL：启动失败

xTimerReset()
- 功能概述：重启软件定时器，如果软件定时器已经启动，则重新计算超时时间，如果软件定时器没有启动，则启动软件定时器。
- 参数：
  - 1.软件定时器句柄
  - 2.发送软件定时器命令，阻塞等待时间，其实内部调用的消息队列
- 返回值：
  - pdPASS：重启成功
  - pdFAIL：重启失败

pvTimerGetTimerID()
- 功能概述：获取软件定时器标识符值
- 参数：获取软件定时器句柄
- 返回值：软件定时器标识符

xTimerChangePeriod()
- 功能概述：修改软件定时器周期值，如果软件定时器没有启动，会启动软件定时器
- 参数：
  - 1.软件定时器句柄
  - 2.新的定时周期
  - 3.阻塞等待时间
- 返回值：
  - pdPASS：修改成功
  - pdFAIL：修改失败

### 4.2.3 功能设计
用户通过对串口终端写入数据设置闹钟参数，通过芯片驱动GPIO控制LED。

功能业务划分：
- 实时时钟：RTC功能开发
- 命令参数配置：串口解析功能开发
- 软件定时功能：软件定时器
- 多任务消息同步：消息队列

### 4.2.4 功能实现
功能实现流程一：
- STM32CubeMX配置
  - 配置RTC
  - 配置串口
  - 创建任务
  - 创建消息队列
- 实时时钟读写操作：
  - 设置实时时钟
  - 读取实时时钟

功能实现流程二：
- 命令解析任务：
  - 使能串口接收中断
  - 串口中断发送消息队列
  - 解析命令字符串
  - 解析实时时钟字符串
  - 解析闹钟字符串
  - 计算闹钟与实时时钟间隔
- 软件定时器回调函数：
  - 定时打印实时时钟
  - 闹钟回调函数
- LED处理任务：
  - LED处理任务

## 4.3 软件定时器实现原理
### 4.3.1 软件定时器控制块
包含内容：
- xMessageID
- xTimerParameters
  - xMessageValue
  - pxTimer
    - pcTimerName
    - xTimerListItem
    - xTimerPeriodInTicks
    - uxAutoReload
    - pvTimerID
    - pxCallbackFunction
- xCallbackParameters
  - pxCallbackFunction
  - pvParameter1
  - ulParameter2

### 4.3.2 软件定时器任务&软件定时器创建
软件定时器任务创建
- vTaskStartScheduler
- xTimerCreaterTimerTask
- prvCheckForValidListAndQueue
  - 进入临界段
  - 初始化定时器活动列表1
  - 初始化定时器活动列表2（两个活动列表为了防止溢出）
  - 创建定时器消息队列
  - 退出临界段
- xTaskCreate
- prvInitialiseNewTimer
- vListInitialiseItem

### 4.3.3 软件定时器启动&停止
软件定时器启动&停止：
![软件定时器启动&停止](software_timer_start.PNG)

### 4.3.4 软件定时器任务
![软件定时器任务](software_timer_task.PNG)

prvGetNextExpireTime：<br>
![prvGetNextExpireTime](prvGetNextExpireTime.PNG)

prvProcessTimerOrBlockTask:<br>
![prvProcessTimerOrBlockTask](prvProcessTimerOrBlockTask.PNG)

prvProcessReceivedCommands:<br>
![prvProcessReceivedCommands](prvProcessReceivedCommands.PNG)


# RTOS韦东山版（寄存器版）
Link：https://www.bilibili.com/video/BV1V54y1C7hq?spm_id_from=333.999.0.0

# 1. 单片机概述
## 1.1 嵌入式发展史简述及相关概念
需要了解的相关概念：
- 计算机系统架构：CPU+RAM+Storage
- MPU/CPU
- MCU：CPU+RAM+FLASH
- 应用处理器：MCU的升级版

简单概括：有计算能力的非电脑就是嵌入式设备

## 1.2 嵌入式系统硬件组成
一句话引出整个嵌入式系统：支持多种设备启动

CPU如何实行SPI FLASH上的代码？一上电，CPU执行的第一个程序、第一条指令在哪里？
- 在ROM中

ARM板子支持多种启动方式：XIP设备启动、非XIP设备启动等等。
<br>
比如：Nor Flash、SD卡、SPI FLASH，甚至支持UART、USB、网卡启动。
<br>
这些设备中，很多都不是XIP设备。

既然CPU无法直接运行非XIP设备的代码，为何可以从非XIP设备启动？
- 上电后，CPU运行的第一条指令、第一个程序，位于片内ROM中，它是XIP设备。
- 这个程序会执行必要的初始化，
- 比如设置时钟、设置内存；
- 再从“非XIP设备”中把程序读到内存；
- 最后启动这个程序

# 2.硬件知识
## 2.1 LED原理图
点亮LED程序在ARM中相当于编程中的Hello World

如何点亮？
- 看原理图确定控制LED的引脚
- 看主芯片手册确定如何设置/控制引脚
- 写程序

输出高电平或低电平点亮取决于原理图

三极管

二极管

## 2.2 GPIO引脚操作方法概述
GPIO模块一般结构：
- 有多组GPIO，每组有多个GPIO
- 使能：电源/时钟
- 模式：引脚可用于GPIO或其他功能
- 方向：引脚模式设置为GPIO时，可以继续设置它是输出引脚还是输入引脚
- 数值：
  - 对于输出引脚，可以设置寄存器让它输出高、低电平
  - 对于输入引脚，可以读取寄存器得到引脚的当前电平

GPIO寄存器操作：
- 芯片手册一般有相关章节，用来介绍：power/clock，可以设置对应寄存器使能某个GPIO模块，有些芯片的GPIO是没有使能开头的，即它总是使能的
- 一个引脚可以用于GPIO、串口、USB或其他功能，有对应的寄存器来选择引脚的功能
- 对于已经设置为GPIO功能的引脚，有方向寄存器用来设置它的方向：输出、输入
- 对于已经设置为GPIO功能的引脚，有数据寄存器用来写、读引脚电平状态

## 2.3 STM32F103的GPIO操作方法
引脚频率越高，在高低电平进行变化时，变化的曲线会更陡峭，更加接近于理想状态下的高低电平变化

# 3. ARM与汇编
## 3.1 地址空间&RISC&CISC
在ARM CPU看来，内存、IO的操作是一样的

在x86架构中内存和IO是分开的
- x86架构使用不同的指令访问同样的内存地址，说明这两个内存地址不在同一片内存空间中。

### 3.1.1 RISC
RISC：精简指令集，有如下特点：
- 对内存只有读、写指令
- 对于数据的运算是在CPU内部实现
- 使用RISC指令的CPU复杂度小一点，易于设计

对于乘法运算a = a * b，在RISC中要使用4条汇编指令：
- 读内存a
- 读内存b
- 计算a*b
- 把结果写入内存

### 3.1.2 CISC
CISC：复杂指令集，它所用的指令比较复杂，比如某些复杂的指令，它是通过“微程序”来实现的。<br>
比如执行乘法指令是，实际上会去执行一个“微程序”，在“微程序”例，一样是去执行四步操作：
- 读内存a
- 读内存b
- 计算a * b
- 把结果写入内存

但是对于程序员来说，他看不到“微程序”，他好像用一条指令就搞定了这一切。

### 3.1.3 RISC和CISC比较
- CISC的指令能力强，但多数指令使用率低，却增加了CPU的复杂度，指令是可变长格式；
- RISC的指令大部分为单周期指令，指令长度固定，操作寄存器，对于内存只有Load/store操作
- CISC支持多种寻址方式；
- CISC通过微程序控制技术实现；
- RISC增加了通用寄存器，硬布线逻辑控制为主，采用流水线；
- CISC的研制周期长
- RISC优化编译，有效支持高级语言

## 3.2 ARM内部寄存器
无论是Cortex-M3/M4，还是Cortex-A7，CPU内部都有R0、R1...、R15寄存器；他们可以用来暂存数据。

对于R13、R14、R15，还另有用途：
- R13：别名SP（Stack Pointer），栈指针
- R14：别名LR（Link Register），用来保存返回地址
- R15：别名PC（Program Counter），程序计数器，表示当前指令地址，写入新值即可跳转

除了以上寄存器之外，M3/M4中还有一个寄存器：xPSR（Program Status Register），在A7中是cPSR（Current Program Status Register）。

xPSR实际上对应3个寄存器：
- APSR：应用PSR
- IPSR：中断PSR
- EPSR：执行PSR

这三个寄存器，可以单独访问：
- MRS R0,APSR：读APSR
- MRS R0,IPSR：读IPSR
- MSR APSR,R0：写APSR

这三个寄存器，也可以一次性访问：
- MRS R0,PSR：读组合程序状态
- MSR PSR,R0：写组合程序状态

## 3.3 ARM汇编
### 3.3.1 ARM指令集
一开始，ARM公司发布两类指令集：
- ARM指令集，这是32位的，每条指令占据32位，高效，但是太占空间
- Thumb指令集，这是16位的，每条指令占据16位，节省空间
- 要节省空间用Thumb指令，要效率时用ARM指令

一个CPU既可以运行Thumb指令，也能运行ARM指令。怎么区分当前指令是Thumb指令还是ARM指令呢？
- 程序状态寄存器中有一位，名为T，它等于1时表示当前运行的是Thumb指令。

假设函数A是使用Thumb指令写的，函数B是使用ARM指令写的，怎么调用A/B？
- 我们可以往PC寄存器例写入函数A或B的地址，就可以调用A或B

但是怎么让CPU在执行A函数时进入Thumb状态，在执行B函数时进入ARM状态？
- 调用函数A时，让PC寄存器的Bit0等于1，即：PC=函数A地址+(1<<0);
- 调用函数B时，让PC寄存器的Bit0等于0，即：PC=函数B地址

汇编指令可分为：
- 内存读写
- CPU运算
- 函数调用的跳转/分支
- 比较

引入Thumb2指令集，可以对16位、32位混合编程。（针对Cortex-M）

在程序前面用CODE32/CODE16/THUMB表示指令集：ARM/Thumb/Thumb2

### 3.3.2 立即数
立即数：
- 有这样一条指令：MOV R0,#VAL
- 意图是把VAL这个值存入R0寄存器

VAL可以是任意值吗？
- 不可以，必须是立即数

为什么？
- 假设VAL可以是任意数，MOV R0,#VAL本身是16位或32位，剩余给VAL的值是在一定的范围内的，所以VAL必须符合某些规定
  - 立即数：某个8位数 循环移位 偶位数
  - a constant that can be produced by rotating an 8-bit value by any even number of bits within a 32-bit word

### 3.3.3 LDR伪指令
去判断一个VAL是否为立即数很麻烦，并且我就是想把任意数值赋给R0，怎么办？
- 可以使用伪指令 LDR R0,=VAL
- LDR作为伪指令是，指令中有一个=，否则它就是真实的LDR指令了
- 编译器会把伪指令替换成真实的指令，比如：
  - LDR R0,=0X12
  - 0x12是立即数，那么替换为：MOV R0,#0X12
- LDR R0,=0X12345678
  - 0X12345678不是立即数，那么替换为：
  - LDR R0,\[PC,#offset\] 2.使用Load Register读内存指令读出值，offset是链接程序时确定的
  - ...
  - Label DCD 0x12345678 1.编译器在程序某个地方保存有这个值

### 3.3.4 ADR伪指令
ADR的意思是：address，用来读某个标号的地址
- ADR{cond} Rd，label

实例：
```
ADR R0,Loop

Loop 
    ADD R0,R0,#1
```

## 3.4 内存访问指令
### 3.4.1 四条内存指令
LDR：Load Register

LDM：Load Multiple Register

STR：Store Register

STM：Store Multiple Register

LDM{add_mode} {cond} Rn{!},reglist{^}

STM{add_mode} {cond} Rn{!},reglist{^}

addr_mode:
- IA：increment after，每次传输后才增加Rn的值
- IB：increment before，每次传输前就增加Rn的值（ARM指令才能用）
- DA：decrement after，每次传输后才减小Rn的值（ARM指令才能用）
- DB：decrement before，每次传输前就减小Rn的值
- ！：表示修改后的Rn值会写入Rn寄存器，如果没有！，指令执行完后Rn恢复/保持原值
- ^：会影响CPSR

### 3.4.2 栈的四种方式
根据栈指针指向，可分为满(Full)/空(Empty)：
- 满SP指向最后一个入栈的数据，需要先修改SP再入栈
- 空SP指向下一个空位置，先入栈再修改SP

根据压栈时SP的增长方向，可分为增/减：
- 增：SP变大
- 减：SP变小

常用的满减：
- 入栈时用STMDB，也可以用STMFD，作用一样
- 出栈时用LDMIA，也可以用LDMFD，作用一样
- FD:Full Decrement，满减

## 3.5 数据处理指令
UAL汇编格式：
- Operation{cond} {S} Rd,Rn,Operand2

## 3.6 跳转指令
### 3.6.1 跳转指令简介
C程序中，函数A调用函数B的实质是什么？
- 跳转去执行函数B的代码，函数B执行完后，还有回到函数A继续执行后面的代码。

对应的汇编指令就是跳转指令

### 3.6.2 分支/跳转指令
核心指令：
- B：Branch，跳转
- BL：Branch with Link，跳转前先把返回地址保持在LR寄存器中
- BX：Branch and eXchange，根据跳转地址的Bit 0切换为ARM或Thumb状态（0：ARM状态，1：Thumb状态）
- BLX：Branch with Link and eXchange，根据跳转地址的Bit 0切换为ARM或Thumb状态（0：ARM状态，1：Thumb状态）

# 4. 编程知识
## 4.1 字节序&位操作
低位保存在低地址：小字节序，little endian

高位保存在低地址：大字节序，big endian

## 4.2 汇编&反汇编&机器码
汇编：汇编文件转换为目标文件（里面是机器码）

反汇编：可执行文件（目标文件，里面是机器码）转换为汇编文件

PC，the program counter:
- when executing an ARM instrction, PC reads as the address of trhe current instruction plus 8
- when executing a Thumb instrction, PC reads as the address of trhe current instruction plus 4

## 4.3 C与汇编深入分析
汇编代码如何调用C语言代码：
- BL main

如何传参？
- r0-r3：调用者和被调用者之间传参数（给函数传参，存放函数的返回值）
- r4-r11：函数可能被使用，所以在函数的入口保存它们，在函数的出口恢复它们（用来保存局部变量）
