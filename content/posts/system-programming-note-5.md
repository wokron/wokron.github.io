+++
title = "系统编程之进程间通信"
tags = ["C", "Linux"]
categories = ["操作系统"]
series = ["系统编程笔记"]
aliases = ["/posts/c41aa3a"]
date = "2022-11-16T09:54:43+08:00"
+++
## 一、进程间通信简述
进程是程序的一次运行的动态过程，为了完成一个任务，很多进程之间需要进行通信，从而相互合作以实现需要的功能。操作系统内核中提供了进程间通信的方法，主要有以下几种：
1. 管道：
   管道是最基本的进程通信机制，可以想象成一个管道，两端分别连着 2 个进程，一个进程往里面写，一个进程从里面读。如果读或写管道的时候没有内容可供读或写，进程将被阻塞，直到有内容可供读写为止。
2. 消息队列：
   消息队列本质上在内核空间中开辟了一块内存空间，这块内存是其他进程可以访问到的，在其中使用链表的方式实现了一个队列，进程可以向该队列中发送数据块或读取数据块，从而达到进程间通信的目的。其中每个数据块包含两部分，首先是一个类型为 long 的 type，然后是具体的数据，数据块的 type 可以作为进程之间相互约定好的协议。例如一个进程发送 type 为`123`的消息，另一个进程接收 type 为`123` 的消息，后者便可确认这就是前者发送的信息，并信任该数据块中的数据。
3. 信号量：
   不同进程之间存在对资源的竞争，信号量就是用来标明可用资源的数量的数据结构，本质是为了实现多个进程之间的同步。需要注意，信号量（semaphore）与 “信号”（signal）没有关系。
4. 共享内存：
   共享内存的本质就是把两个或多个进程的虚拟地址映射到同一块物理内存。这样，一个进程通过对这块内存的读写就能被其他进程访问到，从而实现进程间通信的功能。

## 二、进程间通信操作
### （1）准备操作
#### 获取 key
```c
#include <sys/types.h>
#include <sys/ipc.h>
key_t ftok( const char * fname, int id );
```
共享内存，消息队列，信号量等进程间通信方式都需要寻找一个中间介质来进行通信。不同的介质需要用不同的信息来进行区分，这就是进程间通信的 key。`ftok()` 函数就可以生成一个唯一的 key，该函数获取一个文件路径和一个字序号，生成一个用于区分的 key。
> 注意，选择文件路径只是因为文件的编号是独有的。设置的文件路径与代码和程序并没有什么关系。

#### 命令管理进程间通信
若没有调用控制函数进行删除，则已分配的进程间通信不会自动释放。如果共享内存，消息队列，信号量在新进程执行时依旧有之前残留的信息，可能导致程序运行结果错误。可以通过 `ipcs` 和 `ipcrm` 命令进行管理。
```shell
ipcs # 显示所有进程间通信信息
ipcrm -q MsgID # 删除消息队列
ipcrm -s SemID # 删除信号量
ipcrm -m ShrID # 删除共享内存
```

### （2）消息队列
#### 消息队列获取
```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
int msgget(key_t, key, int msgflg);
```
`msgget` 函数返回获得的消息队列的标识符，出错时返回-1，错误内容存于error中。
`key` 获取消息队列的标识，取 `IPC_PRIVATE` 时会建立当前进程内的消息队列。
`msgflg` 设置消息队列的访问权限和模式，如 `0666|IPC_CREAT`。

#### 消息队列控制
```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
```
`msgctl` 函数获取和设置消息队列的属性。
`msqid` 为消息队列标识符；`cmd` 获取该函数的行为，如 `IPC_STAT` 获取消息队列的属性，`IPC_SET` 设置消息队列的属性。
`buf` 为保存消息队列属性的结构体。

#### 消息队列消息收发
```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);
```
`msgp` 获取要传递和接收的消息，它必须是一个结构体，包括一个 long 型的 mtype 和一个数组，如下
```c
struct msgbuf
{
    long mtype;
    char mtext[MAX_TEXT_LEN];
}
```
`msgsz` 是 `mtext` 的字节数；
`msgflg` 获取发送和接受信息的方法，如 `IPC_NOWAIT` `IPC_EXCEPT` `IPC_NOERROR`，设置为 0 将采取默认阻塞的方式。
`msgtyp` 设定要接收消息的 `mtype`。若值等于0则接受在队列首的消息，值大于0则接受类型等于该值的第一个消息，值小于0则接收类型小于等于该值绝对值的第一个消息。

