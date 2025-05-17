+++
title = "Linux 进程的内存管理"
tags = ["Linux", "C"]
categories = ["操作系统"]
date = 2025-05-17T09:33:11+08:00
+++

虽然我们都学过一个进程的内存由堆和栈组成。但是这样的模型还是太抽象了，其中掩盖了许多操作系统的细节。所以这里简单梳理一下进程的内存管理有关知识。

## 堆的增长

libc 中提供 `malloc` 函数申请堆上内存。底层由 `brk` 系统调用负责申请堆上内存。

在内核的视角下，堆空间是一个简单的结构。它由一个固定的堆底（符号 `end`）和可变的堆顶（称为 program break）组成。内核所要做的就是根据用户设定的 program break 将堆底和堆顶之间的内存标为有效。而 `brk` 系统调用的作用便是将某一地址设置为堆顶。

在 `brk` 系统调用之上，glibc 提供了两个不同的函数 `int brk(void *addr)` 和 `void *sbrk(intptr_t increment)`。前者直接设置 program break 地址，后者则根据 `increment` 取值调整 program break 位置。

下面使用 `sbrk()` 函数进行内存分配。`sbrk()` 会返回调用之前原本的 program break 地址，因此还需额外进行加减以获取当前地址。这一语义比较合理，因为这样 `brk(N)` 和 `malloc(N)` 的返回值都代表新分配的内存的起始地址。

```c
#include <stdio.h>
#include <unistd.h>

extern char end;

int main() {
    printf("Address of end symbol: %p\n", (void *)&end);
    printf("Current program break: %p\n", sbrk(0));
    printf("New program break:     %p\n", sbrk(1024) + 1024);
    printf("After deallocation:    %p\n", sbrk(-512) - 512);
}
```

运行结果如下

```console
Address of end symbol: 0x55b76abe2040
Current program break: 0x55b77f1aa000
New program break:     0x55b77f1aa400
After deallocation:    0x55b77f1aa200
```

可见有 `end <= program break`。此处两者不同的原因是在 `main()` 之前 glibc 已经使用了堆内存。

另外，虽然 program break 的地址不一定按页对齐，但是堆空间依然是按页分配的（当然了）。这意味着在堆顶之外进行访存可能并不会造成 `SIGSEGV`。这一点容易造成隐性 bug。

如下程序将 program break 设置为按页对齐再加 1 字节的位置。这时这一页的内存都可以被访问。只在超过一页范围后才会触发 `SIGSEGV`。

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    const size_t PAGE_SIZE = sysconf(_SC_PAGESIZE);

    void *origin_ptr = sbrk(0);
    size_t origin_ptr_size = (size_t)origin_ptr;
    size_t ptr_size_up_aligned =
        (origin_ptr_size + PAGE_SIZE - 1) & ~(PAGE_SIZE - 1);
    void *ptr_up_aligned = (void *)ptr_size_up_aligned;

    // Allocate 1 byte more
    brk(ptr_up_aligned + 1);

    char *ptr = (char *)ptr_up_aligned;

    for (int i = 0; i < PAGE_SIZE; i++) {
        ptr[i] = 1;
    }
    write(1, "ok\n", 3);
    
    ptr[PAGE_SIZE] = 2;
    write(1, "not ok\n", 9);
}
```

运行结果如下

```console
ok
fish: Job 1, './a.out' terminated by signal SIGSEGV (Address boundary error)
```

还需注意 `brk` 所申请的堆空间并未全部映射到物理内存（也是当然），物理内存的分配只发生在实际访存时。下面代码申请 `total` 页堆内存，并访问其中 `access` 页。我们运行程序，通过 `perf` 统计 page-fault 次数。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <total> <access>\n", argv[0]);
        exit(1);
    }

    int total = atoi(argv[1]);
    int access = atoi(argv[2]);

    if (access > total) {
        fprintf(stderr, "Access exceeds allocated memory\n");
        exit(1);
    }

    const size_t PAGE_SIZE = sysconf(_SC_PAGESIZE);
    char *ptr = sbrk(total * PAGE_SIZE);

    for (int i = 0; i < access; i++) {
        ptr[i * PAGE_SIZE] = 1;
    }
    return 0;
}
```

结果如下，可以看到 page-faults 随着访存数量线性增长。这说明 `brk` 不分配物理内存，而只是在页表中将对应项标为可用。

