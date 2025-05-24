+++
title = "试试触发（几乎）所有信号"
tags = ["C", "Signal", "Linux"]
categories = ["操作系统"]
date = 2025-05-23T17:22:16+08:00
draft = true
+++

输入 `kill -L`，你可以看到 Linux 下所有可用的标准信号，总共有 31 个。

```console
$ kill -L
 1 HUP      2 INT      3 QUIT     4 ILL      5 TRAP     6 ABRT     7 BUS
 8 FPE      9 KILL    10 USR1    11 SEGV    12 USR2    13 PIPE    14 ALRM
15 TERM    16 STKFLT  17 CHLD    18 CONT    19 STOP    20 TSTP    21 TTIN
22 TTOU    23 URG     24 XCPU    25 XFSZ    26 VTALRM  27 PROF    28 WINCH
29 POLL    30 PWR     31 SYS
```

今天试试触发这些信号。当然不是指通过 `kill` 等方式，而是在这些信号原本的应用场景下触发。

## 给信号分类

信号是一种进程级别的异步通知机制，本质上是硬件和软件中断在操作系统层面的封装。信号的异步性也源自于中断的异步性。依据信号的来源和作用的不同，我们可以将 31 个信号分为三类。分别是：

- **作业控制**：用于控制进程运行状态的信号。这类信号产生自用户输入或其他进程。
- **错误处理**：由于进程执行了某些不应当的操作而产生的信号。这类信号产生自进程自身。
- **异步操作**：为了通知某些异步操作的完成而产生的信号。这类信号产生自外部设备。

