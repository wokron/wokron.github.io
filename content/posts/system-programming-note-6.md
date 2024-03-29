+++
title = "系统编程之线程管理"
tags = ["C", "Linux"]
categories = ["操作系统"]
series = ["系统编程笔记"]
aliases = ["/posts/3fc6e65a"]
date = "2022-12-22T22:07:37+08:00"
+++
## 一、Linux 多线程简述
进程和线程的关系老生常谈。线程是最小的调度单位，进程是最小的资源分配单位。同一进程中的多个线程是在共享的内存空间中并发的多道执行路径，它们**共享一个进程的资源**。

对于Linux来说，Linux线程属于用户级线程，即线程的调度是在用户空间执行的。也就是说，Linux线程的实现是在内核之外的，多线程的概念对于内核来说并不是真实存在的，而只是通过线程库中的程序模拟的并发效果。

Linux线程遵循POSIX线程接口，称为pthread。pthread在其他平台也有对应的实现，如在windows。

## 二、线程操作
### （1）库的使用
在开始多线程编程之前，需要说明一下 pthread.h 库。在编译使用pthread.h库的代码时，一般需要加-lpthread。pthread在glibc2.34之前是在glibc里面的，之后分出来变成一个单独的库，因此有的情况下，不加-lpthread也能编译成功。

### （2）基本操作
1. 创建线程
```c
int pthread_create(pthread_t _Nullable * _Nonnull __restrict,
    const pthread_attr_t * _Nullable __restrict,
    void * _Nullable (* _Nonnull)(void * _Nullable),
    void * _Nullable __restrict);
```
该函数中第一个参数为指向一个线程标识变量的指针。第二个参数用来手动设置线程的各项属性，一般可以用NULL选择默认属性。第三个参数为一个函数指针，表示新建线程时需要执行的函数。注意该函数的参数类型和返回值类型，使用时需要进行强制类型转换。第四个参数为传递给函数的参数，也就是线程执行的函数的参数。不传递参数时可设置为NULL。
如下举一个创建线程的例子。
```c
pthread_t tid;
if (pthread_create(&tid, NULL, do_something, NULL)) 
{
    // error handler
}
```

2. 线程退出
```c
void pthread_exit(void *ral_ptr);
```
当某一线程执行该函数时，会导致该线程结束。结束时会将ral_ptr指针传递给`pthread_join` 函数的 `rval_ptr`

3. 线程取消
```c
int pthread_cancel(pthread_t tid);
```
某一线程调用该函数，可以终止同一进程内的其他线程。 `tid` 即要终止的线程。

4. 线程挂起
```c
int pthread_join(pthread_t thread, void **rval_ptr);
```
某一线程调用该函数会阻塞该线程，直到参数 `thread` 所指示的线程退出。第二个参数为一个指向 `pthread_exit` 所设置的 `ral_ptr` 指针的指针。

## 三、线程的控制
多线程中，经常需要多个线程对同一资源进行访问。在这种情况下，保持访问的原子性以及确定访问的顺序就十分重要。互斥量和条件变量就是对线程进行控制的工具。

### （1）互斥量（mutex）
互斥量的作用是避免对同一资源的同时访问。使用类似于信号量。只不过互斥量同时只能由一个线程持有。
1. 创建互斥量
```c
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);
```
调用该函数初始化一个互斥量。第一个参数为一个指示互斥量的变量，第二个参数为互斥量的属性，一般可设为NULL。
互斥量的属性有如下几种：
   - `PTHREAD_MUTEX_TIMED_NP` 普通锁。当一个线程加锁以后，其余请求锁的线程将形成一个等待队列，并在解锁后按优先级获得锁。这种锁策略保证了资源分配的公平性
   - `PTHREAD_MUTEX_RECURSIVE_NP` 嵌套锁，允许同一个线程对同一个锁成功获得多次，并通过多次unlock解锁。如果是不同线程请求，则在加锁线程解锁时重新竞
   争
   - `PTHREAD_MUTEX_ERRORCHECK_NP` 检错锁，如果同一个线程请求同一个锁，则返回EDEADLK，否则与PTHREAD_MUTEX_TIMED_NP类型动作相同。这样就保证当不允许多次加锁时不会出现最简单情况下的死锁
   - `PTHREAD_MUTEX_ADAPTIVE_NP` 适应锁，动作最简单的锁类型，仅等待解锁后重新竞争

2. 加锁
```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
```
调用该函数以对该互斥量进行加锁。加锁时，除获得锁的线程外，试图加锁的其他线程都会被阻塞，直到获得锁的线程进行解锁。

3. 解锁
```c
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
调用该函数对互斥量进行解锁。没有获得锁的线程调用不会产生效果。

4. 销毁互斥量
```c
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

### （2）条件变量
条件变量与互斥量一起使用时，允许线程以无竞争的方式等待特定条件的发生。这类似于Java中的wait()和notify()。这能用于实现线程的同步和顺序执行。

1. 创建条件变量
```c
int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *attr);
```

2. 等待
```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
```
调用该函数，可以使调用的线程陷入阻塞状态，直到该条件变量被通知。

条件变量的等待需要与互斥量配合使用。在调用pthread_cond_wait前，需要使互斥量处于锁住状态。这样pthread_cond_wait函数可以以原子的方式，将调用
线程放到等待条件的线程列表上。

等待线程的操作顺序为：
   - 调用pthread_mutex_lock
   - 调用pthread_cond_wait
   - 调用pthread_mutex_unlock

