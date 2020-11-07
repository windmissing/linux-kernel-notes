---
layout: post
title:  "Linux2.6虚拟内存管理"
category: [操作系统]
tags: [linux2.6]
---

*除了分段管理中的LDT和GDT有较大改动，分页管理中增加了三级分页模型以外，大部分内容可以参考[Linux0.12-内存寻址](http://blog.csdn.net/mishifangxiangdefeng/article/details/7277060)*

#### 一、分段管理：
##### 1.Linux中，段基址总是0，逻辑地址与线性地址是一致的。
或者说，在Linux中，没有实际上地使用分段机制

##### 2.一个进程可以使用一个GDT和一个LDT
GDT包含：  
（1）LDT在GDT中的描述符  
（2）3个局部段描述符  
（3）3个与高级电源管理相关的段  
（4）5个PnP的BIOS服务程序相关的段  
（5）1个特殊的TSS段  
（6）1个任务状态段TSS  
（7）内核态和用户态的代码段和数据段（共4个）  
LDT包含：8191个段  
大多数用户态下的Linux程序不使用KDT，因此内核定义了一个缺省的LDT供大多数进程共享。  

##### 3.GDT第0项不用，因为防止在加电后段寄存器未经初始化就进入保护模式并使用GDT

#### 二、分页管理：
##### 1.几种分页模型的比较
| 	|二级分页模型	|三级分页模型	|四级分页模型|
|---|---|---|---|
|操作系统支持	|Linux2.6.10以前	|Linux2.6.10	|Linux2.6.11|
|物理支持	|32位	|启用物理地址扩展（PAE）的32位（物理地址36位，线性地址32位）|	64位|
|线性地址格式	|10位PD + 10位PT + 12位offset	|2位PGD + 9位PMD + 9位PT + 12位offset	|9位PGD + 9位PUD + 9位PMD + 9位PT + 12位offset|
|页表项结构	|20位物理地址 + 12位标志位	|24位物理地址 + 12位标志位||
|页面大小	|4K	|4K|    | 
|页框数	|2^20	|2^24	 ||
|页表项大小	|4B	|8B	| |
|页表总大小	|4MB	|128MB	 ||
|一页页表包含的页表项数	|1024	|512	 ||

计算方法：  
页框大小 = 2 ^ offset  
页框数 = 物理内存大小 / 页框大小  
页表项大小 = 页表长度 / 8  
页表总大小 = 页框数 * 页表项大小  
一页页表包含的页表项 = 页面大小 / 页表项大小  
几级分布就要访问几次内存  

##### 2.为了缓解CPU与RAM之间的速度不匹配，在分页单元与主内存之间，插入一个高速缓存单元cache。
N路组关联的高速缓存：主存中的任意一个行可以存放在高速缓存N行中的任意一行。  
访问一个RAM存储单元时，先与高速缓存中的这N行进行匹配  

##### 3.为了加快线性地址的转换，引入了TLB
当CPU的cr3寄存器被修改时，本地TLB自动无效  
cr3存放的是PD的基址（这是一个物理地址）  
分段没有类似的机制，因为段址是二维的。  

##### 4.为了使二级页表模型与三级页表模型兼容，当使用二级页表时，页中间目录仅含有一个指向下属页表的目录项

##### 5.主内核页全局目录：
内核维持着一组自己使用的页表，驻留在主内核页全局目录中  
主内核页全局目录的最高目录项部分作为参考模型，为系统中的每个普通进程对应的页全局目录项提供参考模型  
内核页表的初始化是的实模式下进行的，此时分页功能还没有启用。具体过程参见Linux2.6物理内存管理  

##### 6.内核线程并不拥有自己的页表集，它们使用刚在CPU上执行过的普通进程的页表集
内核线程不访问用户态地址空间。  

##### 7.内存寻址过程中出现页面异常时的处理过程：
![](http://my.csdn.net/uploads/201204/28/1335583613_8028.gif)  
图中的连线没画箭头，都是从左往右指