> 当然 [glibc 里给出了更细致的分类](https://sourceware.org/glibc/manual/2.41/html_node/Standard-Signals.html)。不过分为三类更加简洁一些。

按照这一标准，我们列举各类所包含的信号：

- 作业控制
    - HUP（hang up）：会话终止，请求进程终止
    - INT（interrupt）：用户请求程序终止当前操作
    - QUIT（quit）：因为异常，用户请求程序终止
    - TRAP（trap）：程序被调试器控制运行
    - KILL（kill）：用户强制请求程序终止
    - USR1/USR2（user）：用户自定义的请求
    - TERM（terminate）：用户请求程序终止
    - CHLD（child）：通知父进程，子进程停止或终止
    - CONT（continue）：请求当前进程继续执行
    - STOP（stop）：请求当前进程停止执行
    - TSTP（terminal stop）：通过键盘请求当前进程停止执行
- 错误处理
    - ILL（illegal instruction）：非法的指令
    - ABRT（abort）：因为错误，程序自行调用 `abort()` 终止
    - BUS（bus）：访问设备时发生总线错误
    - FPE（float point exception）：浮点错误
    - SEGV（segment violation）：访问无效的地址
    - PIPE（pipe）：向一个已经关闭的管道/socket写入数据
    - SIGSTKFLT：“协处理器栈错误”（未使用）
    - TTIN（tty input）：在后台的进程尝试从终端读取
    - TTOU（tty output）：在后台的进程尝试向终端写入
    - XCPU（exceeds cpu limit）：进程超出了 cpu 运行时间限制
    - XFSZ（exceeds file size）：进程超出了可操作的文件大小限制
    - PWR（power）：电源将要耗尽
    - SYS（system call）：系统调用有误
- 异步操作：
    - ALRM（alarm）：定时器到时
    - URG（ugent）：socket 上存在紧急数据
    - VTALRM（virtual alarm）：虚拟定时器到时
    - PROF（profile）：性能分析定时器到时
    - WINCH：终端窗口大小变化
    - POLL：某文件描述符可以进行读写

## 作业控制

### （1）HUP

创建终端模拟器窗口、使用SSH登陆等操作都会在目标主机中创建一个终端会话。通常一个 shell 会作为这个会话的控制进程。当我们断开会话时，内核会向作为控制进程的 shell 发送 SIGHUP 信号。随后 shell 又会向其所管理的其他所有进程发送 SIGHUP 信号。SIGHUP 信号的默认处置方式是终止进程。这使得在默认情况下，断开会话将终止会话内的所有进程。

使用如下程序接收 SIGHUP 信号

```c
// handle_hup.c
#include <signal.h>
#include <stdlib.h>
#include <unistd.h>

void handle_hup(int sig) {
    write(1, "HUP\n", 4);
    exit(1);
}

int main() {
    signal(SIGHUP, handle_hup);
    while (1) {
        pause();
    }
}
```

启动一个会话，运行该程序之后退出。查看 `out.txt` 可以发现进程接收到了 SIGHUP 信号。

```console
$ ./handle_hup >out.txt &
$ exit
...
$ cat out.txt
HUP
```

### （2）INT

当一个进程在前台运行时，输入 `ctrl + c` 可以向其发送一个 SIGINT 信号。该信号的含义是终止正在运行的操作，以便用户进行输入。对于非交互式程序来说，这等同于终止进程；但对于交互式程序来说，这仅仅意味着当前的一部分操作被打断。

一个例子是 python 的 REPL 模式。我们在交互模式中输入一个死循环，之后再输入 `ctrl + c`。可以发现这时我们仅仅打断了死循环的执行，python 解释器本身并未退出。

```console
$ python3
Python 3.10.12 (main, Feb  4 2025, 14:57:36) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> while True:
...   pass
... 
^CTraceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyboardInterrupt
>>> 
```

### （3）QUIT

当一个进程在前台运行时，输入 `ctrl + \` 可以向其发送一个 SIGQUIT 信号。该信号的默认处置将终止当前进程，同时生成 core dump 文件。使用这个信号的场景可能是用户发现了当前运行的程序存在明显的异常，比如说死循环。这时终止进程，并通过 core dump 文件检查程序中的问题可能是一个好办法。

以一个简单的死循环程序作为例子。

```c
// loop.c
#include <stdio.h>

int main() {
    while (1) {
    }
    printf("Hello World\n");
    return 0;
}
```

编译，运行。随后输入 `ctrl + \` 发送 SIGQUIT 信号。

```console
$ clang loop.c -o loop -g
$ ./loop 
^\退出 (核心已转储)
```

因为本人环境中用了 `systemd-coredump`，所以这里直接使用 `coredumpctl` 就可以在 `gdb` 中载入 core dump 信息。

```console
$ coredumpctl gdb loop
...
#0  main () at main.c:4
4           while (1) {
(gdb) 
```

> 没有 core dump 文件记得调整 `ulimit`

### （4）TRAP

SIGTRAP 由 `ptrace` 所使用，用于实现断点调试、跟踪系统调用等功能。不过 `ptrace` 目前了解不多，所以先略过。

### （5）KILL

当一个进程收到 SIGKILL 指令后，该进程**必定被杀死**。除了使用 `kill -9 <pid>` 或类似操作外，没有其他产生该信号的场景。不过需要注意 SIGKILL 并不是 `kill` 命令默认发出的信号（SIGTERM 才是）。因为 SIGKILL 无法被捕获，当收到 SIGKILL 信号时，进程无法做任何额外工作，只能直接被杀死。这可能导致一些资源未能正确释放。

下面的程序尝试捕获 SIGKILL 信号。

```c
// try_handle_kill.c
#include <signal.h>
#include <stdlib.h>
#include <unistd.h>

void handle_kill(int signum) {
    write(1, "KILL\n", 5);
    exit(0);
}

int main() {
    signal(SIGKILL, handle_kill);

    while (1) {
        pause();
    }
    return 0;
}
```

我们运行该程序，并向其发送 SIGKILL 信号。可以发现并未执行 `handle_kill()` 函数。

```console
$ ./try_handle_kill &
[1] 12345
$ kill -KILL $!
$ 
[1]+  已杀死               ./try_handle_kill
```

### （6）USR1/USR2

SIGUSR1 和 SIGUSR2 是为用户预留的信号，用户可以自定义这些信号的含义。不过相较于使用这两个标准信号，使用实时信号（32-63）或许更好。

SIGUSR1 和 SIGUSR2 的默认处置方式是终止进程，这可能是为了防止用户勿用未自定义的 SIGUSR1/SIGUSR2 信号。

```console
$ ./loop &
[1] 12345
$ kill -USR1 14475
$ 
[1]+  用户自定义信号 1 ./loop
```

### （7）TERM

SIGTERM 是 `kill` 命令默认发送的信号。不同于 SIGKILL，程序可以捕获 SIGTERM 信号，并在退出之前执行一些额外的处理操作；或者也可以暂时屏蔽 SIGTERM 信号，避免重要操作被信号打断。这一点十分重要，例如数据库在写入时需要保证数据的完整性，如果使用 SIGKILL 打断这一操作可能导致数据的损坏。

下面程序在重要操作前屏蔽了 SIGKILL 和 SIGTERM 信号。

```c
// test_mask.c
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

void do_something_important() { sleep(2); }

int main() {
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGTERM);
    sigaddset(&mask, SIGKILL);
    sigprocmask(SIG_BLOCK, &mask, NULL);

    do_something_important();
    printf("Done\n");

    sigprocmask(SIG_UNBLOCK, &mask, NULL);

    while (1) {
        pause();
    }

    return 0;
}
```

如果在运行程序之后发送 SIGTERM 信号，程序会在重要操作完成后才处理该信号

```c
$ ./test_mask & sleep 1; kill $! & wait
[1] 123456
[2] 654321
Done
[1]-  已终止               ./test_mask
[2]+  已完成               kill $!
```

而如果发送 SIGKILL，则程序会立即终止，尽管我们“屏蔽”了 SIGKILL。

```c
$ ./test_mask & sleep 1; kill -9 $! & wait
[1] 123456
[2] 654321
[1]-  已杀死               ./test_mask
```

### （8）CHLD

当一个子进程**终止**、**暂停**或**继续运行**时，父进程会收到一个 SIGCHLD 信号。

这一信号在某些情况下很有用，比如子进程的生命周期并不确定的情况。这时子进程随时可能终止，而父进程却由于有自己的任务而不能时刻调用 `wait()` 回收僵尸子进程。通过将子进程终止的情况通过信号的方式传递给父进程，父进程就可以在自己的程序执行之外完成对子进程的处理。

> 这一点类似于被 detach 的线程。不过线程可以在结束时自动释放自己的资源，而进程却只有通过父进程才能实现彻底的回收。

> 注意使用 SIGCHLD 时不需要担心父进程率先终止的情况，因为如果父进程率先终止，则子进程的父进程会自动变为 init 进程（pid=1）。而 init 进程总会进行僵尸进程的回收。
> > 这一点类似于内存泄露。在进程持续运行的时候才是问题，如果进程已经终止了，那么其资源也会随之释放。

下面程序创建了两个子进程，并且注册了 SIGCHLD 信号的处理函数。在其中我们**循环**调用 `waitpid()`，直到没有新的僵尸子进程被回收。

```c
// test_child.c
#include <signal.h>
#include <sys/wait.h>
#include <unistd.h>

void handle_child(int sig) {
    write(1, "CHLD\n", 5);
    while (waitpid(-1, NULL, WNOHANG) > 0) {
        write(1, "WAIT\n", 5);
    }
}

int main() {
    signal(SIGCHLD, handle_child);

    for (int i = 0; i < 2; i++) {
        int pid = fork();
        if (pid == 0) {
            while (1) {
                pause();
            }
            return 0;
        }
    }

    while (1) {
        pause();
    }

    return 0;
}
```

我们运行这个程序。对于两个子进程，我们终止第一个子进程，暂停第二个子进程，之后再让其恢复运行。

```console
$ ./test_child &
$ ps
    PID TTY          TIME CMD
  12345 pts/0    00:00:00 bash
 123001 pts/0    00:00:00 test_child
 123002 pts/0    00:00:00 test_child
 123003 pts/0    00:00:00 test_child
 123456 pts/0    00:00:00 ps
$ kill 123002
CHLD
WAIT
$ ps
    PID TTY          TIME CMD
  12345 pts/0    00:00:00 bash
 123001 pts/0    00:00:00 test_child
 123003 pts/0    00:00:00 test_child
 123456 pts/0    00:00:00 ps
$ kill -STOP 123003
CHLD
$ kill -CONT 123003
CHLD
```

### （9）CONT

SIGCONT 信号用于使一个**被暂停**的进程**恢复运行**。如果进程没有被暂停，则 SIGCONT 不会有任何影响。用户可以注册该信号的处理函数，这样当进程再度运行时会首先执行处理函数中的代码。

想要恢复运行，首先需要暂停执行。所以 SIGCONT 需要与 SIGSTOP 或 SIGTSTP 配合使用。

下面程序处理 SIGCONT 信号。

```c
// test_cont.c
#include <signal.h>
#include <unistd.h>

void handle_cont(int signum) { write(1, "CONT\n", 5); }

int main() {
    signal(SIGCONT, handle_cont);

    while (1) {
        pause();
    }
    return 0;
}
```

先暂停进程运行，之后发送 SIGCONT 使程序恢复运行。这里需要注意的是，如果进程已经在运行，发送 SIGCONT 依然会调用信号处理函数。

```console
$ ./test_cont &
[1] 123456
$ kill -STOP $!

[1]+  已停止               ./test_cont
$ kill -CONT $!
CONT
$ jobs
[1]+  运行中               ./test_cont &
$ kill -CONT $!
CONT
```

### （10）STOP

SIGSTOP 信号会使进程暂停。和 SIGKILL 类似，SIGSTOP 也不可被忽略或屏蔽，一旦进程收到该信号则必定暂停。

SIGSTOP 只能通过 `kill` 等方式触发。在前面我们已经使用过多次了，所以这里略过。

### （11）TSTP

SIGTSTP 信号的默认处置行为和 SIGSTOP 相同。进程在收到该信号后会暂停运行。不过该信号可以被忽略或屏蔽，其与 SIGSTOP 的关系类似于 SIGTERM 与 SIGKILL。

对于在前台运行的进程，可以通过 `ctrl + z` 来向该进程发送一个 SIGTSTP 信号。一般情况下这会使得前台进程暂停运行，shell 重新成为前台进程。这样的前后台的切换是**作业控制**的重要功能，shell 可以据此让许多不同作业在前后台并发执行。

这里我们用前几节中的死循环程序为例。我们首先运行该程序，运行一段时间后，我们输入 `ctrl + z` 暂停该程序。输入 `jobs` 指令查看该作业已暂停。

```console
$ ./loop 
^Z
[1]+  已停止               ./loop
$ jobs 
[1]+  已停止               ./loop
```

随后我们后台运行另一个程序。

```console
$ ./loop &
[2] 257314
$ jobs 
[1]+  已停止               ./loop
[2]-  运行中               ./loop &
```

通过 `bg` 指令使之前暂停的进程在后台运行

```console
$ bg %1
[1]+ ./loop &
$ jobs 
[1]-  运行中               ./loop &
[2]+  运行中               ./loop &
```

通过 `fg` 指令使原本在后台运行的进程在前台运行。之后再通过 SIGINT 使其停止。

```console
$ fg %2
./loop
^C
```

最后通过 `kill` 停止正在后台运行的进程

```console
$ kill %1
$ 
[1]+  已终止               ./loop
```

## 错误处理

### （1）ILL

当程序运行过程中出现处理器无法识别的指令时，就会触发 SIGILL 信号。产生该信号的方式也很简单。使用如下代码即可：

```c
// test_ill.c
int main() {
    asm(".byte 0x0f, 0x00");
    return 0;
}
```

这段代码可以正常编译，但是运行时就会触发 SIGILL 信号。由于该信号意味着程序存在错误，所以默认处置会产生 core dump 文件。

```console
$ ./test_ill
非法指令 (核心已转储)
```

由于大部分情况下程序的机器码是由编译器或汇编器翻译而来的，所以这一信号并不常见。即便是错误地将 pc 指向了数据段部分，也会由于数据段没有执行权限而不会出现 SIGILL 信号。例如下面的代码

```c
// invalid_call.c
typedef void (*func_t)(void);

const char a = 0;

int main() {
    func_t f = (func_t)&a;
    f();
    return 0;
}
```

运行之后并不会触发 SIGILL 信号，而是 SIGSEGV 信号

```console
$ ./invalid_call
段错误 (核心已转储)
```

### （2）ABRT

当程序发现自身出现了无法恢复的错误时，可调用 `abort()` 异常终止当前进程。这时该进程会收到一个 SIGABRT 信号。该信号的默认处置会终止当前进程，同时产生 core dump 文件。例如下面的程序。

```c
// test_abort.c
#include <stdlib.h>
#include <unistd.h>

int factorial(int n) {
    if (n < 0) {
        abort();
    } else if (n == 0) {
        return 1;
    } else {
        return n * factorial(n - 1);
    }
}

int main() {
    int n = -1;
    return factorial(n);
}
```

运行之后，调用 `abort()` 触发 SIGABRT 信号，进程终止。

```console
$ ./test_abort
已中止 (核心已转储)
```

标准库中 `assert` 的实现中同样调用了 `abort()` 函数。

### （3）BUS

SIGBUS 是由于不正确的方式访问设备而产生的信号。一个比较常见的产生该信号的场景是使用 `mmap()` 时访问了文件范围外的地址。例如下面的例子。

```c
// test_bus.c
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>
#include <unistd.h>

int main() {
    int fd = open("test.txt", O_RDWR | O_CREAT | O_EXCL, S_IRUSR | S_IWUSR);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    void *addr = mmap(NULL, 1, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    *(char *)addr = 0;

    close(fd);
    return 0;
}
```

这里我们创建了一个新文件 `test.txt`。此时文件大小为 0。我们使用 `mmap()` 映射该文件的第一个字节的数据（因为页，实际上会对齐到页面大小）。可由于文件大小为 0，所以对应地址上的数据应当是无效的。

执行该程序的结果如下。我们访问的位置并没有对应的磁盘数据，所以触发了一个 SIGBUS 信号。

```console
$ ./test_bus
总线错误 (核心已转储)
```

### （4）FPE

SIGFPE 信号表示浮点异常。会在整数除零等情况下出现。产生该信号的代码很简单。

```c
// test_fpe.c
int main() {
    int x = 1;
    int y = 0;
    return x / y;
}
```

运行该程序的结果如下。

```console
$ ./test_fpe
浮点异常 (核心已转储)
```

需要注意的是浮点数除零并不会产生 SIGFPE 信号。因为 IEEE 754 中定义了 inf。

### （5）SEGV

SIGSEGV 应该是最常出现的表达错误的信号。当进程所访问的地址对应页面不存在或没有权限时就会产生这一信号。例如下面的程序。

```c
// test_segv.c
int main() {
    *(char*)1 = 0;
    return 0;
}
```

```console
$ ./test_segv
段错误 (核心已转储)
```

需要注意 SIGSEGV 并不只会出现在地址无效时。例如 ILL 一节虽然访问的地址有效，但是由于没有可执行权限，依然产生了 SIGSEGV 信号。

### （6）PIPE

当一个进程尝试写入一个对端已经关闭的管道或 socket 时，就会产生 SIGPIPE 信号。该信号的**默认处置会终止当前进程的执行**。

下面的程序创建了一个管道，并在第二次写入之前关闭了读端。

```c
// test_pipe.c
#include <stdio.h>
#include <unistd.h>

int main() {
    int pipefd[2];
    pipe(pipefd);

    printf("Before write (1)\n");
    write(pipefd[1], "Hello, World!", 13);
    printf("After write (1)\n");

    close(pipefd[0]);

    printf("Before write (2)\n");
    write(pipefd[1], "Hello, World!", 13);
    printf("After write (2)\n");

    return 0;
}
```

接下来运行该程序，发现只输出了前三行内容。

```console
$ ./test_pipe
Before write (1)
After write (1)
Before write (2)
```

查看进程的返回值，为 141（128 + 13）。说明该进程是由于第 13 号信号 SIGPIPE 导致终止的。

```console
$ echo $?
141
```

对于 socket 我们也能举一个例子。下面是一个 server 端的代码。这个程序接受一个连接请求，并以一秒为间隔不断向对端写入数据。

```c
// server.c
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <unistd.h>

int main() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        exit(1);
    }

    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = 0;

    if (bind(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) <
        0) {
        perror("bind");
        close(sockfd);
        exit(1);
    }

    socklen_t addr_len = sizeof(server_addr);
    if (getsockname(sockfd, (struct sockaddr *)&server_addr, &addr_len) < 0) {
        perror("getsockname");
        close(sockfd);
        exit(1);
    }
    printf("Socket bound to port %d\n", ntohs(server_addr.sin_port));

    if (listen(sockfd, 5) < 0) {
        perror("listen");
        close(sockfd);
        exit(1);
    }

    int client_sockfd = accept(sockfd, NULL, NULL);
    if (client_sockfd < 0) {
        perror("accept");
        close(sockfd);
        exit(1);
    }
    printf("Accepted a connection\n");

    char buffer[] = "Hello, World!\n";

    while (1) {
        if (send(client_sockfd, buffer, sizeof(buffer), 0) < 0) {
            perror("send");
            close(client_sockfd);
            close(sockfd);
            exit(1);
        }
        printf("Sent message to client\n");
        sleep(1);
    }

    return 0;
}
```

我们运行这个程序，并用 `netcat` 接收 server 发送的信息。一段时间后使用 SIGINT 终止 `netcat` 的运行。结果如下：

**server**：

```console
$ ./server 
Socket bound to port 54321
Accepted a connection
Sent message to client
Sent message to client
Sent message to client
Sent message to client
Sent message to client
Sent message to client
$ echo $?
141
```

**client**：

```console
$ netcat 0.0.0.0 54321
Hello, World!
Hello, World!
Hello, World!
Hello, World!
Hello, World!
^C
```

可以发现出现了这样的情况：**client 程序关闭了连接，但 server 程序却因此而终止**。对于可用性有较高要求的 server 端来说，这样的行为可难以接受。

所以对于使用 socket 的程序，尤其是 server 程序，需要记得对 SIGPIPE 进行额外的处理。例如可以选择将忽略 SIGPIPE 信号，或者可以在 `send()` 调用时增加 `MSG_NOSIGNAL` 来避免这一问题。在设置了上述选项后，出现读端关闭的情况将会使得 `write()`、`send()` 等函数错误返回，并将 `errno` 设置为 `EPIPE`。

> 还有一个对偶的情况：当读取数据时写端关闭。这种情况的判断条件是 `read()` 等函数正常返回，且返回的读取字节数为 0。因为被定义为正常情况，所以就不需要信号或错误码了。

### （7）TTIN

如果一个进程是后台进程，且该进程希望从终端输入中读取内容，那么就会触发 SIGTTIN 信号。因为这时候的用户的输入是针对于前台进程而言的。不过不同于错误处理分类里的其他信号，TTIN（以及 TTOU）的默认处置是**暂停当前进程**。因为虽然用户现在不需要与触发 SIGTTIN 的进程交互，但是之后很可能将该进程设置为前台进程。

下面的程序煎蛋地读入一个数组，再将该数字输出。

```c
// iecho.c
#include <stdio.h>

