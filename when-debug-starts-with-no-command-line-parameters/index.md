# 当DEBUG无参数启动时, 我们在debug什么?

## 0x00 前言
好奇无参数启动debug时, debug在对什么程序进行debug, 于是进行一番探究. 
## 0x01 过程
查了一堆类似"What DEBUG for DOS debug without parameters", "What DEBUG open by default"之类的问题, 没找到什么答案, 搜索的过程感觉在考古, 有用信息较少.  
尝试用-u指令反汇编阅读代码, 看不出什么有用信息.  
然后尝试二分法寻找指令段的结尾, 也没找到什么有用信息. 在这个过程中发现`add [BX+SI], AL`指令对应的机器码是0000.  
尝试将整个程序dump下来, 没找到怎么dump.  
于是再次返回搜索引擎, 仔细阅读[Debug (command) - Wikipedia](https://en.wikipedia.org/wiki/Debug_(command)#cite_note-Using_Debug-14), 找到了这段, 他的第一句话看起来有用: 
>When DEBUG is started without any parameters the DEBUG prompt, a "-" appears. The user can then enter one of several one or two-letter subcommands, including "A" to enter the assembler mode, "D" to perform a [hexadecimal dump](https://en.wikipedia.org/wiki/Hex_dump "Hex dump"), "T" to trace and "U" to unassemble (disassemble) a program in memory.[[13]](https://en.wikipedia.org/wiki/Debug_(command)#cite_note-TechNet-13) DEBUG can also be used as a "DEBUG script" [interpreter](https://en.wikipedia.org/wiki/Interpreter_(computing) "Interpreter (computing)") using the following syntax. 

实际上也没什么想要的信息, 还是不知道debug打开的到底是什么. 后来开始翻下面的reference, 找到了这条: 
> Sedory, Daniel B. ["A Guide to DEBUG"](http://thestarman.pcministry.com/asm/debug/debug.htm). Retrieved 2014-11-29.  

里面有一段这个: 

<a href="https://ibb.co/Wfjbwrd"><img src="https://i.ibb.co/5LwNyQz/G-NHLPK-9-4-KW7-MG-3-B7.png" alt="G-NHLPK-9-4-KW7-MG-3-B7" border="0"></a>
>**A.** When DEBUG **_starts_** with **_no_** command-line **_parameters,_** it:  
>**1)** Allocates _all_ **64 KiB** of the first **free** Memory **Segment.**  
>**2)** The Segment registers, CS, DS, ES and SS are all set to the value of that **64 KiB** Segment's location (**CS=DS=ES=SS=_Segment Location_**).
>**3)** The Instruction Pointer (IP) is set to **cs:0100** _and_ the Stack Pointer (SP) is set to **ss:FFEE** (under DOS **3.0** or above).  
>**4)** The registers, AX, BX, CX, DX, BP, SI and DI are _cleared_ to **zero** along with the **flag bits** in the Flag Register; with _one exception_: The **Interrupts** Flag is **_set_** to **E**nable **I**nterrupts. (See the Appendix, [The 8086 CPU Registers](https://thestarman.pcministry.com/asm/debug/8086REGs.htm#REGS) for more information.)

DEBUG会分配空闲的前64KB内存段, 将CS, DS, ES, SS同时指向它也就是将它同时当作代码段, 数据段, 附加段和栈段, IP设为100, SP设为FFEE. 
## 0x02 结论
所以说, DEBUG无参数打开时只是将第一段空闲内存中的无用数据作为指令序列而已, 并且作为运行在实模式下的单任务系统, DOS的内存分布相当固定, 我的DOSbox每次打开debug, 段寄存器的值都是0DBD. 
另外, 上面的["A Guide to DEBUG"](http://thestarman.pcministry.com/asm/debug/debug.htm)是个不错的DEBUG祖传使用指南, 从07年更新到20年, 可能还在持续更新. 

