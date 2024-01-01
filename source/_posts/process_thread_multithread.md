---
title: 进程、线程理解及多线程
date: 2017-01-06 22:42:18
tags:
- 多线程
categories:
- 学习笔记
---

### 进程 Process
进程是正在运行的程序的实例，一个任务就是一个进程
如下图，每一个运行的程序就是一个进程，有系统的，也有用户的

<!-- more -->

![process](http://7xsnb0.com1.z0.glb.clouddn.com/2017-01-04-064747.jpg)

### 进程间通信 IPC (Inter-Process Communication)
至少两个进程或县城建传送数据或信号的一些技术和方法，进程是彼此隔离的，为了让不同进程互相访问资源并协调工作，所以要进程间通信

主要的IPC方法有

- 文件

    每个进程都可以使用文件

- 信号

    可以通过发送信号给一个线程，操作系统可以终端进程的正常控制流程，任何非原子(nonatomic)操作都会被中断。
例如我们在终端运行了某个进程的时候，Ctrl-C发送INT信号，该进程会被终止，Ctrl-Z发送TSTP信号，该进程会被挂起

- Socket (Unix domain socket)

    用C语言写成的应用程序开发库，可用于进程间通信。在系统内核里，Socket可以被系统进程引用，两个进程可以同时使用一个Socket来通信

- 消息队列 Message queue

    可以在进程间通信或在同一进程的不同线程中通信。消息队列是异步的，一个进程通知以一个进程发生了一个事件，不需要等待，但是接收者必须轮询消息队列，才能收到最近的消息

- 管道

    是一个由标准输入输出链接起来的进程集合，每一个进程的输出被直接作为下一个进程的输入，例如在终端中输入 ls -l | less，这样ls -l的输出作为输入到less分页器中。
进程间的通信便是一个进程的输出作为另一个进程输入。

在大多数类UNIX操作系统中，管线上的所有进程同时启动，输入输出流也已经被正确地连接，并且这些进程被调度程序所管理。最为重要的一点就是，所有的UNIX管道和其他管道实现不一样的地方就是缓存的概念：输出进程可能会以每秒5000 byte的速度输出，但是接收进程也许每秒只能接收100 byte，但不会有数据丢失。原因就是管道上游的进程的所有输出都会被放入一个队列中。当下游进程开始接收数据时，操作系统就会将数据从队列传至接收进程，并将传完的数据从队列中移除。当缓存队列空间不足时，上游进程会被终止，直到接收进程读取数据为上游进程腾出空间。在Linux中，缓存队列的大小是65536 byte。

- 信号量 Semaphore

计数信号量具备两种操作动作，之前称为 V（又称signal( )）与 P（wait( )）。 V操作会增加信号量 S的数值，P操作会减少它。
两个进程共享信号量sv，一旦其中一个进程执行了P(sv)操作，它将得到信号量，并可以进入临界区，使sv减1。而第二个进程将被阻止进入临界区，因为当它试图执行P(sv)时，sv为0，它会被挂起以等待第一个进程离开临界区域并执行V(sv)释放信号量，这时第二个进程就可以恢复执行。
互斥锁(Mutex)也是这样实现的。信号量也可以用于同步，一般同步信号量的初值为0，互斥信号量的初值为1.

- 共享内存

可以被多个进程存取的内存

#### 进程的同步 Synchronization

进程的同步是解决进程间协作关系的手段，多个线程基于某个条件来协调它们的活动，一个进程的执行依赖于另一个协作进程的消息或信号，当一个进程没有得到来自于另一个进程的消息或信号时则需等待，直到消息或信号到达才被唤醒

主要的同步方法有
- 私有信号量
    我觉得用管道和Socket也可以
- 线程 Thread
    每一个进程中至少有一个线程，你看上面的图有的进程甚至有上百个线程

线程是进程中的调度单位，例如打开QQ，就运行了一个进程，QQ开一个线程传输文字，开一个线程传输图片，或者下载软件下载东西的时候，开一个线程下载某一部分，再开另一个线程下载另一部分。

同一进程中的多条线程共享该进程的全部系统资源，但是他们都有各自的调用栈，寄存器环境和线程本地存储。

### 多线程
#### PThreads ( POSIX Threads )
PThreads是基于C语言的多线程方案

创建线程

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    pthread_t pt;
    pthread_create(&pt, NULL, thread_func, NULL);
    
    pthread_t pt2;
    apt = pthread_create(&pt2, NULL, thread_func2, NULL);
    
    for (int i = 0; i < 10; ++i) {
        NSLog(@"1111111111 ---- %d",i);
        
    }
}