```console
$ sudo perf stat -e 'page-faults' -- ./test_page_fault 10000 0

 Performance counter stats for './test_page_fault 10000 0':

                54      page-faults
 ...
$ sudo perf stat -e 'page-faults' -- ./test_page_fault 10000 10

 Performance counter stats for './test_page_fault 10000 10':

                63      page-faults
 ...
$ sudo perf stat -e 'page-faults' -- ./test_page_fault 10000 100

 Performance counter stats for './test_page_fault 10000 100':

               154      page-faults
 ...
$ sudo perf stat -e 'page-faults' -- ./test_page_fault 10000 1000

 Performance counter stats for './test_page_fault 10000 1000':

             1,054      page-faults
 ...
$ sudo perf stat -e 'page-faults' -- ./test_page_fault 10000 10000

 Performance counter stats for './test_page_fault 10000 10000':

            10,054      page-faults
 ...
```

## `malloc` 内存管理

如果只是需要申请堆内存，那么 `sbrk()` 和 `malloc()` 没有什么区别。真正的区别在于 `free()` 函数上。`malloc` 的机制实现了内存的回收重用，减缓了内存使用量的增长、避免了内存泄露，同时减少了 `brk` 系统调用的次数。

考虑 `malloc` 的实现是一个有意思的事。因为 `malloc` 的作用是管理动态增长的堆内存，为此需要维护一套动态增长的数据结构，这些数据结构又需要堆内存本身来存储。所以 `malloc` 不仅需要管理用户的内存申请，还需要管理自身的内存申请。

glibc 的实现会在用户数据之前存储内存块数据。这样在调用 `free()` 函数时就可以得知要释放的内存块大小。

```
     chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             Size of previous chunk, if unallocated (P clear)  |
             +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             Size of chunk, in bytes                     |A|M|P|
       mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             User data starts here...                          .
             .                                                               .
             .             (malloc_usable_size() bytes)                      .
             .                                                               |
 nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             (size of chunk, but used for application data)    |
             +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             Size of next chunk, in bytes                |A|0|1|
             +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

对于空闲的内存块，则会将原本用于存储用户数据的部分用来维护空闲内存块链表。这样当用户调用 `malloc()` 请求分配内存时，便可以通过空闲链表选择适当的内存块进行分配。

```
     chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             Size of previous chunk, if unallocated (P clear)  |
             +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     `head:' |             Size of chunk, in bytes                     |A|0|P|
       mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             Forward pointer to next chunk in list             |
             +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             Back pointer to previous chunk in list            |
             +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             Unused space (may be 0 bytes long)                .
             .                                                               .
             .                                                               |
 nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     `foot:' |             Size of chunk, in bytes                           |
             +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
             |             Size of next chunk, in bytes                |A|0|0|
             +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

> 上述 ascii 图源自 [glibc/malloc/malloc.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=malloc/malloc.c;h=fe56a631bce6f5271d94096d18c2bef2c5fd0050;hb=refs/heads/master)。

通过在相同数据结构上同时维护用户数据和 `malloc` 数据，`malloc()` 实现了灵活的堆内存分配。不过如果用户程序未能合理使用内存，造成越界访问的话，也很容易破坏 `malloc` 的数据结构，使得程序出现漏洞或直接崩溃。

下面的程序尝试修改 `malloc_chunk` 中数据

```c
#include <stdlib.h>

struct malloc_chunk {
    size_t mchunk_prev_size; /* Size of previous chunk (if free).  */
    size_t mchunk_size;      /* Size in bytes, including overhead. */

    struct malloc_chunk *fd; /* double links -- used only if free. */
    struct malloc_chunk *bk;

    /* Only used for large blocks: pointer to next larger size.  */
    struct malloc_chunk *fd_nextsize; /* double links -- used only if free. */
    struct malloc_chunk *bk_nextsize;
};

struct malloc_chunk *container_of(void *p) {
    size_t off = (size_t)(&((struct malloc_chunk *)NULL)->fd);
    return (struct malloc_chunk *)(p - off);
}

int main() {
    void *ptr = malloc(32);
    struct malloc_chunk *chunk = container_of(ptr);
    chunk->mchunk_size = 1;
    
    free(ptr);
    return 0;
}
```

> `struct malloc_chunk` 源自 [glibc/malloc/malloc.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=malloc/malloc.c;h=fe56a631bce6f5271d94096d18c2bef2c5fd0050;hb=refs/heads/master)。


