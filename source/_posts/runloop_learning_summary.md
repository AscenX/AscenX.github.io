---
title: RunLoop学习总结
date: 2016-07-08 23:36:12
tags:
- Objective-C
- RunLoop
categories:
- 学习笔记
---

### 开始
很久之前就看了一次YY的文章，没看懂。后来又看了sunny的视频和叶孤城的直播的视频，找了很多资料，对RunLoop也越来越清晰，然后又看了两三次YY的文章，虽然还没完全看懂，不得不说写的非常好，帮助很大。还有官方文档，学到了很多东西。还有就是github上的一些demo，文中一些代码别人写过了，我就直接拿过来了。文中一些结论也是取自大神的文章或者视频。非常感谢这些前辈大神们的分享吧！！

<!-- more -->

### 什么是RunLoop
RunLoop其实就是一种处理事件、消息的机制被面向对象化，它就是一个对象。其实它的本质就是一个do-while循环

```
do {
    //如果有事件/消息，就处理
	//没事件/消息或者都处理完了就沉睡
} while(1)
```

### NSRunloop和CFRunloop
`NSRunLoop`是对`CFRunLoop`的封装，使之更面向对象化。我们一般在`NSRunLoop`层面上操作。`CFRunLoop`基于C语言，从属于`CoreFoundation`(开源)。

官方文档里对`CFRunLoop`的解释很好，它建立任务来监测着输入源(事件源，事件/消息)和准备好处理输入源的时候调度管理，输入源包括用户输入的设备，网络连接，周期性或者延迟的事件，还有异步的回调等。

### RunLoop的结构组成
一个RunLoop可以监测三个对象，`Sources`, `Timers`, `Observers`，如图

![RunLoop](http://7xsnb0.com1.z0.glb.clouddn.com/2016-07-05_RunLoop_0.png)

图来自YY的文章，这个图很棒

其实是一个RunLoop对象可以包含一个或者多个Mode，这个Mode可以说是一种模式，一种配置。而Mode又包含着输入源（事件源）Source，观察者Observer，和计时器Timer和其他一些信息。

### `CFRunLoopMode`
Mode决定了RunLoop在什么时候处理什么事情，在某个Mode在某个时候能做什么不能做什么，在某个Mode某个时候只能做什么。
切换Mode需要先推出RunLoop再重新进入。

Mode有几种

- DefaultMode，`CFRunLoop`的是`kCFRunLoopDefaultMode`，`NSRunLoop`的是`NSDefaultRunLoopMode`，这是默认的mode，即APP平常的状态
- UITrackingRunLoopMode，追踪scrollView滑动的状态，当有scrollView滑动的时候RunLoop会切换到该mode下，滑动结束再切换到其他mode
- UIInitializationRunLoopMode，这个是私有的，在APP启动时到显示第一个界面期间会进入这个mode
- NSRunLoopCommonModes，这个是Mode集合，自定义或者若干个集合，默认是包含了defaultMode和trackingMode
- GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到。

这里先说一个例子，例如我们创建的NSTimer对象在运行时碰到有scrollView的空间滑动的时候是默认不会继续计时的

```
    [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(tiktok) userInfo:nil repeats:YES];
    
- (void)tiktok
{
    NSLog(@"%d",self.count++);
}
```

![defaultMode](http://7xsnb0.com1.z0.glb.clouddn.com/7%E6%9C%88-05-2016%2009-14-14.gif)

那是因为用scheduleTime...这个方法，timer默认被添加到了RunLoop的defaultMode，而滑动tableView的时候RunLoop切换到了trackingMode所以停止了。

解决方法就是把timer添加到当前RunLoop的UITrackingRunLoopMode或者NSRunLoopCommonModes

```
//只要添加一句
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];
//或者
//[[NSRunLoop currentRunLoop] addTimer:timer NSRunLoopCommonModes];
```

![UITrackingRunLoopMode](http://7xsnb0.com1.z0.glb.clouddn.com/7%E6%9C%88-05-2016%2009-19-29.gif)

##### Mode的结构

`CFRunLoopMode`是一个`struct`，最主要是`Source`、`Timer`和`Observer`，这些被称为`mode item`，还有一些其他信息，例如name，就是指上面的defaultMode或者trackingMode的名字，还有一些线程锁标志信息等。

```c
struct __CFRunLoopMode {
    //Mode的名字
    CFStringRef _name;
    
    //Source就两个
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    
    //Observer数组
    CFMutableArrayRef _observers;
    
    //Timer数组
    CFMutableArrayRef _timers;
    
    //还有其他的一些信息
    //.....
};
```

### CFRunLoopSource
输入源(input source)是指用户的操作事件（例如点击屏幕）或者网络端口接收到的信息等。而一个CFRunLoopSource对象就是跑在RunLoop上输入源的抽象概念，Source可以说是用户操作的事件、消息和RunLoop的中间层。

有两种Source

- Source0，被APP所管理，处理内部事件，如UIEvent、CFSocket
- Source1，被RunLoop和内核所管理，由Mach port驱动,如CFMachPort、CFMessagePort
这里已经涉及到很底层的东西了，查了下Mach port，很底层，太少资料，大概了解到一点是用来通信的通道，端口，这个还有待研究

### CFRunLoopObserver
这个就是观察者，同来用来接收各种回调(callbakcs)。一个Observer一次只能被一个RunLoop注册。当RunLoop的状态切换的时候Observer可以观察到

这个是RunLoop的状态

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry  = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```

然后我们来测试一下，创建一个Observer来观察RunLoop的状态

```
//创建一个Observer，观察RunLoop的所有状态
//这里打印出来的数字是上面struct的数字X的2^X
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
       NSLog(@"RunLoop状态-%zd", activity);
   });
   
   CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopDefaultMode);
   //注意CF对象要释放
   CFRelease(observer);