#### 代码示例
如下程序实现客户端进程和服务器进程，通过**消息队列**进行通信。客户端进程接受用户从终端的输入，并通过 Up 消息队列将消息传递给服务器进程，然后等待服务器进程从 Down 消息队列传回消息。服务器进程从 Up 接收到消息后**将大小写字母转换**，并通过 Down 传回给客户端进程，客户端随后输出转换后的消息。（例如：客户端通过 Up 发送'linuX', 将从 Down 接收到'LINUx'）。
```c
// server.c
#include <stdio.h>
#include <unistd.h>
#include <sys/msg.h>
#include <ctype.h>
#define MAXLEN 1000
struct msgbuf
{
    long mtype;
    char mtext[1000];
};

int main()
{
    key_t upkey = ftok(".", 10);
    key_t downkey = ftok(".", 20);
    int length = sizeof(struct msgbuf) - sizeof(long);
    int upqid = msgget(upkey, 0666|IPC_CREAT);
    int downqid = msgget(downkey, 0666|IPC_CREAT);
    while (1)
    {
        struct msgbuf buf;
        msgrcv(upqid, &buf, length, 0, 0);
        for (int i = 0; buf.mtext[i] != '\0'; i++)
        {
            if (isupper(buf.mtext[i]))
                buf.mtext[i] = tolower(buf.mtext[i]);
            else if (islower(buf.mtext[i]))
                buf.mtext[i] = toupper(buf.mtext[i]);
        }
        msgsnd(downqid, &buf, length, 0);
    }
}
```
```c
// client.c
#include <stdio.h>
#include <unistd.h>
#include <sys/msg.h>
#include <string.h>
#define MAXLEN 1000
struct msgbuf
{
    long mtype;
    char mtext[MAXLEN];
};

int main()
{
    struct msgbuf buf;
    key_t upkey = ftok(".", 10);
    key_t downkey = ftok(".", 20);
    int length = sizeof(struct msgbuf) - sizeof(long);
    int upqid = msgget(upkey, 0666|IPC_CREAT);
    int downqid = msgget(downkey, 0666|IPC_CREAT);
    char text[MAXLEN];
    while (1)
    {
        printf("Enter some text:");
        scanf("%s", text);
        buf.mtype = getpid();
        strcpy(buf.mtext, text);
        msgsnd(upqid, &buf, length, 0);

        msgrcv(downqid, &buf, length, getpid(), 0);
        printf("Receive converted message:%s\n", buf.mtext);
    }
}
```

效果：
![msg](msg.png)

### （3）信号量
#### 信号量获取
```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>
int semget(key_t key, int nsems, int semflg);
```
`nsems` 指定创建的信号量的个数。也就是单个 `semid` 对应的信号量个数。

#### 信号量控制
```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>
int semctl(int semid, int semnum, int cmd, union semun arg);
```
`semnum` 表示第几个信号量，从 0 开始。`cmd` 设置要进行的操作命令。`arg` 是命令的参数，不同的命令对应不同的参数类型。

#### 信息量操作
```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>
int semop(int semid, struct sembuf *sops, unsigned nsops);
```
`sops` 设置需要进行的操作，结构体的内容如下
```c
struct sembuf  
{  
  short sem_num;  // 信号量的序号
  short sem_op;            // 对信号量的操作 
  short sem_flg;           // 操作标识
};  
```
`sem_num` 设置要进行设置的信息量序号；
`sem_op` 设置对信号量的增减，值大于0表示对信号量增加对应的值，小于0表示对信号量减小对应的值。等于0则信号量阻塞，直到信号量值为0；
`sem_flg` 设置对信号量的操作，如设置 `IPC_NOWAIT` 设置信号量操作不等待。

#### 代码示例
如下的程序实现这样的功能：一个进程创建 3 个子进程A、B、C，每个子进程都打印相同的数字串，但要求每个进程都打印完这一位数字后，才能有进程开始下一位数字的打印，并且进程打印顺序按照进程A、B、C依次打印。（**在打印的数字前加上A、B、C**以便区分）