运行结果如下。`free()` 函数中调用 `abort()` 终止了程序运行。

```console
free(): invalid pointer
fish: Job 1, './a.out' terminated by signal SIGABRT (Abort)
```

如果修改空闲链表中的数据，同样会导致程序崩溃。下面的程序在调用 `free()` 之后还对指针对应数据进行了修改。这一行为覆盖了空闲链表中的前后指针数据。

```c
#include <stdlib.h>
#include <string.h>

int main() {
    void *a = malloc(32);
    void *b = malloc(32);
    free(a);
    free(b);

    memset(b, 1, 32);

    b = malloc(32);
    a = malloc(32);

    return 0;
}
```

运行结果如下。再次调用 `malloc()` 后，指针 `b` 所对应的 `malloc_chunk` 项中的指针指向了无效的地址，因此程序崩溃。

```console
malloc(): unaligned tcache chunk detected
fish: Job 1, './a.out' terminated by signal SIGABRT (Abort)
```

> 一个好习惯是在调用 `free()` 之后将指针置为空。这样程序运行到 `memset()` 时就会出错，而不是 `a = malloc(32)` 处。

## 栈上内存分配

通常来说，栈的大小在编译期确定。但是这一点也有例外。使用 `alloca()` 函数可以在栈上分配动态大小的空间。

> LLVM 中的 `alloca` 指令或许与此有某种渊源。

下面的程序给出了 `alloca()` 的使用例子。`alloca()` 在行为上实际可以等价于变长数组 `T arr[N]`。 

```c
#include <alloca.h>
#include <stdio.h>

int main() {
    int n;
    scanf("%d", &n);

    int *arr = alloca(sizeof(int) * n);
    for (int i = 0; i < n; i++) {
        scanf("%d", &arr[i]);
    }
    
    for (int i = n - 1; i >= 0; i--) {
        printf("%d ", arr[i]);
    }
    printf("\n");
    return 0;
}
```

`alloca()` 是通过编译器开孔实现的。其背后是 `__builtin_alloca`。我们通过一个例子查看其实现原理。

```c
// alloca_example.c
#include <alloca.h>

void func(int m, int n) {
    for (int i = 0; i < m; i++) {
        int *ptr = alloca(sizeof(int) * n);
    }
}
```

使用 clang 编译，以 riscv64 为后端（我只看得懂这个……）。

```console
$ clang -S alloca_example.c --target=riscv64-linux-gnu
```

得到如下汇编码，注释标出了需要关注的内容。在栈帧内，当需要申请额外栈内存时，sp 寄存器向低地址移动，并将移动后地址作为 `alloca()` 的返回值。而 fp 寄存器则在此过程中保持不变，局部变量的寻址也基于 fp 寄存器进行。在函数返回时，同样根据 fp 寄存器的地址将 sp 寄存器重置为最开始的取值。

```asm
func:
        addi    sp, sp, -48
        sd      ra, 40(sp)
        sd      s0, 32(sp)
        addi    s0, sp, 48 # s0 即 fp，因为 fp 在栈帧内不会变化，所以后面使用 fp 寻址
        sw      a0, -20(s0)
        sw      a1, -24(s0)
        li      a0, 0
        sw      a0, -28(s0)
        j       .LBB0_1
.LBB0_1:
        lw      a0, -28(s0)
        lw      a1, -20(s0)
        bge     a0, a1, .LBB0_4
        j       .LBB0_2
.LBB0_2:
        lw      a0, -24(s0)
        slli    a0, a0, 2
        addi    a0, a0, 15
        andi    a1, a0, -16
        mv      a0, sp
        sub     a0, a0, a1 # sp 向下移动，栈帧增长
        mv      sp, a0
        sd      a0, -40(s0)
        j       .LBB0_3
.LBB0_3:
        lw      a0, -28(s0)
        addiw   a0, a0, 1
        sw      a0, -28(s0)
        j       .LBB0_1
.LBB0_4:
        addi    sp, s0, -48 # 根据 fp 寄存器移动 sp
        ld      ra, 40(sp)
        ld      s0, 32(sp)
        addi    sp, sp, 48
        ret
```

需要注意源代码中 `alloca()` 是位于循环中的，翻译为汇编码后 sp 寄存器的移动依然处于循环之中。这意味着在一次循环结束后并不会释放当次 `alloca()` 所申请的内存。只有当从函数返回时才会将所有 `alloca()` 申请的内存释放。

