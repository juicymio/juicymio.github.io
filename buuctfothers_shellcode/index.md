# [BUUCTF]others_shellcode wp

## 0x00 前置知识
x86_32下应用程序调用系统调用的过程: 
1. 把系统调用号存入eax. 
2. 把函数参数存在其他寄存器(ebx, ecx, edx, esi, edi), 当系统调用参数大于6个时，全部参数应该依次放在一块连续的内存区域里，同时在 ebx 中保存指向该内存区域的指针. 
3. 触发0x80中断, 切换到内核态, 执行中断处理函数. (为了节约宝贵的中断号, Linux用int 0x80触发所有系统调用. 再为每个系统调用分配与中断号用法类似的系统调用号)

## 0x01 题目分析
IDA看到main函数里只执行了一个getShell()函数, 代码如下.  
```c
int getShell()
{
  int result; // eax
  char v1[9]; // [esp-Ch] [ebp-Ch] BYREF

  strcpy(v1, "/bin//sh");
  result = 11;
  __asm { int     80h; LINUX - sys_execve }
  return result;
}
``` 
插入了汇编代码int 0x80. 根据IDA的注释可以看到eax的值就是result的值11. 而32位下系统调用号11即为execve.  32位调用参数存在栈里, 此处即为栈顶的v[1]内的"/bin/sh". 所以相当于手动调用了系统调用execve("/bin/sh"). 