```c
//sem.c
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/types.h>
#include <sys/sem.h>
#include <unistd.h>
#include <sys/wait.h>

void print(int num);
void P(int semid, int num);
void V(int semid, int num);

int main()
{
    pid_t pid;
    for (int i = 0; i < 3; i++)
    {
        pid = fork();
        if (pid == 0) // child
        {
            print(i);
            return 0;
        }
    }
    key_t key = ftok(".", 1);
    struct sembuf buf;
    int semid = semget(key, 3, 0666|IPC_CREAT);
    V(semid, 0);

    while (wait(NULL) > 0);
    printf("\n");
    fflush(stdout);
    return 0;
}

void P(int semid, int num)
{
    struct sembuf buf;
    buf.sem_num = num;
    buf.sem_op = -1;
    semop(semid, &buf, 1);
}

void V(int semid, int num)
{
    struct sembuf buf;
    buf.sem_num = num;
    buf.sem_op = 1;
    semop(semid, &buf, 1);
}

char getAlpha(int num)
{
    if (num == 0)
        return 'A';
    else if (num == 1)
        return 'B';
    else
        return 'C';
}

void print(int num)
{
    char stdID[] = "21371326";
    key_t key = ftok(".", 1);
    struct sembuf buf;
    int semid = semget(key, 3, 0666|IPC_CREAT);
    
    for (int i = 0; stdID[i] != '\0'; i++)
    {
        P(semid, num);
        printf("%c%c", getAlpha(num), stdID[i]);
        fflush(stdout);
        V(semid, (num+1)%3);
    }
}
```
效果：
![sem](sem.png)

### （4）共享内存
#### 共享内存获取
```c
#include <sys/ipc.h>
#include <sys/shm.h>
int shmget(key_t key, size_t size, int shmflg);
```
`size` 设置要获取的内存空间大小。

#### 共享内存控制
```c
#include <sys/ipc.h>
#include <sys/shm.h>
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```
此处类似信号量控制。

#### 共享内存映射到地址
```c
#include <sys/ipc.h>
#include <sys/shm.h>
void *shmat(int shmid, const void *shmaddr, int shmflg);
```
该函数获取一个地址，可以使用指针对共享内存进行访问。
`shmaddr` 指定共享内存的地址位置，通常设为 NULL 来让内核自动选择地址；
`shmflg` 设置内存的读写模式，如 `SHM_RDONLY` 为只读模式。

#### 共享内存断开连接
```c
#include <sys/ipc.h>
#include <sys/shm.h>
int shmdt(const void *shmaddr);
```
`shmaddr` 表示连接的共享内存的起始地址。

#### 代码示例
如下的代码将使用共享内存和信号量实现两个进程的资源共享。一个进程向共享内存写，然后终止，然后再启动一个进程从共享内存中读。

```c
// a.c
#include <stdio.h>
#include <sys/ipc.h>
#include <unistd.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <stdlib.h>
#include <time.h>

void writeNum();
void P(int semid, int num);
void V(int semid, int num);

int main()
{
    srand(time(0));

    int key = ftok(".", 1);
    int semid = semget(key, 2, 0666|IPC_CREAT);

    pid_t pid = fork();
    if (pid > 0)
    {
        V(semid, 0);
        for (int i = 0; i < 10; i++)
        {
            P(semid, 0);
            writeNum();
            V(semid, 1);
        }
    }
    else
        execl("./b", "b", NULL);
}

void writeNum()
{
    key_t key = ftok(".", 2);
    int shmid = shmget(key, sizeof(int), 0666|IPC_CREAT);
    int* p = shmat(shmid, NULL, 0);
    *p = rand();
    printf("write:  %d\n", *p);
    fflush(stdout);
}

void P(int semid, int num)
{
    struct sembuf buf;
    buf.sem_num = num;
    buf.sem_op = -1;
    semop(semid, &buf, 1);
}

void V(int semid, int num)
{
    struct sembuf buf;
    buf.sem_num = num;
    buf.sem_op = 1;
    semop(semid, &buf, 1);
}
```
```c
// b.c
#include <stdio.h>
#include <sys/ipc.h>
#include <unistd.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <stdlib.h>
#include <time.h>

void readNum();
void P(int semid, int num);
void V(int semid, int num);

int main()
{
    int key = ftok(".", 1);
    int semid = semget(key, 2, 0666|IPC_CREAT);

    for (int i = 0; i < 10; i++)
    {
        P(semid, 1);
        readNum();
        V(semid, 0);
    }
}

void readNum()
{
    key_t key = ftok(".", 2);
    int shmid = shmget(key, sizeof(int), 0666|IPC_CREAT);
    int* p = shmat(shmid, NULL, 0);
    printf("read:   %d\n\n", *p);
    fflush(stdout);
}

void P(int semid, int num)
{
    struct sembuf buf;
    buf.sem_num = num;
    buf.sem_op = -1;
    semop(semid, &buf, 1);
}

void V(int semid, int num)
{
    struct sembuf buf;
    buf.sem_num = num;
    buf.sem_op = 1;
    semop(semid, &buf, 1);
}
```
效果：
![shm](shm.png)