---
title: Dart中的异步编程
date: 2019-02-03 01:25:59
tags:
- Dart
categories:
- 学习笔记
---

## Dart中的异步

最近在写Flutter，结果发现Dart的异步编程还和我目前学过的都不太一样，简单总结一下。

<!-- more -->

### Future 

#### 什么是 Future

`Future` 就是一个`Future<T>` 对象，代表就是一个异步操作返回的结果，类型是T
如果返回的是不可用的值，则应该为`Future<void>`

举个例子

```dart

void main() {
  print('test1:${DateTime.now()}');
  testDelay();
  print('test2:${DateTime.now()}');
}

testDelay() {

  Future<String> f = testFuture();
  f.then((str){
    print(str);
  });
  print('testDelay:${DateTime.now()}');
}

Future<String> testFuture() {
    // 延迟3秒
   return Future.delayed(Duration(seconds: 3),() { 
      return 'future:${DateTime.now()}';
   });
}

```

输出结果

```
test1:2019-02-03 03:03:41.206838
testDelay:2019-02-03 03:03:41.226762
test2:2019-02-03 03:03:41.227339
三秒后...
future:2019-02-03 03:03:44.227833
```

这就是`Future`的用法，其中用到了`then`，如果需要使用链式编程，可以这样

```
expensiveA()
    .then((aValue) => expensiveB())
    .then((bValue) => expensiveC())
    .then((cValue) => doSomethingWith(cValue));
```

### async / await

`Dart`在1.9后，新加了两个关键字，`async`和`await`，即使不用`Future`也能让异步编程看起来就像同步编程一样

上面的例子改成使用`async`和`await`

```
void main() {
  print('test111:${DateTime.now()}');
  testDelay();
  print('test2${DateTime.now()}');
}

testDelay() async {
  print('testDelay:${DateTime.now()}');
  String str = await testFuture();
  print(str);
}

Future<String> testFuture() {
   return Future.delayed(Duration(seconds: 3),() {
      DateTime time = DateTime.now();
      return 'future:$time';
   });
}
```

同样的输出效果
就相当于直接从`Future`获取到了值，这样与原来不同的是，在`await`执行后的代码都被延时后执行，这样也可以使用`await`来保证执行顺序

```
main() async {
  await expensiveA();
  await expensiveB();
  doSomethingWith(await expensiveC());
}
```
保证了执行顺序是 A -> B -> C

`Dart`中的异步编程除了`Future`还有`Stream`，接下来我们看一下`Stream`，`Stream`在函数式编程中非常常见

### Streams

#### 什么是Stream

一个`Stream`就是数据的一个异步的序列（数据流），序列用户生成的事件或者从文件读取的数据

怎么生成一个`stream`呢，我们来看一下`yield`关键词

#### yield

`yeild`在异步编程会生成一个`stream`，需要用`async*`来标识，`yield`的功能可以理解成一个叠加器，会不断累积前面的结果，加入新的数据，最后返回一个包含所有结果的容器

```
Stream<int> testYield() async* {
  for (int i = 0; i < 15;++i) {
    yield i;
  }
}

// 要使用 `await for` 必须要在异步函数中
test() async {
  Stream<int> st = testYield();
  // 使用 await for 遍历stream的内容
  await for (int item in st) {
    print(item);
  }
}
```


如果在`yield`在同步方法中使用会怎么样，那生成的就不是`stream`，而是一个`Iterable`对象

#### Iterable

使用`yield`必须用`sync*`来标识，然后不能在

```
test() {
  Iterable it = testYield();
  it.forEach(print);

  // 也可以用for来遍历
  // for (var item in it) {
  //   print(item);
  // }
}

Iterable testYield() sync* {
  for (int i = 0; i < 15;++i) {
    yield i;
  }
}
```


#### yield*

如果我们要使用递归，对`yield`的结果进行`yield`会怎么样，看这个例子

```dart
void main() {
  test();
}

test() {
  Iterable it = testYield(10);
  it.forEach(print);
}

testYield(int n) sync* {
  if (n > 0) {
    yield n;
    yield testYield(n - 1);
  }
}
```

输出结果

```
(9, (8, (7, (6, (5, (4, (3, (2, (1, ())))))))))
```

这并不是我们想要的结果，看来对`yield`的结果进行`yield`则是直接把`yield`生成的容器进行了`yield`，我们需要把`yield`生成容器的结果进行累加，则需要使用 `yield*`

修改为

```
void main() {
  test();
}

test() {
  Iterable it = testYield(10);
  it.forEach(print);
}

testYield(int n) sync* {
  if (n > 0) {
    yield n;
    yield* testYield(n - 1);
  }
}
```

这样就能达到我们想要的结果了。


###  参考

[dart官方文档](https://www.dartlang.org/articles/language/beyond-async)