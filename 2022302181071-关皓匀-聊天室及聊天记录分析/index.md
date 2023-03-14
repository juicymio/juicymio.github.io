# 

# 实验目的
搭建使用TCP协议的聊天室, 并实现简单的聊天记录分析. 
# 实验设备(环境)及要求
代码中socket编程部分使用了winsock2库, 必须在windows系统下编译和运行且系统必须含有wsock32.dll和ws2_32.dll, 否则无法正常运行. 
若使用gcc编译需要在编译时添加`-lwsock32 -lws2_32`手动连接两个库. 如下
```sh
gcc client.c -o client -lwsock32 -lws2_32
gcc server.c -o server -lwsock32 -lws2_32
```
为了方便测试, 代码中直接设置了服务器为本机(127.0.0.1), 若需要测试局域网/广域网只需修改客户端代码的服务器ip.
# 实验内容与步骤
## 问题描述与分析
聊天室分为服务器端和客户端, 每个客户端都分别与服务器建立连接, 客户端向服务器发信息, 服务器将信息转发给聊天室内其他客户端. 如果服务器只有一个线程,不同的客户端同时与服务器连接就会产生冲突与阻塞, 这就需要服务器使用多线程技术, 每个线程单独与一个客户端通信. 同时客户端的发信与收信互不关联, 同时进行, 也需要两个线程来完成. 
聊天记录分析有按用户名, 关键词查询聊天记录和提取近n条聊天记录的功能.  
## 模块划分
1. 服务器端(server)
	 负责与所有客户端连接, 接受每个客户端发来的信息并过滤违禁词后转发给所有其它客户端, 同时存储聊天记录. 
2. 客户端(client)
	与服务器建立连接, 由两个线程完成发信息和收信息的功能.
3. 聊天记录分析(log) 
	多种方式查询聊天记录. 
## 流程设计
![[1_2 1.png]]
## 核心代码解读
服务器端(server):
由于服务器端对各个客户端的socket的操作是发送信息时向所有客户端发送, 某个客户端下线时单独删除其socket, 即**遍历查询, 单点修改**, 完全不需要链表最不擅长的随机查询, 所以适合使用链表存储线程句柄. 以下代码摘自server.c. 
```c
typedef struct linknode
{
    int sock;
    struct linknode* next;
} node;
node* head;

node* add_node(node* current, int sock)
{
    node* newnode = (node*)(malloc(sizeof(node)));
    newnode->sock = sock;
    newnode->next = NULL;
    current->next = newnode;
    return newnode;
}
//添加节点
current = add_node(current, client_sock);
//删除节点
node* temp = head;
for (node* p = head; p != NULL; p = p->next)
{
	if (p->sock == client_sock)
	{
		temp->next = p->next;
		free(p);
		break;
	}
	temp = p;
}
```
违禁词和谐: 用KMP算法匹配聊天记录中的违禁词, 并替换成'￥'和'\*'. 
```c
void keyword_detection(char *S)
{
    int pos[BUF];
    for (int i = 0;i < wcnt - 1;i ++)
    {
        int len = strlen(keywords[i]);
        int flag = KMP_multiple(S, keywords[i], pos);
        if (flag) continue;
        
        for (int j = 0; pos[j] != -1; j ++)
        {
            for (int k = 0; k < len; k ++)
            {
                if (keywords[i][k] < 0)
                {
                    S[pos[j] + k] = 0xA3;
                    k ++;
                    S[pos[j] + k] = 0xA4;
                    //￥ gbk编码为0xA3A4
                }
                else
                    S[pos[j] + k] = '*';
            }
        }
    }
}
```
KMP_multiple函数在kmp.h中, 它实现了对文本串中所有模式串的匹配, 所有模式串出现的位置存储在形参传入的res整型数组(即上文代码的pos)中:
```c
int KMP_multiple(char *S, char *T,int* res)
{
    int flag = 0, cnt = 0;
    int next[256 + 30];
    cal_next(T, next); 
    int k = -1;
    int len_s = strlen(S), len_t = strlen(T);
    for (int i = 0; i < len_s; i ++)
    {
        while (k > -1 && T[k + 1] != S[i])
            k = next[k];
        if (T[k + 1] == S[i]) k ++;
        if (k == len_t - 1) 
        {
            flag = 1;
            res[cnt ++] = i - len_t + 1;
            k = -1;
            res[cnt] = -1;
        }
    }
    if (flag) return 0;
    return -1;
}
```

