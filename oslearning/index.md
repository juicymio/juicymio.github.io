# 操作系统学习笔记

## 0x00 序言
赶上吾爱破解十五周年开放注册, 注册了个账号. 随便翻了翻精品帖子, 看到[从0到-1写一个操作系统-0xFF-!!完结撒花!!](https://www.52pojie.cn/thread-1748588-1-1.html), 感觉还不错, 于是开始动手实践, 开一个博客记录, 也是督促自己坚持下去. 
## 0x01 操作系统的启动 
### 实模式
刚开机时CPU进入实模式, 因为此时只能使用物理地址.  
实模式地址的分布是固定的,以8086为例, BIOS入口在0xFFFF0到0xFFFFF之间, 其中是一个jmp指令, 跳转到真正的BIOS位置. 中断向量表在0x00000到0x003FF. MBR加载地址在0x7C00到0x7DFF. 
### BIOS(Basic Input/Output System)
基本输入输出系统. 早期存储在ROM中, 现在存储在主板上的一个或多个芯片中. 目前其继承者UEFI(Unified Extensible Firmware Interface)正在全面取代BIOS.  
BIOS首先进行加电自检(Power-On Self-Test), 缩写为POST, 检查CPU, 内存, 主板, 硬盘, 显卡等设备. POST结束后系统BIOS调用其他设备的BIOS对各个设备进行检测和初始化. 如果自检没有出现问题, 将执行启动程序. 
### Boot
BOOT(靴子), 意为"to load a program into a computer from a disk; to start or ready for use especially by booting a program." 
BOOTSTRAP(鞋带)
> pull oneself up by one's bootstraps 靠自己自立自强
> by one's own bootstraps 自己努力, 自强

必须先运行程序, 计算机才能启动, 但是计算机不启动就无法运行程序. 这与人没法拽着鞋带把自己拉起来类似. 于是早期工程师们设法先将一小段程序装入内存, 称之为"boot", 意为"自引导程序". 
### MBR(Master boot record)
BIOS按照设定的启动顺序, 将控制权转移给启动顺序首位的存储设备. BIOS首先寻找他们的MBR, 如果该磁盘的主引导扇区(0柱头, 0磁头, 1扇区, 即前512个字节)最后两字节为0x55和0xaa, BIOS则认为MBR有效, 并将控制权转交给MBR. 如果未找到MBR, BIOS会去启动顺序中下一顺位的设备中寻找. 
MBR由三部分组成: 启动代码(446字节), 磁盘分区表(16x4字节), 结束标志(0x55, 0xaa 两字节). 
### Boot Loader
Boot Loader(启动引导程序)用于加载操作系统内核文件. 用户可以用Boot Loader来选择启动不同的操作系统. Windows的Boot Loader是Windows Boot Manager. 
### 操作系统
操作系统内核被载入内存后, 再经过一系列初始化过程完成操作系统的启动. 

## 参考文献
[计算机是如何启动的](https://www.ruanyifeng.com/blog/2013/02/booting.html)

