---
layout: post
title:  "Linux2.6可延迟中断"
category: [操作系统]
tags: [linux2.6]
---

#### 一、基本概念

##### 1.Linux把紧随中断要执行的操作分为三类

| 	|特点	|处理方法	|举例|
|---|---|---|---|
|第一类	|紧急的	|在禁止可屏蔽中断的情况下立即执行| 修改设备和处理器同时访问的数据结构|
|第二类	|非紧急的	|在开中断的情况下立即执行| 修改那些只有处理器才会访问的数据结构（例如，按下一个键后读扫描码）|
|第三类	|非紧急可延迟的	|由独立的函数来执行| 把缓冲区的内核拷贝到某个进程的地址空间|

##### 2.把可延迟中断从中断处理程序中抽出来，由独立的函数来执行，有助于使内核保持较短的响应时间

##### 3.Linux2.6使用可延迟函数和工作队列来处理可延迟中断，这两个都是内核函数。

#### 二、可延迟函数

##### 1.可延迟函数包括软中断和tasklet，tasklet是在软中断之上实现的

tasklet是I/O驱动程序中实现可延迟函数的首选方法。  
tasklet建立在HI_SOFTIRQ和TASKLET_SOFTIRQ这两个软中断之上  
几个tasklet可以与同一软中断相关联，每个tasklet执行自己的函数  
| 	|分配方式	|并发性	|可重入性|
|---|---|---|---|
|软中断	|编译时静态分配	|可以并发地在多个CPU上运行|	必须是可重入的，并明确地使用自旋锁保护其数据结构|
|tasklet	|运行时动态分配|	相同类型的tasklet总是被串行地执行。不同类型的tasklet可以在几个CPU上并发地执行	|不必是可重入的|

##### 2.由给定CPU激活的一个可延迟函数必须在同一个CPU上执行
可延迟函数执行时不允许内核抢占，因为从一个CPU移到另一个CPU的过程需要将进程挂起

##### 3.中断与软中断所使用的数据结构比较

| 	|中断	|软中断|
|---|---|---|
|中断向量号描述符	|irq_desc[中断向量号]|	softirq_vec[软中断号]|
|中断请求寄存器	|中断请求寄存器	|__softirq_active|
|中断屏蔽寄存器	|中断屏蔽寄存器	|__soft_mask|
|中断处理程序	|do_handler_name	
bh[]：32个|
|中断处理程序描述项	|irqaction	|softirq_action|
|中断机制的初始化	|trap_init()：异常初始化|
init_IRQ()：中断初始化	softirq_action|
|中断请求队列初始化	|init_ISA_irqs()	|open_softirq|
|中断处理程序与中断请求队列相关联	|request_irq()	|init_bh()|
|执行中断	|do_IRQ()	|do_softirq|

##### 4.激活可延迟函数

###### （1）激活软中断

A.把软中断置为把挂起状态，并周期性地检查处于挂起状态的软中断。如果有，就调用do_softirq()  
B.一般是在以下几个点来检查挂起的软中断的a.调用local_bh_enable()激活本地软中断时b.do_IRQ()完成I/O中断的处理即将退出时c.用于周期性检查挂起状态软中断的内核线程ksoftirqd被唤醒时d.else，我觉得不太重要  

###### （2）激活tasklet

A.把自己定义的描述符加入到tasklet_vec指向的链表中即可。调用tasklet_action()时，依次处理队列中的每个tasklet描述符，然后清空tasklet_vec指向的链表。  
B.tasklet的每次激活至多触发tasklet函数的一次执行，除非tasklet函数重新激活自己  

#### 三、工作队列

##### 1.它们允许内核函数被激活，并稍后由一种叫做工作者线程的特殊内核线程来执行

如果系统有n个CPU，就会创建n个工作者，但是只会创建一个工作队列

##### 2.工作队列与可延迟函数的区别：

可延迟函数运行中断上下文中，而工作队列运行在进程中下文中。  
可延迟函数不能阻塞，工作队列可执行可阻塞函数  
可延迟函数被执行时不可能有任何正在运行的进程，工作队列中的函数由内核线程来执行，不访问用户态地址空间  

##### 3.工作队列的激活
把要执行的函数的描述符插入到工作队列中。  
工作者等待函数被插入队列，被唤醒后把函数描述符取下并执行。  
由于工作队列函数可以阻塞，工作者线程可以睡觉，或移动另一个CPU  
