#Windows进程通信之匿名管道

最近操作系统上机有个题目，基本要求是用匿名管道实现父子进程之间相互通信，网上搜了一下觉得写得都差不多，并不合意，动手写了一下，觉得有几个坑点需要记一下。

实现的时候只用了必要的几个Windows API，没有MFC实现的界面，只有控制台显示。

### 用到的API

#### 匿名管道

匿名管道主要用于父子进程之间或者兄弟进程之间（通过父进程中转）的通信，功能非常朴素，比较好写。

##### 创建管道

```c
BOOL WINAPI CreatePipe (	//创建一个匿名管道，并把对管道的读写句柄赋值给传入的前两个参数
    __out PHANDLE hReadPipe,	//读句柄
    __out PHANDLE hWritePipe,	//写句柄
    __in LPSECURITY_ATTRIBUTES lpPipeAttributes,	//决定子进程能否继承管道句柄
    __in DWORD nSize	//缓冲区大小，值为0的时候使用默认的设置
); 
```

#### 创建子进程

```c
BOOL CreateProcess(
    LPCTSTR lpApplicationName,
    LPTSTR lpCommandLine,
    LPSECURITY_ATTRIBUTES lpProcessAttributes,	//安全描述符，进程句柄能否被继承
    LPSECURITY_ATTRIBUTES lpThreadAttributes,	//线程能否被继承
    BOOL bInheritHandles,	//决定子进程是否从父进程继承句柄
    DWORD dwCreationFlags,	//附加信息
    LPVOID lpEnvironment,	//环境信息
    LPCTSTR lpCurrentDirectory,
    LPSTARTUPINFO lpStartupInfo,	//启动信息
    LPPROCESS_INFORMATION lpProcessInformation	//进程信息
);
```

这个函数利用前两个参数启动某个路径下一个可执行文件作为子进程，具体用法见度娘，如果父子进程的可执行文件在同一路径下，只需要把前两个参数任意一个设置成子进程名就可以。

附加信息是一个标志，具体还是见度娘，如果想要将子进程的控制台单独显示出来可以设置为`CREAT_NEW_CONSOLE`，这同时需要在进程信息里设置窗口可见。

安全信息结构体

####启动信息

`STARTUPINFO`用于指定新进程的主窗口特性，这个结构的第一个成员为结构的大小，必须将其初始化为`sizeof(STARTUOINFO)`，其`hStdInput`和`hStdOutput`两个成员保存的是希望设定为新进程输入输出句柄的句柄。

#### 管道读写

```c
BOOL ReadFile(	//读管道
    HANDLE hFile,	//文件句柄
    LPVOID lpBuffer,	//缓冲区
    DWORD nNumberOfBytesToRead,	//要读入的字节数
    LPDWORD lpNumberOfBytesRead,	//DWORD类型的指针，指向的位置储存了实际读入的字符数
    LPOVERLAPPED lpOverlapped
    //如文件打开时指定了FILE_FLAG_OVERLAPPED，那么必须，用这个参数引用一个特殊的结构。
    //该结构定义了一次异步读取操作。否则，应将这个参数设为NULL
);
```

写管道的`WriteFile`与读管道类似。

### 实现

#### 思路

使用`CreatPipe()`创建两个匿名管道分别用于父进程向子进程和子进程向父进程发送信息，子进程由`CreatProcess()`产生，并将输入输出句柄重定向到管道，使用`ReadFile()`和`WriteFile()`读写管道实现进程的通信。

创建子进程后先由子进程向父进程发送消息，父进程接收到后再向子进程发送消息，子进程把接收到的消息发回父进程作为回显，父子进程分别为father.exe和son.exe，为了简单将他们放在同一目录下，就可以进行通信。

#### 几个问题

1. `ReadFile()`可以读取管道内容，但是当管道为空时会导致程序被挂起，所以使用了`PeekNamePipe()`预读取管道内容，该函数不会清除管道的内容，发现管道不空时再用`ReadFile()`读取管道。
2. 想要让子进程继承父进程的管道句柄，需要将结构体`SERURITY_ATTRIBUTES`的`bINheritHandle`设置为TRUE，并设置`CreatProcess()`第五个参数为TRUE，保证句柄可以继承以及创建子进程时继承这些句柄。
3. 创建子进程后，由于继承关系，子进程的标准输入输出句柄会被重定义到管道句柄上，此时子进程标准输入输出就成了读管道和写管道，不会有控制台显示。



最后贴一下代码： 

