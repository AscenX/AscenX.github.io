---
title: C++ RAII与智能指针
date: 2017-01-08 16:11:47
tags:
- c++
categories:
- 学习笔记
---

### 什么是RAII
资源获取即初始化 (Resource Acquisition Is Initialization, RAII)，RAII是一种资源管理机制，资源的有效期与持有资源的对象生命期严格绑定，即由对象的构造函数完成资源的分配，由析构函数完成资源的释放，总结一句话就是 用对象来管理资源

<!-- more -->

### RAII实现原理
当一个对象离开作用域的时候就会被释放，会调用这个对象类的析构函数，这都是自动管理的，不需要我们手动调用。所以我们可以把资源封装到类的内部，当需要用资源的时候初始化对象即可，当对象被释放的时候资源也会被释放

看一个小例子

```
#include <iostream>
using namespace std;

class ArrayOperation
{
public :
    ArrayOperation()
    {
        m_Array = new int [10];
    }

    void InitArray()
    {
        for (int i = 0; i < 10; ++i)
        {
            *(m_Array + i) = i;
        }
    }

    void ShowArray()
    {
        for (int i = 0; i <10; ++i)
        {
            cout<<m_Array[i]<<endl;
        }
    }

    ~ArrayOperation()
    {
        cout<< "~ArrayOperation is called" <<endl;
        if (m_Array != NULL )
        {
            delete[] m_Array;
            m_Array = NULL ;
            cout << "m_Array is released" << endl;
        }
    }

private :
    int *m_Array;
};

void test(){
    ArrayOperation arrayOp;
    arrayOp.InitArray();
    arrayOp.ShowArray();
    cout << "test end" << endl;
}

int main()
{
    test();
    cout << "return 0" << endl;
    return 0;
}
```

输出结果如下
![ascen.me_cpp_raii](http://7xsnb0.com1.z0.glb.clouddn.com/2017-01-06-214110.jpg)

可以看到在test函数结束之后会释放对象arrayOp，然后会调用ArrayOperation的析构函数，会释放掉指针

简单说明一下，创建arrayOp

```
ArrayOperation arrayOp;
//相当于 ArrayOperation arrayOp = ArrayOperation();
//隐式、显示调用的区别
```
是在栈中分配内存，如果

```
ArrayOperation* arrayOp = new ArrayOperation();
```
则是在堆中分配内存，如果不

```
delete arrayOp;
```
则不会调用析构函数

### 智能指针 

#### auto_ptr

auto_ptr的特性就是在栈上构建对象，在离开作用范围时会自动析构

上面的例子可以改成

```
void test(){
	auto_ptr<ArrayOperation> arrayOp(new ArrayOperation);
	//相当于auto_ptr<ArrayOperation> arrayOp(new ArrayOperation());
	arrayOp->InitArray();
	arrayOp->ShowArray();
	cout << "test end" << endl;
}
```
这样的话不需要 delete arrayOp 也会调用析构函数来释放资源

auto_ptr有一个问题，就是被销毁时会自动删除它指向的对象，所以不能让多个auto_ptr指向同一个对象。如果有多个auto_ptr指针指向同一个对象的话便会多次删除对象。

但是若通过copy构造函数或者 copy assignment 操作符复制它们，它们就会变成null，而复制所得的指针将取得资源的唯一控制权(所有权），注意，是拷贝初始化，或者拷贝赋值才会这样，即发生了拷贝行为即原指针会变为null

```
int*p=new int(0);
auto_ptr<int>ap1(p);
auto_ptr<int>ap2(p);
```
这样ap1、ap2都指向p所指之物，危险行为，会删除p两次。

如果

```
int*p=new int(0);
auto_ptr<int>ap1(p);
auto_ptr<int>ap2 = ap1;
//或者 auto_ptr<int> ap2(ap1);
```
ap2是拷贝初始化，ap1会变为null，ap2指向p所指之物

而且因为auto_ptr的析构函数中删除指针用的是delete,而不是delete [],所以我们不应该用auto_ptr来管理一个数组指针！！

#### shared_ptr
share_ptr也是一种智能指针，自动管理对象的释放，并且可以让多个指针指向同一个对象，会有一个iOS ARC中的“引用计数”来记录这个对象被多少指针所引用，如果这个对象的引用计数变为0，则被会自动释放空间

shared_ptr跟auto_ptr一样自动管理对象的释放，并且没有auto_ptr的那些缺点，因为使用了“引用计数”，但是这样也产生了新的问题，那就是循环引用、自引用等问题

#### weak_ptr
由于shared_ptr有了循环引用等问题，于是就有了weak_ptr这种弱引用的指针，不增加引用计数

这里和iOS Property中的机制差不多，很多地方的实现思想都是相似的

#### unique_ptr
unique_ptr是只能有一个unique_ptr指向一个对象，因此unique_ptr不支持拷贝和赋值

```
unique_ptr<string> p1(new string("test"));
unique_ptr<string> p2(p1); //拷贝，错误
unique_ptr<string> p3;
p3 = p2;	//赋值，错误
```
不能拷贝或者赋值，但是可以通过 release 或 reset来转换指针的所有权

```
unique_ptr<string> p2(p1.release); //release将p1置空
unique_ptr<string> p3(new string("test"));
p2.reset(p3.release);	//reset释放了p2原来指向的内存
```

### 参考
C++中的RAII机制
Effective C++ 第13条款
C++ Primer

