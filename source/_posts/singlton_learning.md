---
title: 单例模式探索
date: 2016-07-02 21:33:03
tags:
- c++
- 设计模式
categories:
- 学习笔记
---

以前写OC中的单例很固定，一直都这样写，后来我就把它放在快捷代码块里面，只要输入singleton就直接输出这段代码

<!-- more -->

```
+ (instancetype)instance
{
    static Class *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}
```

一直这样写，并没有关注内部实现
这两天一直在复习C++，然后看到了C++的单例模式，发现内部实现还是挺有讲究的

先来看OC中的
这个方法刚学OC的时候就记下来了

```
void dispatch_once( dispatch_once_t *predicate, dispatch_block_t block);
```

官方文档的解释是，这个方法让block里面的代码在整个应用的生命周期里只执行一次，如果多线程同时调用，这个函数会让线程在block执行完之前都会同步地等待。只执行一次，顺带帮我们解决多线程问题，这是苹果帮我们封装好的，官方也建议用来写单例的方法！！

然后来看C++的

```c++
Class Singleton {
private:
    //构造函数
	Singleton();
	
	//静态对象
	static Singleton* instance;
    
public:
	static Singleton* getInstance {
		if(instance == NULL)
			instance = new Singleton();
		return instance;
	}
}
```

首先，我们要把构造方法设置private，让别人无法调用构造方法
C++的静态对象和方法就像OC中的类的类对象和方法，类似在OC中是以+开头的方法
把instance和getInstance方法都设为静态，让类直接调用

这是一种实现方法，但是并没有考虑到线程安全
在getInstance()方法应该上锁

```
static Singleton* getInstance {
	lock();	//这里不实现上锁方法，只是探讨单例模式
	if(instance == NULL)
		instance = new Singleton();
	unlock();	//伪
	return instance;
}
```

这样是可行的，但是每次通过getInstance方法都会加上一个同步锁，加锁是一个非常耗时的操作，可以在没有必要的时候应该尽量避免。

进一步完善，当instance还没有创建的时候才需要加锁，等第二个线程上锁的时候再判断一次是否已经创建了instance

```
static Singleton* getInstance {
	if(instance == NULL){
		lock();	//伪
		if(instance == NULL)
			instance = new Singleton();
		unlock();	//伪
	}
	return instance;
}
```

另外，要保证只能从单例方法获取单例，不能赋值和拷贝
所以要声明赋值和拷贝方法，但是不实现

1
2
Singleton(const Singleton&){};
Singleton&operator=(const Singleton&){};
在C++11也可以这样

```
Singleton(const Singleton&) = delete;
Singleton&operator=(const Singleton&) = delete;
```

所以这是一种比较好的实现方法

```
Class Singleton {
private:
	//构造函数
	Singleton();
	
	//静态对象
	static Singleton* instance;
	
	//赋值和拷贝方法
	Singleton(const Singleton&){};
	Singleton&operator=(const Singleton&){};
	
public:
	static Singleton* getInstance {
		if(instance == NULL){
			lock();	//伪
			if(instance == NULL)
				instance = new Singleton();
			unlock();	//伪
		}
		return instance;
	}
}
```

关于类的静态成员
在C++中，静态成员不属于类的任何一个对象，所以它们并不是在创建类的对象时被定义的，它们不是由类的构造函数初始化的，一般来说不能再累的内部初始化静态成员，必须在类的外部定义和初始化每个静态成员，类似于全局变量，静态成员定义在任何函数之外，一旦被定义，将一直存在于成员的整个生命周期中
我们可以利用整个特性来写单例模式

```
class Singleton {
private:
	Singleton();
	static Singleton* instance;
public:
	static Singleton* getInstance{
		
		return instance;
	}
}
//然后在类外初始化
Singleton* Singleton::instance = new Singleton();
```

这种方法更加简洁，并且也是线程安全的，因为除了函数里面的静态变量，其他的静态变量都是在main函数之前就创建好了，单线程创建的，参考stackoverflow上的一个回答

在C#中可以直接在类内初始化

```
public sealed class Singleton {
	private Singleton(){}
	private static Singleton instance = new Singleton();
	public static Singleton getInstance {
		get
		{
			return instance;
		}
	}
}
```

这种方法比上一种二重锁的方法性能要更好
在C#中还有更好的方法

```
public sealed class Singleton {
	Singleton(){}
	public static Singleton getInstance{
		get {
			return Nested.instance;
		}
	}
	Class Nested{
		static Nested(){}
		internal static readonly Singleton instance = new Singleton();
	}
}
```

嵌套类，当用到Nested类的时候才会调用构造方法创建instance
由于无法实现类内初始化instance，所以这种方法我还不知道C++怎么实现
以上方法都是参考自剑指Offer

