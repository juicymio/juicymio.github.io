# 为什么bss上有时会有stdin,stdout,stderr指针

研究了一下bss段上为什么会有stdin,stdout,stderr, 根据经验观察似乎在使用setvbuf时会出现这种情况, 经过控制变量测试, 发现只要setvbuf这三个变量的一个, 就会使该变量出现在bss段中. 
setvbuf使用了这三个stdio.h中的extern变量 
```c
/* Standard streams.  */
extern FILE *stdin;		/* Standard input stream.  */
extern FILE *stdout;		/* Standard output stream.  */
extern FILE *stderr;		/* Standard error output stream.  */
/* C89/C99 say they're macros.  Make them happy.  */
#define stdin stdin
#define stdout stdout
#define stderr stderr
```
他们定义在stdio.c中, 不过右边的仍然是extern变量
```c
#include "libioP.h"
#include "stdio.h"

#undef stdin
#undef stdout
#undef stderr
FILE *stdin = (FILE *) &_IO_2_1_stdin_;
FILE *stdout = (FILE *) &_IO_2_1_stdout_;
FILE *stderr = (FILE *) &_IO_2_1_stderr_;
```
他们最初的定义在stdfile.c中
```c
DEF_STDFILE(_IO_2_1_stdin_, 0, 0, _IO_NO_WRITES);
DEF_STDFILE(_IO_2_1_stdout_, 1, &_IO_2_1_stdin_, _IO_NO_READS);
DEF_STDFILE(_IO_2_1_stderr_, 2, &_IO_2_1_stdout_, _IO_NO_READS+_IO_UNBUFFERED);

struct _IO_FILE_plus *_IO_list_all = &_IO_2_1_stderr_;
libc_hidden_data_def (_IO_list_all)

```
因为stdin等三个变量是未初始化的全局变量, 所以他们被分配到bss段. 在执行setvbuf时, 经过两次重定位将libc地址填入其中.  
如果不setvbuf, 这三个符号就不会被用到, 也就不会被解析.  
### 问题
发现似乎在使用setbuf时, 它们会作为libc里的变量出现在got表里, 这又是为什么呢?