```c
//father.c
#include <stdio.h>
#include <windows.h>
#define BUFF_SIZE 100
int main (int argc, char* argv[])
{
	HANDLE hReadPipe, hWritePipe, hReadPipe2, hWritePipe2, hi, ho;
	char Buff[BUFF_SIZE];    //缓冲区
	SECURITY_ATTRIBUTES sa;  //安全信息
	STARTUPINFO si={sizeof (si)};  //启动信息，需要对长度进行初始化
	PROCESS_INFORMATION pi;  //进程信息
	long BytesRead, BytesWrite;
	sa.nLength = sizeof (sa);    //一些初始化
	sa.lpSecurityDescriptor = 0;
	sa.bInheritHandle = TRUE;    //使父进程的句柄可以被子进程继承
	si.dwFlags = STARTF_USESHOWWINDOW|STARTF_USESTDHANDLES;    //决定成员有效的标志
	si.wShowWindow = SW_HIDE;    //子进程没有控制台输入输出，直接设置窗口隐藏
	/*创建管道*/
	if (!CreatePipe (&hReadPipe, &hWritePipe, &sa, 0))     //子进程写，父进程读
	{
    	printf ("创建管道失败\n");
    	exit (1);
	}
	if (!CreatePipe (&hReadPipe2, &hWritePipe2, &sa, 0))    //子进程读，父进程写
	{
    	printf ("创建管道失败\n");
    	exit (1);
	}
	printf ("创建管道成功\n");
	/*创建子进程*/
	hi = GetStdHandle (STD_INPUT_HANDLE);
	ho = GetStdHandle (STD_OUTPUT_HANDLE);    //记录父进程原本的标准输入输出句柄，以备恢复
	SetStdHandle (STD_INPUT_HANDLE, hReadPipe2);
	SetStdHandle (STD_OUTPUT_HANDLE, hWritePipe);    //标准输入输出的句柄重定向到管道句柄，让子进程继承这些句柄
	GetStartupInfo(&si);    //获取当前进程信息
	char s[]="son.exe";
	int sta=CreateProcess (s, s, NULL, NULL, TRUE, FALSE, NULL, NULL, &si, &pi);    	//创建子进程，此处第五个参数要为TRUE保证句柄可以被子进程继承
	SetStdHandle (STD_INPUT_HANDLE, hi);
	SetStdHandle (STD_OUTPUT_HANDLE, ho);    //恢复父进程的标准输入输出句柄
	if (!sta)
	{
    	printf ("子进程创建失败\n");
    	exit (1);
	}
	else
	{
    	printf ("子进程产生成功：son.exe\n");
    	printf ("进程句柄：%d\n", pi.hProcess);
    	printf ("进程ID：%d\n", pi.dwProcessId);
	}
	/*接收消息*/
	printf ("正在等待读取子进程消息…\n");
	while (1)
	{
    	PeekNamedPipe (hReadPipe, Buff, BUFF_SIZE, &BytesRead, 0, 0);    //预读管道内容，防止直接调用ReadFile造成进程阻塞
    	if (BytesRead)
    	{
        	memset (Buff, 0, sizeof (Buff));
        	ReadFile (hReadPipe, Buff, BUFF_SIZE, &BytesRead, 0);    //读取来自子进程的消息
        	printf ("接收到来自子进程的消息\n");
        	printf ("%s", Buff);
        	break;
    	}
	}
	/*发送消息*/
	printf ("正在向子进程发送消息…\n");
	char str[]="Hello, there is your father process\n";
	while (1)
	{
    	if (WriteFile (hWritePipe2, str, strlen (str)+1, &BytesWrite, 0))    //向子进程发送消息
    	{
        	printf ("发送成功\n");
        	break;
    	}
    	else exit (1);
	}
	printf ("等待消息回显…\n");
	while (1)    //由于子进程的标准输出被重定向到了管道上，无法只管判断是否接收到消息，现将收到的消息发回以进行验证
	{
    	PeekNamedPipe (hReadPipe, Buff, BUFF_SIZE, &BytesRead, 0, 0);    //预读管道内容，防止直接调用ReadFile造成进程阻塞
    	if (BytesRead)    //管道不空
    	{
        	memset (Buff, 0, sizeof (Buff));
        	ReadFile (hReadPipe, Buff, BUFF_SIZE, &BytesRead, 0);    //读取回显信息
        	printf ("%s", Buff);
        	printf ("收到回显，通信成功\n");
        	break;
    	}
	}
	CloseHandle(hi);
	CloseHandle(ho);
	CloseHandle(hReadPipe);
	CloseHandle(hReadPipe2);
	CloseHandle(hWritePipe);
	CloseHandle(hWritePipe2);    //关闭句柄
	return 0;
}
```


```c
//son.c
#include <windows.h>
#include <stdio.h>
#define BUFF_SIZE 100

int main (int argc, char* argv[])
{
    HANDLE hReadPipe=GetStdHandle(STD_INPUT_HANDLE);    //将从父进程继承来的句柄取出
    HANDLE hWritePipe=GetStdHandle(STD_OUTPUT_HANDLE);
    /*发送消息*/
    char str[]="Hello, there is the son process.\n";
    DWORD BytesWrite;
    if (!WriteFile (hWritePipe, str, strlen(str)+1, &BytesWrite, NULL))  //向管道内传入消息
        exit (1);
    /*接收消息*/
    char Buff[BUFF_SIZE];
    DWORD BytesRead=0;
    while (1)
    {
        PeekNamedPipe (hReadPipe, Buff, BUFF_SIZE, &BytesRead, 0, 0);  //预读管道内容，防止直接调用ReadFile造成进程阻塞
        if (BytesRead)    //当管道内有内容
        {
            memset (Buff, 0, sizeof (Buff));
            ReadFile (hReadPipe, Buff, BUFF_SIZE, &BytesRead, 0);    //读取管道内容
            WriteFile (hWritePipe, Buff, BUFF_SIZE, &BytesWrite, NULL);    //将接收到的内容作为回显信息发给父进程
            break;
        }
    }
    CloseHandle(hReadPipe);
    CloseHandle(hWritePipe);    //关闭句柄
    return 0;
}
```