聊天记录分析: 
按关键词查询聊天记录同样使用了KMP算法进行字符串匹配, 将时间复杂度从暴力匹配的$O(nm)$提升到了$O(n+m)$, 由于是在很多文本串中匹配同一个模式串, 所以计算next数组的额外开销并不显著.以下代码摘自kmp.h
```c
void cal_next(char *T, int *next)
{
    next[0] = -1;
    int k = -1;
    int len = strlen(T);
    for (int q = 1; q < len; q ++)
    {
        while (k > -1 && T[k + 1] != T[q]) k = next[k];
        if (T[k + 1] == T[q]) k ++;
        next[q] = k;
    }
}
int KMP(char *S, char *T)
{
    int next[BUF];
    cal_next(T, next); 
    int k = -1;
    int len_s = strlen(S), len_t = strlen(T);
    for (int i = 0; i < len_s; i ++)
    {
        while (k > -1 && T[k + 1] != S[i])
            k = next[k];
        if (T[k + 1] == S[i]) k ++;
        if (k == len_t - 1) return i - len_t + 1;
    }
    return -1;
}
```
错误处理: 在代码各处进行错误检测, 并统一使用error函数完成错误信息打印和保证程序异常退出前终止ws2_32.dll的使用. 
```c
void error(const char* msg)
{
    printf("%s\n", msg);
    WSACleanup();
    exit(1);
}
```
用事件进行线程同步, 在多个线程可能产生冲突的位置进行等待, 防止出现错误. 
```c
WaitForSingleObject(event, INFINITE);
```
# 实验结果分析
## 问题记录
问题1. 由于Windows系统默认使用GB2312编码, 而vscode的代码默认使用UTF-8, 编码不同导致在文件写入时出现了乱码

解决方案1. 将代码修改为GBK编码 

问题2. 聊天记录不能实时同步到文件中  

解决方案2. 该问题是由于文件写入先存在了缓存区, 所以在每次输入后立即fflush(fp)即可

## 使用说明
0. 修改keyword.txt设置违禁词, 每个违禁词占一行, 最多10000个违禁词, 每个违禁词长度不能超过256(即消息最大长度), 在发送时违禁词中的中文和英文会在服务器端转发前分别被替换为'\￥'和'\*'. 
1. 打开server.exe启动服务器
2. 打开client.exe启动客户端, 输入合法用户名(30字符以内, 中文占2个字符)后上线, 上线后所有已在线成员会收到上线提醒,可以直接输入消息, 按下回车发送. 发送'q'字符下线并退出客户端, 下线后所有在线成员会收到下线提醒. 若不下线直接关闭程序可能导致连接异常. 需要打开至少两个client才能看到效果. 
3. 聊天记录在log.txt中实时存储, 无需关闭server, 随时可以打开查看或使用聊天记录分析工具log.exe查看, 该程序自带操作说明.
4. 可以在server.exe中按下q键退出, 但退出操作不会立即执行, 而是在按下q键后的第一个连接建立后进行. 也可以直接关闭server.exe程序来退出, 无论如何退出, 所有客户端的连接都会断开. 
注: 必要顺序为先编辑keyword.txt中的违禁词, 再打开server.exe, 否则新增违禁词不会生效. 先打开server.exe后再打开client.exe, 否则客户端找不到服务器, 无法建立连接. 
## 创新点
在聊天室基础功能的基础上加入了违禁词过滤和聊天记录分析的功能. 
## 操作演示
见视频
# 参考文献
1. [Microsoft Learning](https://learn.microsoft.com/zh-cn/windows/win32/winsock/getting-started-with-winsock) 
2. 深入理解计算机系统 网络编程
