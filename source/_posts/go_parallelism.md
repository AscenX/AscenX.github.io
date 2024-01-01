---
title: Go并发初步学习
date: 2017-01-29 17:37:31
tags:
- Golang
- 并发
categories:
- 学习笔记
---

### 并发(concurrent)与并行(parallelism)
并发就是一个处理器同时处理多个任务，不是真正的同时处理，需要在多线程中快速切换，由于切换速度非常快造成同时的假象
并行就是真正的同时发生，一般是多个处理器或者多核处理器同时处理多个任务

<!-- more -->

这张图很有概括性，图来自网上

![ascen.me_go](http://7xsnb0.com1.z0.glb.clouddn.com/2017-01-27-185928.jpg)

### goroutine
在Go语言中，没有线程的概念，并发编程最小的调度单位是goroutine，而且本身就很小，创建一个goroutine只有几KB

用法也非常简单，只需要在函数前面加一个 go

```
go test()	//让test()并发运行
```
但是有一个问题就是mian函数不会等待并发执行的任务，举个例子

```
func test() {
	time.Sleep(1 * time.Second)
	fmt.Println("test")
}

func main() {
	go test()
}
```
这样的话是得不到输出的，因为在test()结束之前，mian函数已经结束了。
需要在mian函数也添加一句

```
time.Sleep(2 * time.Second)	
//如果也是1秒的话结果不一定，可能主函数先完也可能test函数先完
```
这里就要并发运行的函数能够告诉主函数，我运行完了，你现在才可以运行完，所以就要用到管道(channel)，

### 管道 channel
官方文档叫信道，都一样，只是一条传递信息的通道
管道的创建方式有两种，带缓冲和不带缓冲的

```
c1 := make(chan int)	//不带缓冲的int管道
c2 := make(chan int, 2)	//缓冲为2的int管道，可以缓冲两个int
```
刚才的例子可以改成

```
var c chan int = make(chan int)

func test() {
	fmt.Println("test")
	c <- 1
}

func main() {
	go test()
	<-c	//阻塞直到收到值
}
```
管道若不带缓冲，那么在管道中没有数据的话，接收方就会阻塞直到有值。如果发送者还要发送值的话，要等管道中的值被接收之后（也就是清空管道了）才能继续发送。若管道是带缓冲的，则发送者仅在值被复制到缓冲区前阻塞； 若缓冲区已满，发送者会一直等待直到某个接收者取出一个值为止。

这里的缓存也是结构应该也是队列，先进先出

直接读取管道中的内容的话只能读取一个值，可以使用for-range来读取管道，但是如果读完了管道中的内容之后就会阻塞，所以需要在连续发送完使用close(chan)来管道

举个简单的例子，使用匿名函数（闭包）来并发运行

```
c := make(chan int)
go func(){
	fmt.Println("go func")
	c <- 1
	c <- 2
	close(c)	//如果没有这句就会形成死锁(deadlock)
}()
for v := range c {
	fmt.Println(v)
}
```

输出

```
go func
```

如果不用for-range读取管道的话，是读取不到2的

一般用带缓存的队列可以用来限制吞吐量

#### 并行处理
如果计算过程被分为几个独立执行的过程，每个过程可以在管道中发送信号，从而实现并行处理

这个例子取自 《实效Go编程》

```
type Vector []float64

func (v Vector) DoSomething(i, n int, u Vector, c chan int) {
	for ; i < n; i++ {
		v[i] += u.Op(v[i])
	}
	c <- 1	//发信号表示这一块计算完成
}

  
const NCPU = 4  // CPU核心数

func (v Vector) DoAll(u Vector) {
	c := make(chan int, NCPU)  // 缓冲区是可选的，但明显用上更好
	//任务分成NCPU个部分
	for i := 0; i < NCPU; i++ {
		go v.DoSomething(i*len(v)/NCPU, (i+1)*len(v)/NCPU, u, c)
	}
	// 排空信道。
	for i := 0; i < NCPU; i++ {
		<-c    // 等待任务完成
	}
	// 一切完成。
}
```

Go默认不会并行，如果要开启并行的话，在运行你的工作时将 GOMAXPROCS 环境变量设为你要使用的核心数， 要么导入 runtime 包并调用 runtime.GOMAXPROCS(NCPU)。 runtime.NumCPU() 的值可能很有用，它会返回当前机器的逻辑CPU核心数。

#### sync.WaitGroup
前面channel的作用是等到所有goroutine运行完之后再结束主程序
在sync包里面有一个WaitGroup 同样可以实现这样的功能

WaitGroup有3个方法

Add(delta int) //添加或者减少等待goroutine的数量
Done() //完成，相当于Add(-1)
Wait() //执行阻塞，直到所有WaitGroup数量变成0

```
func mian(){
	wg := sync.WaitGroup{}
	wg.add(10)	//添加10个goroutine事件
	for i := 0; i < 10; i++ {
		go Go(&wg, i)
	}
	wg.Wait()	//等待直到完成
}

func Go(wg *sync.WaitGroup, index int){
	a := 1
	for i := 0;i < 1000;i++ {
		a += i
	}
	fmt.Println(index, a)
	wg.Done()	//本次事件完成
}
```

#### select
select是一个控制结构，类似switch，但是是随机执行一个可运行的case，如果没有case可运行那就会阻塞

```
c1 := make (chan int, 1)
c2 := make (chan int, 1)

select {
case <-c1:
    fmt.Println("c1")
case <-c2:
    fmt.Println("c2")
}
```
如果ch1，ch2 都执行完任务，之后就要关闭管道，防止一直阻塞

### 总结
Go语言中的并发编程很多东西好像写起来很简单，但是可以有很多种高深的写法，能做很多事情。还有很多东西要学习，并发也还要深入的学习。

### 参考
[实效Go编程-goroutine部分](https://go-zh.org/doc/effective_go.html#Go%E7%A8%8B)
Go编程基础视频goroutine部分