从汇编中可以看出这一过程中 fp 寄存器的作用是十分关键的。因为这一寄存器记录了当前栈帧的底部位置。一般来说当编译器启用优化时（`-Oxx`）都会包含“消除帧指针”优化。不过当此处我们启用这一优化时，会发现使用 `alloca()` 
的函数依然会保留 fp 寄存器。

```console
$ clang -S alloca_example.c --target=riscv64-linux-gnu -fomit-frame-pointer
```

> 虽然“消除帧指针”优化通常都会开启，但还是关闭比较好。

虽然说了这么多，不过实际上 `alloca()` 的用处很小。因为每个线程的栈空间有限，通常为 4KB，这使得无法在栈上申请较大的空间。另外，就算扩充栈的大小，`alloca()` 也没有太大用处。因为其生命周期是和栈帧绑定的，当函数返回时，`alloca()` 也自动释放。如果想要延长这些数据的生命周期，只能通过复制的方式实现，而不能通过指针。

## 内存映射区

我们都说内存分为堆和栈。那么 `mmap()` 申请的内存属于哪里呢？这部分内存肯定不属于栈，但其实也不属于堆。`mmap()` 所申请的内存属于堆和栈之外的内存映射区。

内存映射区位于堆和栈的中间，高于内核常量 `TASK_UNMAPPED_BASE` 的位置。和 `malloc()` 不同，`mmap()` 内存区域的申请由内核实现。`mmap()` 同样按页申请内存。

下面程序调用 `mmap()` 申请了一块内存。

```c
#include <sys/mman.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    int size = 10;
    void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    printf("ptr: %p\n", ptr);
    pause();
    munmap(ptr, size);
    return 0;
}
```

我们运行该程序，并通过 proc 查看该进程的内存布局

```console
$ clang run_mmap.c -o run_mmap
$ ./run_mmap &
ptr: 0x7f81ffca2000
$ cat /proc/$!/maps
```

该进程的内存布局如下。这里用 `=====` 隔开了内存空间各部分。`*` 标记出了程序中调用 `mmap()` 所申请的内存所处位置。

```
=====
      55a9bde3a000-55a9bde3b000 r--p 00000000 103:05 8392660                   /path/to/run_mmap
      55a9bde3b000-55a9bde3c000 r-xp 00001000 103:05 8392660                   /path/to/run_mmap
ELF   55a9bde3c000-55a9bde3d000 r--p 00002000 103:05 8392660                   /path/to/run_mmap
      55a9bde3d000-55a9bde3e000 r--p 00002000 103:05 8392660                   /path/to/run_mmap
      55a9bde3e000-55a9bde3f000 rw-p 00003000 103:05 8392660                   /path/to/run_mmap
=====  
Heap  55a9c4507000-55a9c4528000 rw-p 00000000 00:00 0                          [heap]
=====
      7f81ffa83000-7f81ffa86000 rw-p 00000000 00:00 0 
      7f81ffa86000-7f81ffaa8000 r--p 00000000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
      7f81ffaa8000-7f81ffc20000 r-xp 00022000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
      7f81ffc20000-7f81ffc78000 r--p 0019a000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
      7f81ffc78000-7f81ffc7c000 r--p 001f1000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
      7f81ffc7c000-7f81ffc7e000 rw-p 001f5000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
MMap  7f81ffc7e000-7f81ffc8b000 rw-p 00000000 00:00 0 
    * 7f81ffca2000-7f81ffca5000 rw-p 00000000 00:00 0 
      7f81ffca5000-7f81ffca6000 r--p 00000000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      7f81ffca6000-7f81ffcce000 r-xp 00001000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      7f81ffcce000-7f81ffcd8000 r--p 00029000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      7f81ffcd8000-7f81ffcda000 r--p 00033000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      7f81ffcda000-7f81ffcdc000 rw-p 00035000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
=====
Stack 7ffe63299000-7ffe632ba000 rw-p 00000000 00:00 0                          [stack]
=====
      7ffe6332c000-7ffe63330000 r--p 00000000 00:00 0                          [vvar]
      7ffe63330000-7ffe63332000 r-xp 00000000 00:00 0                          [vdso]
      ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
=====
```