void *thread_func(void *atgs){
    int i;
    for(i = 0;i < 10;++i){
        NSLog(@"2222222222 ---- %d",i);
    }
    return NULL;
}

void *thread_func2(void *atgs){
    int i;
    for(i = 0;i < 10;++i){
        NSLog(@"3333333333 ---- %d",i);
        if(i == 5){
            pthread_exit(NULL);
        }
    }
    return NULL;
}
```

`pthread_create(..)`是创建一个线程的方法，第三个参数需要传入一个C函数指针，OC中的方法不行

上面创建了两个线程，然后在viewDidLoad里面也输出0到10，可以看到结果是3个线程里面的方法随机输出

`pthread_exit(NULL)`是退出当线程，可以看到当i==5的时候退出打印3的线程

![](http://7xsnb0.com1.z0.glb.clouddn.com/2017-01-04-182025.jpg)

`pthread_once` 表示永远只执行一次

```
pthread_once_t pot;
pthread_once(&pot, thread_once);
```

#### 互斥锁

```
pthread_mutex_t mt;
pthread_mutex_init(&mt, NULL);
//...
pthread_mutex_lock(&mt);
//互斥的操作...
pthread_mutex_unlock(&mt);
```

简单例子
现在有一个方法里面有一个计数cnt自增，并且暂停一秒（模拟方法需要执行很久的时候），有3个线程同时调用这个方法并且输出cnt值

```
void thread_mutextest(void){
    ++cnt;
    sleep(1);
}
```

这时候sleep(1)完之后几乎3个线程一起读取cnt的值

![thread](http://7xsnb0.com1.z0.glb.clouddn.com/2017-01-05-104836.jpg)

```
void thread_mutextest(void){
    pthread_mutex_lock(&mt);
    ++cnt;
    sleep(1);
    pthread_mutex_unlock(&mt);
}
```

如果加了互斥锁，每一秒只能有一个线程去读取cnt的值


读写锁
互斥锁会把试图进入已保护的临界区的线程都阻塞，即同时只有一个线程持有锁
读写锁会根据当前进入临界区的线程是读还是写的属性来判断是否允许线程进入。多个读者可以同时持有锁。

适用于读数据的频率远大于写数据的频率的应用中，这样可以在任何时刻运行多个读线程并发的执行。

```
pthread_rwlock_t rwt;
pthread_rwlock_init(&rwt, NULL);

//读加锁
pthread_rwlock_rdlock(&rwt);

//写加锁
pthread_rwlock_wrlock(&rwt);

//读写锁的解锁
pthread_rwlock_unlock(&rwt);
```

#### GCD
基本操作就不说了，网上太多文章了，很多都写的很好
但是我们来看看`dispatch_barrier`

`dispatch_barrier`是一种屏障，也可以理解成一种锁，它让block里面的任务等待队列(queue)之前的任务执行完之后再执行，执行block里面的任务时，队列后面的任务也要等待。

例如

```
//创建一个并行队列
dispatch_queue_t dqt = dispatch_queue_create("concurrent_queue", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(dqt, ^(){
    for(int i = 0;i < 5;++i){
        NSLog(@"dispatch-1");
    }
});
dispatch_async(dqt, ^(){
    for(int i = 0;i < 5;++i){
        NSLog(@"dispatch-2");
    }
});
dispatch_barrier_async(dqt, ^(){
    for(int i = 0;i < 5;++i){
        NSLog(@"dispatch-barrier");
    }
});

dispatch_async(dqt, ^(){
    for(int i = 0;i < 5;++i){
        NSLog(@"dispatch-3");
    }
});
dispatch_async(dqt, ^(){
    for(int i = 0;i < 5;++i){
        NSLog(@"dispatch-4");
    }
});
```

输出结果

![gcd](http://7xsnb0.com1.z0.glb.clouddn.com/2017-01-05-195338.jpg)

但是在全局队列(dispatch_get_global_queue(..))中不起作用，在自己创建的并行、串行队列中，使用同步、异步都起作用

### 参考
[进程和线程定义，概念，区别详解 - 慕课网](http://www.imooc.com/article/11011)
[进程间通信 - 维基百科](https://zh.wikipedia.org/wiki/%E8%A1%8C%E7%A8%8B%E9%96%93%E9%80%9A%E8%A8%8A)

还有网上很多文章

