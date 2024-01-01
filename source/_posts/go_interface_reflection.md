---
title: Go语言中的接口和反射
date: 2017-01-27 17:32:59
tags:
- Golang
categories:
- 学习笔记
---

### 接口 interface
接口就是一个或多个方法的集合，若某个类型的对象实现了所有的方法，那么这个对象就实现了这个接口
接口还可以存储值，只要这个类型的对象实现了这个接口，那么这个接口就可以存储这个类型的对象
又因为空接口(interface{})没有一个方法，每个类型的对象至少实现了0个或者0个以上的方法，所以空接口能存储任何值

<!-- more -->

既然接口可以被很多对象实现，那么接口就可以存储很多类型的对象或值，那函数的形参就可以用 接口类型，这样函数就可以接收实现了这个接口的多种类型的对象或值

那现在有一个以接口为形参的函数，要怎么知道这个接口是什么类型的呢？

我们可以这样判断，加入函数的形参是一个接口B类型的值b，我们需要知道b是不是A类型的，并且判断A对象是否实现了B接口

我们可以使用接口中的 .(type) 来判断类型

```
func test(b B) {
	if a, ok := b.(A); ok {
		fmt.Println("a实现了接口B",a)
	}
}
```
当前，形参也可以是空接口
空接口可以接收任何值，一个小例子

```
var x float32 = 5.3
var i interface{}
i = x
switch i := i.(type) {
case float64:
	fmt.Println("float64",i)
case float32:
	fmt.Println("float32",i)
default:
	fmt.Println("who care",i)
}
//输出 float32 5.3
```

#### 内嵌接口

和结构体中的匿名类型差不多
io包中的Writer和Reader

```
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

// ReadWriter 接口结合了 Reader 和 Writer 接口。
type ReadWriter interface {
	Reader
	Writer
}
```
### 反射 reflection
#### 从接口值到反射对象
类型-值
Type Value
reflect.TypeOf() 获取的是reflect.Type
reflect.ValueOf() 获取的是reflect.Value

```
// TypeOf 返回 interface{} 中的值的反射类型Type
func TypeOf(i interface{}) Type
```
接收参数的空接口类型（interface{}），因为任何值都有0个或0个以上的方法，可以说都实现了空接口，可以空接口可以接收任何值

reflect中有一个Kind方法，Kind() 和 TypeOf()都是返回类型，但是又有区别

```
type myInt int

func main() {
	var x myInt = 5
	a := reflect.ValueOf(x)
	fmt.Println("Type:", reflect.TypeOf(x), "Kind:", a.Kind())
}
```

会输出

```
Type: main.myInt Kind: int
```
Type能识别自定义的类型，Kind只能识别到底层的类型

关于结构体的反射对象到值
一个简单的例子(取自官方文档）

```
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```
输出

```
0: A int = 23
1: B string = skidoo
```

#### 从反射对象到接口值
给了一个 reflect.Value

有一个方法

```
func (v Value) Interface() interface{}
```
例如

```
var x int = 5
a := reflect.ValueOf(x)
fmt.Println(a.Interface())
```

要修改反射对象，其值必须可设置

```
var x float64 = 5.4
a := reflect.ValueOf(x)
a.SetFloat(5.4)
```

调用SetFloat()之后会报panic

```
panic: reflect: reflect.Value.SetFloat using unaddressable value
```
无论是想改x，还是x的副本a 这样都无法修改
但是如果想修改x，可以这样

```
var x float64 = 5.4
//取x的引用
a := reflect.ValueOf(&x)
c := a.Elem()
//这时候c是可以调用`setFloat()`的，它的canSet()是true
c.setFloat(11.1)
fmt.Println("c:", c, "x:", x)
```

输出

```
c: 11.1 x: 11.1
```

### 参考
[Go语言interface详解](https://www.jb51.net/article/56812.htm)
[反射三法则 - Go博客](https://blog.go-zh.org/laws-of-reflection)