由上表可知，虚拟内存空间地址由低到高依次为**程序（ELF各段）**、**堆**、**内存映射**和**栈**。调用 `mmap()` 所申请的内存空间在内存映射区内，这同时包括匿名映射和文件映射。另外我们还可以看到内存映射区内包含了两个共享库（`libc.so.6` 和 `ld-linux-x86-64.so.2`），这是因为动态链接库的加载（`dlopen()`）实际上是基于 `mmap()` 实现的。`dlopen()` 或读取共享库文件，并将其中各段映射到内存中对应位置，并赋予对应的读、写和执行权限。

> `vvar`、`vdso` 和 `vsyscall` 是为了加速某些系统调用而增加的内存段。

## 非主线程的线程栈

当进程里只有一个线程时，只需要使用单一栈空间，这时栈空间已经在程序运行前预先分配了给定的大小。但是如果存在多个线程，就需要拥有多个线程栈。这些线程栈的位置与主线程的栈不同。非主线程的线程栈实际上位于内存映射区内。

下面是一个简单的 pthread 多线程程序。

```c
#include <pthread.h>
#include <unistd.h>

void *thread_function(void *arg) {
    return NULL;
}

int main() {
    pthread_t thread;
    pthread_create(&thread, NULL, thread_function, NULL);
    pthread_join(thread, NULL);
    return 0;
}
```

我们使用 `strace` 追踪该程序运行过程中调用的系统调用。

```console
$ clang test_thread.c -o test_thread
$ strace ./test_thread
```

得到的结果如下

```console
execve("./test_thread", ["./test_thread"], 0x7ffc659bbb00 /* 73 vars */) = 0
brk(NULL)                               = 0x55d5e4bb3000
...
mmap(NULL, 8392704, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7fb614801000
mprotect(0x7fb614802000, 8388608, PROT_READ|PROT_WRITE) = 0
...
```

我们关注的是其中的 `mmap` 系统调用。该调用中传入的 flags 中包含 `MAP_STACK`。这说明申请的地址是作为程序栈使用的。紧接着 `mmap` 之后是一个 `mprotect` 调用，该调用将 `mmap` 所申请的一部分内存设置为可读写。这部分内存是栈实际使用的部分。

我们从 `/proc/<PID>/maps` 中也可找到这段内存映射，如下表中 `*` 所标记的地方。这说明对于非主线程，其线程栈由 `mmap` 申请，位于内存映射区内。

```
  55d5b2fd6000-55d5b2fd7000 r--p 00000000 103:05 8392953                   /path/to/test_thread
  55d5b2fd7000-55d5b2fd8000 r-xp 00001000 103:05 8392953                   /path/to/test_thread
  55d5b2fd8000-55d5b2fd9000 r--p 00002000 103:05 8392953                   /path/to/test_thread
  55d5b2fd9000-55d5b2fda000 r--p 00002000 103:05 8392953                   /path/to/test_thread
  55d5b2fda000-55d5b2fdb000 rw-p 00003000 103:05 8392953                   /path/to/test_thread
  55d5e4bb3000-55d5e4bd4000 rw-p 00000000 00:00 0                          [heap]
* 7fb614801000-7fb614802000 ---p 00000000 00:00 0 
* 7fb614802000-7fb615005000 rw-p 00000000 00:00 0 
  7fb615005000-7fb615027000 r--p 00000000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
  7fb615027000-7fb61519f000 r-xp 00022000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
  7fb61519f000-7fb6151f7000 r--p 0019a000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
  7fb6151f7000-7fb6151fb000 r--p 001f1000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
  7fb6151fb000-7fb6151fd000 rw-p 001f5000 103:06 9063362                   /usr/lib/x86_64-linux-gnu/libc.so.6
  7fb6151fd000-7fb61520a000 rw-p 00000000 00:00 0 
  7fb615222000-7fb615224000 rw-p 00000000 00:00 0 
  7fb615224000-7fb615225000 r--p 00000000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
  7fb615225000-7fb61524d000 r-xp 00001000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
  7fb61524d000-7fb615257000 r--p 00029000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
  7fb615257000-7fb615259000 r--p 00033000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
  7fb615259000-7fb61525b000 rw-p 00035000 103:06 9063067                   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
  7ffc7f9b5000-7ffc7f9d6000 rw-p 00000000 00:00 0                          [stack]
  7ffc7f9d9000-7ffc7f9dd000 r--p 00000000 00:00 0                          [vvar]
  7ffc7f9dd000-7ffc7f9df000 r-xp 00000000 00:00 0                          [vdso]
  ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```