int main() {
    int n;
    scanf("%d", &n);
    printf("%d\n", n);
    return 0;
}
```

将该程序置于后台运行时，进程很快暂停运行。之后我们可以通过 `fg` 命令将其转到前台，并与其交互。

```console
$ ./iecho &
[1] 123456
$ 
[1]+  已停止               ./iecho
$ jobs 
[1]+  已停止               ./iecho
$ fg
./iecho
1
1
```

### （8）TTOU

SIGTTOU 和 SIGTTIN 类似。当位于后台的进程尝试向终端写入内容时会产生该信号。在默认的设置下，这个信号并不会产生，后台进程也会向终端输出内容。只有当使用 `stty tostop` 命令明确禁止后台进程输出后该信号才会启用。

还用上一节的 `iecho` 程序进行测试。不增加 `tostop` 配置时，后台进程依然向前台输出。

```console
$ echo 1 | ./a.out & wait
[1] 123456
1
[1]+  已完成               echo 123 | ./a.out
```

但增加 `tostop` 配置后，相同的程序现在会导致进程暂停。

```console
$ stty tostop
$ echo 123 | ./a.out & wait
[1] 524975

[1]+  已停止               echo 123 | ./a.out
bash: wait: 警告： job 1[524975] stopped
$ fg
echo 123 | ./a.out
123
```

### （9）XCPU

使用 `setrlimit()` 可以对单个进程所使用的各项资源进行限制。这些资源包括虚拟内存大小、CPU执行时间、文件大小等等。当进程运行过程中对资源的请求超出其限制时，会在对应时刻产生异常。对某些限制的检查是在调用接口后同步进行的，这时异常会使用错误码等方式传递；而另一些检查是随着程序的执行异步进行的。这些异步产生的异常就使用信号进行传递。

> `rlimit` 的作用范围是单个进程。和 `rlimit` 较为相关的另一个概念是 `cgroup`。`cgroup` 的作用范围是一个进程组。

SIGCPU 表达的异常情况是“进程的运行时间超过了预先设置的CPU执行时间限制”。如下代码先将 CPU 执行时间设置为 1 秒。之后执行死循环造成超时。

```c
// test_xcpu.c
#include <stdio.h>
#include <sys/resource.h>