3. 通知
```c
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *);
```
`pthread_cond_signal` 和 `pthread_cond_broadcast` 都能唤醒因为调用 `pthread_cond_wait` 而陷入阻塞的线程，区别是 `pthread_cond_signal` 唤醒某一个等待该条件的线程， `pthread_cond_broadcast` 唤醒等待该条件的所有线程。

1. 销毁条件变量
```c
int pthread_cond_destroy(pthread_cond_t * cond);
```

### （3）代码实例——实现消费者模型
生产者消费者模型是条件变量最经典的使用场景之一，该问题描述了共享固定大小缓冲区的两个线程——即所谓的“生产者”和“消费者”——在实际运行时会发生的问题。生产者的主要作用是生成一定量的数据放到缓冲区中，然后重复此过程。与此同时，消费者也在缓冲区消耗这些数据。该问题的关键就是要保证生产者不会在缓冲区满时加入数据，消费者也不会在缓冲区中空时消耗数据。

生产者消费者问题主要要注意以下三点：

- 在缓冲区为空时，消费者不能再进行消费
- 在缓冲区为满时，生产者不能再进行生产
- 在一个线程进行生产或消费时，其余线程不能再进行生产或消费等操作，即保持线程间的同步


```c
#include <pthread.h>
#include <stdlib.h>
#include <stdio.h>

#define BUF_SIZE 20
#define MAX_STEP 5
#define T_SIZE 10

int buf[BUF_SIZE];
int size = 0;
int head = 0, top = 0;

pthread_mutex_t mutex;
pthread_cond_t not_full, not_empty;

void* consume(void *vp)
{
    int num = (int)vp;
    for (int i = 0; i < MAX_STEP; i++)
    {
        pthread_mutex_lock(&mutex);
        while (size == 0)
        {
            pthread_cond_wait(&not_empty, &mutex);
        }
        printf("consume %d, get %d\n", num, buf[head]);
        head = (head+1)%BUF_SIZE;
        size--;
        pthread_mutex_unlock(&mutex);
        pthread_cond_signal(&not_full);
    }
}

void* product(void *vp)
{
    int num = (int)vp;
    for (int i = 0; i < MAX_STEP; i++)
    {
        pthread_mutex_lock(&mutex);
        while (size == BUF_SIZE)
        {
            pthread_cond_wait(&not_full, &mutex);
        }
        int r = rand() % 20;
        printf("product %d, send %d\n", num, r);
        buf[top] = r;
        top = (top+1)%BUF_SIZE;
        size++;
        pthread_mutex_unlock(&mutex);
        pthread_cond_signal(&not_empty);
    }
}

int main()
{
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&not_empty, NULL);
    pthread_cond_init(&not_full, NULL);

    pthread_t c[T_SIZE], p[T_SIZE];
    for (int i = 0; i < T_SIZE; i++)
    {
        pthread_create(c+i, NULL, consume, (void*)i);
    }
    
    for (int i = 0; i < T_SIZE; i++)
    {
        pthread_create(p+i, NULL, product, (void*)i);
    }

    for (int i = 0; i < T_SIZE; i++)
    {
        pthread_join(c[i], NULL);
        pthread_join(p[i], NULL);
    }
    
    return 0;
}
```

**一次执行的结果：**
```text
product 0, send 3
consume 0, get 3
product 0, send 6
product 0, send 17
consume 2, get 6
consume 2, get 17
product 0, send 15
product 0, send 13
consume 4, get 15
consume 4, get 13
product 1, send 15
consume 5, get 15
product 4, send 6
product 5, send 12
product 5, send 9
consume 8, get 6
consume 8, get 12
consume 8, get 9
product 2, send 1
consume 6, get 1
product 4, send 2
product 7, send 7
product 7, send 10
product 7, send 19
product 7, send 3
product 7, send 6
consume 2, get 2
consume 2, get 7
consume 2, get 10
consume 4, get 19
consume 4, get 3
consume 4, get 6
product 2, send 0
product 2, send 6
product 2, send 12
product 5, send 16
consume 8, get 0
consume 8, get 6
consume 9, get 12
consume 9, get 16
product 6, send 11
product 1, send 8
product 3, send 7
consume 0, get 11
consume 0, get 8
consume 0, get 7
product 2, send 9
consume 7, get 9
product 8, send 2
consume 1, get 2
product 4, send 10
consume 3, get 10
product 4, send 2
product 4, send 3
product 8, send 7
product 9, send 15
consume 6, get 2
consume 6, get 3
consume 6, get 7
consume 6, get 15
product 6, send 9
product 6, send 2
product 6, send 2
product 6, send 18
consume 3, get 9
consume 3, get 2
consume 3, get 2
consume 3, get 18
product 1, send 9
product 1, send 7
product 1, send 13
consume 1, get 9
consume 1, get 7
consume 1, get 13
product 3, send 16
consume 7, get 16
product 3, send 11
product 3, send 2
product 3, send 9
consume 1, get 11
consume 5, get 2
product 5, send 13
product 5, send 1
consume 0, get 9
consume 5, get 13
consume 5, get 1
product 9, send 19
consume 5, get 19
product 9, send 4
consume 9, get 4
product 9, send 17
consume 9, get 17
product 9, send 18
consume 9, get 18
product 8, send 4
consume 7, get 4
product 8, send 15
consume 7, get 15
product 8, send 10
consume 7, get 10
```
