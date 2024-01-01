---
title: RAC的一些坑
date: 2017-04-15 20:37:12
tags:
- iOS
- ReactiveCocoa
categories:
- 学习笔记
---

### 前言
记得第一个项目用RAC的时候，对MVVM还不理解，通用对RAC也不理解
最近项目中一直在用RAC，OC版的（之前叫ReactiveCocoa，现在分离出OC版的ReactiveObjc），也算入门了吧，也终于理解了MVVM
现在用的越多，发现RAC一些用法是有些坑的，总结一下

<!-- more -->

### 关于Model
刚开始学习MVVM的时候，我一直以为一个整体是四部分，Model+View+ViewModel+Controller，这里的Model不是指的通用Model（例如User,Orders那些），而是指在MVC中例如TableView中的Model，这个在MVVM中是可选的（即可有可无），我也不清楚有没有一个标准说规定要有或者不能有，我局的如果MVC扩展成MVVM的时候又不想改动太大，那可以保留，如果直接从MVVM开始的话，还是不用写的好

因为ViewModel的作用完全可以包括了Model的作用，首先我们要明白Controller只是定义View和绑定数据，而数据处理放在ViewModel，而数据我们可以在ViewModel中定义只读的属性(readonly)来提供，所以一个整体的话一般就只有Controller+View(可能直接在Controller里面定义了)+ ViewModel

### 关于绑定数据
其实也不是说用MVVM就一定要要RAC，你可以用KVO啊（如果不嫌麻烦的话），当然RAC底层就是用KVO来实现的，只不过RAC封装的更好用，使用更简便，让你轻轻松松就绑定了数据

我们先来看看 RAC(...) 和 RACObserve(...) 这两个宏，一般来说我们需要绑定数据的话，例如

```
RAC(self.label, text) = RACObserve(self.viewModel, text)
```
这样我们就把ViewModel上的text绑定到了一个label上，这是最简单的用法，关于底层那些信号流机制我就不说了，说一下我碰到的一个坑，例如现在有一个输入框，我拿到了信息后要发起请求，发起请求的入口当然是在ViewModel中，所以我们要这样绑定

```
RAC(self.viewModel, text) = self.textField.rac_textSignal;
```
这样就绑定好了，然后产品经理给你提了个需求，这是个电话输入框，你在旁边加个通讯录按钮，点完可以进通讯录读取号码显示在输入框上，然后等你写完后你发现数据绑定失效了（当然不是失效了），我们来看看rac_textSignal的实现，其中一句

```
[self rac_signalForControlEvents:UIControlEventAllEditingEvents]
```
原来这个信号只监听所有编辑事件，这种从通讯录读取的是 赋值事件，那还要这样写

```
RAC(self.viewModel, text) = RACObserve(self.textField, text)
```
刚开始学RAC的时候我以为这两种方式是一样的，但是 RACObserve(...) 是监听这个属性本身，往底层说 应该是监听这个属性的内存地址，其实我们想要的是

```
RAC(self.viewModel, text) = [self.textField.rac_textSignal merge:RACObserve(self.textField, text)]
```
然后产品经理说这个界面信息展示太多了还要输入，你弄一个可点击的输入框，一点就跳到另一个界面可以输入名字，电话号码那些信息，也要有通讯录按钮可以拉取名字，还要有历史记录，号码，输入完点确定跳转回来再展示出来就行了，对了，已经有信息的时候点进去 输入框也要默认有信息

这时候，我们就要开始改这里的绑定数据，已经有信息的时候点进去要输入框要有默认信息，那就这样

```
self.textField.text = self.viewModel.text;
RAC(self.viewModel, text) = [self.textField.rac_textSignal merge:RACObserve(self.textField, text)]
```
可是还要有历史记录，持久化那只能在ViewModel里面做，那就用双向绑定吧
在RAC中双向绑定用 RACChannel ，也有特定的宏，这样写

```
RACChannelTo(self.viewModel, text) = RACChannelTo(self.textField, text)
```
这样从通讯录拉取信息并且绑定到ViewModel的属性上了，但是我们发现输入的时候又绑定不了了，那就按上面的加一句这个？

```
RACChannelTo(self.viewModel, text) = self.textField.rac_newTextChannel;
```
然后发现输入事件绑定好了，但是从通讯录一拉取信息，发现app crash了，看了下线程栈，死循环了


那应该是 双向绑定了两次导致信号流来回触发监听事件，具体细节可以看这篇文章
RAC 中的双向数据绑定 RACChannel

所以就不能这样实现，双向绑定还有一些其他坑，假如点进去的界面还要输入详细地址，那我们用一个 textView 来做，然后发现 textView 并没有实现 rac_newTextChannel，这就尴尬了

实质上，RACChannelTo(...) 这个宏得到的是一个 RACChannelTerminal ，这个 RACChannelTerminal 相当于网络连接中的socket，只是一端，而让两个 RACChannelTerminal 则相当于让这两个 RACChannelTerminal相互订阅，那我们可以自定义一个 RACChannelTerminal吗？RAC并不允许你这么做，RACChannelTerminal的初始化方法是私有的

```
//仅声明在.m文件
- (id)initWithValues:(RACSignal *)values otherTerminal:(id<RACSubscriber>)otherTerminal;
```
无奈，我还是放弃使用双向绑定数据了，原来怎么写就怎么写吧

### 关于代理
其实RAC中已经写好取代原生代理的方法了，但是有一个很大的缺点，由于信号流是单向流动的，在订阅后实现你想要实现的代码，但是无法返回值！！也就是说，你无法用RAC写有返回值的代理方法

现在产品经理叫你在实现的 号码输入框，地址输入框添加键盘事件，return key 上面的变成next，最后的变成 done，点了后要收起键盘

我们要用到 textField 和 textView 的这个方法

```
- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string;
```
有返回值，所以就直接写， textView点击done收起键盘，那就这样写

```
- (BOOL)textView:(UITextView *)textView shouldChangeTextInRange:(NSRange)range replacementText:(NSString *)text {
    if([text isEqualToString:@"\n"]) {
        [textView resignFirstResponder];
        return NO;
    }
    return YES;
}
```
然后设置代理等于self，写上协议

运行一下，OK，这个功能就实现了
但是你发现textView的数据绑定挂了

```
[self.textView.rac_textSignal subscribeNext:^(id x) {
	NSLog(@"text--%@",x);
}];
```
订阅信号发现 信号完全挂了
百思不得其解，轮到搜索引擎出场了，Google # RAC textView textSignal
然后发现RAC的一条issue
rac_textSignal not work on UITextView #1981

你发现他和你一样是设置了代理后信号就没用了，然后看到这个


先设置代理就行了吗？！然后你试了一下，发现就是这个问题
RAC特么的还管设置代理和绑定数据先后问题！！！为什么 textField 设置在先后都可以，而 textView不行呢！！！好吧，都是RAC的错

后来我看了一下RAC代理的实现，原来他并不是把Block放到那个方法里面去执行，而是使用 Swizzle来hook那个方法

### 总结
RAC还是很强大的，很多思想值得学习，但是也有一些小坑，总之还有还多东西要学习