int main() {
    // change the CPU time limit to 1 seconds
    struct rlimit rl = {.rlim_cur = 1, .rlim_max = RLIM_INFINITY};
    if (setrlimit(RLIMIT_CPU, &rl) == -1) {
        perror("setrlimit");
        return 1;
    }
    printf("CPU time limit: %ld seconds\n", rl.rlim_cur);

    while (1)
        ;
    return 0;
}
```

运行该程序并记录其执行时间，可以看到大约一秒之后触发 SIGXCPU 信号。默认处置条件是终止进程并生成 core dump 文件。

```console
$ time ./test_xcpu
CPU time limit: 1 seconds
CPU 时间超出限制 (核心已转储)

real    0m1.087s
user    0m0.993s
sys     0m0.005s
```

### （10）XFSZ

另一个进程资源限制相关的信号是 SIGXFSZ。`setrlimit()` 可以设置当前进程所能创建的文件的最大大小。超出该限制将触发 SIGXFSZ 信号。默认处置方式是终止程序并产生 core dump 文件。

如下代码将最大大小设置为 1 字节，之后尝试创建文件并写入超过 1 字节的数据。

```c
// test_xfsz.c
#include <stdio.h>
#include <sys/resource.h>

int main() {
    struct rlimit rl = {.rlim_cur = 1, .rlim_max = RLIM_INFINITY};
    if (setrlimit(RLIMIT_FSIZE, &rl) == -1) {
        perror("setrlimit");
        return 1;
    }
    printf("File size limit changed to: %ld bytes\n", rl.rlim_cur);

    // try to create a file larger than the limit
    FILE *fp = fopen("test.txt", "w");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }
    printf("Writing to file...\n");
    fprintf(fp, "Hello, World!\n");
    fclose(fp);

    return 0;
}
```

运行结果显示当试图写入数据时产生了 SIGXFSZ 信号。

```console
$ ./test_xfsz
File size limit changed to: 1 bytes
Writing to file...
文件大小超出限制 (核心已转储)
```

### （11）PWR

SIGPWR 会在电源即将耗尽时被发送给 init 进程。init 进程会负责在这一时刻完成整个系统的有序退出。此信号的触发条件有些苛刻，因此这里略过。

### （12）SYS

SIGSYS 会在进程发起的系统调用有误的时候触发。不过我没能构造出这样的程序。使用错误的系统调用号以及错误的参数只会让进程直接退出。

## 异步操作

### （1）ALRM/VTALRM/PROF



### （2）URG

当使用 socket 传输紧急数据时会产生一个 SIGURG 信号。不过紧急数据在当下几乎没有使用，所以这里略过。

### （5）WINCH

当用户改变终端窗口大小时会触发一个 SIGWINCH 信号。窗口大小可能影响一些终端应用程序（比如说 vim）的格式化结果。当收到 SIGWINCH 信号后，这样的程序可以根据新的终端窗口大小对显示结果进行重绘。

使用如下程序捕获 SIGWINCH 信号。

```c
// test_winch.c
#include <signal.h>
#include <unistd.h>

void handle_winch(int sig) { write(1, "WINCH\n", 6); }

int main() {
    signal(SIGWINCH, handle_winch);
    while (1) {
        pause();
    }
}
```

新建一个终端窗口，运行该程序。之后调整窗口的大小，可以发现产生了一系列 SIGWINCH 信号。

```console
$ ./test_winch
WINCH
WINCH
WINCH
WINCH
WINCH
WINCH
WINCH
WINCH
```

### （6）POLL

SIGPOLL 用于实现信号驱动 IO。不过基于信号处理机制进行 IO 实现起来太过复杂了，同时相较于多路复用没有明显的性能优势，所以似乎用处不大。这里略过。