```

然后查看打印

![2016-07-05_22:57:57.jpg](http://7xsnb0.com1.z0.glb.clouddn.com/2016-07-05_22:57:57.jpg)

后面的数字即刚才RunLoop的状态，对比struct可以看出，APP一运行就先进入RunLoop，然后2/4切换，即处理Source和Timer然后是32，即将进入睡眠状态，再64，即将从睡眠状态中醒来…然后循环两三次到最后32，即将进入睡眠状态，这就是一个APP打开的时候然后没给任何动作

接下来我滑动一下tableView，来看一下打印

![2016-07-05_23:20:51.jpg](http://7xsnb0.com1.z0.glb.clouddn.com/2016-07-05_23:20:51.jpg)

先醒过来，处理Source和Timer，然后有个128（即对应上面的2^7)，即将退出RunLoop，这是意料之中，因为我滑动tableView把mode切换到了trackingMode，而切换Mode的时候需要先退出mode再从新进入，等松开手后最终状态又变成32，睡眠状态

这也很符合刚开始的do-while循环，一有事件/消息处理就处理，不然就进入睡眠

### RunLoop和线程的关系
线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。(直接取自YY的结论)

### RunLoop和Autorelease Pool
UIkit通过RunLoopObserver在RunLoop两次Sleep间对Autorelease Pool进行Pop和Push将这次Loop中产生的Aotorelease对象释放

### 应用

#### tableView滑动后再加载图片
当一个tableView或者collectionView（这两个都是继承自scrollView)里面放了很多图片的时候并且要加载的时候，有一个优化的措施就是在滑动的时候不加载，在滑动完了之后再开始加载。

```
UIImage *downloadedImage = ...;
self.imageView performSelector:@selector(setImage:) withObject:downloadedImage afterDelay:0 inModes:@[NSDefaultRunLoopMode]];
```

(以上代码取自sunny的视频)

让setImage方法只在NSDefaultRunLoopMode里面运行，在滑动的时候进入trackingMode就不执行

还有更具体的实现可以看下这个github demo

虽然可以这样实现，但是我觉得使用性不强，我看了很多较常用的APP都没有这样实现，且当学习吧

#### 常驻线程

```
[[NSRunLoop currentRunLoop] addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
[[NSRunLoop currentRunLoop] run];
```

或者

```
[[NSRunLoop currentRunLoop] runUntilDate:[NSDate distantFuture]];
```

#### 监测滑动卡顿
前面说到RunLoop会一直切换状态，我们要监测的是在处理Source的时候的时间，如果时间太长，则说明有可能卡顿了

只需要另外再开启一个线程,实时计算这两个状态区域之间的耗时是否到达某个阀值,便能揪出这些性能杀手. 为了让计算更精确,需要让子线程更及时的获知主线程NSRunLoop状态变化,所以dispatch_semaphore_t是个不错的选择,另外卡顿需要覆盖到多次连续小卡顿和单次长时间卡顿两种情景,所以判定条件也需要做适当优化.将上面两个方法添加计算的逻辑如下:`

```
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    MyClass *object = (__bridge MyClass*)info;

    // 记录状态值
    object->activity = activity;

    // 发送信号
    dispatch_semaphore_t semaphore = moniotr->semaphore;
    dispatch_semaphore_signal(semaphore);
}

- (void)registerObserver
{
    CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,kCFRunLoopAllActivities,YES, 0,&runLoopObserverCallBack,&context);
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    // 创建信号
    semaphore = dispatch_semaphore_create(0);
    // 在子线程监控时长
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES)
        {
            // 假定连续5次超时50ms认为卡顿(当然也包含了单次超时250ms)
            long st = dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, 50*NSEC_PER_MSEC));
            if (st != 0)
            {
                if (activity==kCFRunLoopBeforeSources || activity==kCFRunLoopAfterWaiting)
                {
                    if (++timeoutCount < 5)
                        continue;
                    NSLog(@"好像有点儿卡哦");
                }
            }
            timeoutCount = 0;
        }
    });
}
```

以上带去取自https://github.com/suifengqjn/PerformanceMonitor

### 最后
写的有点糟，只是一个小总结，感觉还是有好多不懂，还是要深入学习。

