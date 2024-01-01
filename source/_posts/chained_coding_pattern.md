---
title: 链式编程思想
date: 2016-04-10 00:17:31
tags:
- iOS
- Objective-C
categories:
- 学习笔记
---
### 链式编程是什么
链式编程就是将调用多个方法用点语法连接起来，让代码更加简洁和可读性更高
刚开始接触链式编程是Masonry，用起来真的非常爽

<!-- more -->

```
make.left.right.top.equalTo(self.view);
```

这样一句语句就调用了4个方法
- .left调用了left属性的get方法
- .right, .top调用了right和top方法
- .equalTo()调用了equalTo方法

这种写法极大简化了写约束的方式

### 原理
原理就是调用的属性的类型或者方法的返回类型为原调用属性的类型
例如说UILabel调用了某个方法或者属性，得到的类型还是UILabel，那么还可以继续调用UILabel的属性或者方法

### 如何实现
看了Masonry，我发现有两种实现方式

#### 1.用点语法调用方法
这个其实我之前没发现，写习惯了用方括号调用方法
例如创建一个label 可以这样写UILabel *label = [[UILabel alloc] init
其实也可以这样写UILabel *label = UILabel.alloc.init
不过后种方法几乎没人用，苹果应该也不推荐这种写法，因为有时候这样写是没有代码提示的
但是有一个缺点就是不能调用有参数的方法，所以我们只能写没有参数的方法

创建一个UIButton的分类
写两个方法

```
- (UIButton *)setTextHello;
- (UIButton *)setTextColorRed;
```
并且实现

```
- (UIButton *)setTextColorRed
{
    [self setTitleColor:[UIColor redColor] forState:UIControlStateNormal];
    return self;
}

- (UIButton *)setImage
{
    [self setImage:[UIImage imageNamed:@"Stan1"] forState:UIControlStateNormal];
    return self;
}
```

然后我们创建一个按钮的时候就可以这样写

```
//创建一个按钮
UIButton *button = [UIButton buttonWithType:UIButtonTypeInfoLight];
button.frame = CGRectMake(100, 100, 100, 100);
button.setTextHello.setTextColorRed;
[self.view addSubview:button];
```

效果如图

![chian-1](http://7xsnb0.com2.z0.glb.clouddn.com/chian-1.png)

但是会报一个警告，因为调用的是属性，但是这个属性没有被用到
解决方法是在调用属性前面加(void),这样就可以了

![chian-2](http://7xsnb0.com2.z0.glb.clouddn.com/chian-2.png)

#### 2.用属性调用
新创建一个UILabel的分类
如果要传入参数的话，就返回一个UILabel的block，可以在block里面实现你想要实现的东西

```
@property(nonatomic,copy) UILabel* (^kText)(NSString *text);
@property(nonatomic,copy) UILabel* (^kFont)(NSUInteger fontSize);
@property(nonatomic,copy) UILabel* (^kTextColor)(UIColor *color);
```

加k是为了易于区分，可以不加的
因为这是在分类里面的属性，不会生成setter和getter方法，所以都要自己写
实现如下

```
- (UILabel *(^)(NSUInteger))kFont
{
    return ^(NSUInteger font){
        [self setFont:[UIFont systemFontOfSize:font]];
        return self;
    };
}

- (UILabel *(^)(NSString *))kText
{
    return ^(NSString *text){
        [self setText:text];
        return self;
    };
}

- (UILabel *(^)(UIColor *))kTextColor
{
    return ^(UIColor *color){
        [self setTextColor:color];
        return self;
    };
}

- (void)setKFont:(UILabel *(^)(NSUInteger))kFont{}
- (void)setKText:(UILabel *(^)(NSString *))kText{}
- (void)setKTextColor:(UILabel *(^)(UIColor *))kTextColor{}
```

不会用到setter方法，也不能用，所以setter方法设为空
然后就能愉快的链式编程了

![chian-3](http://7xsnb0.com2.z0.glb.clouddn.com/chian-3.png)

然后我再想，假如没有参数呢，刚开始是想block没有参数

```
@property(nonatomic,copy) UILabel* (^aText)();
@property(nonatomic,copy) UILabel* (^aFont)();
@property(nonatomic,copy) UILabel* (^aTextColor)();
```

实现

```
- (UILabel *(^)())aFont
{
    return ^(){
        [self setFont:[UIFont systemFontOfSize:16.0f]];
        return self;
    };
}

- (UILabel *(^)())aTextColor
{
    return ^(){
        [self setTextColor:[UIColor cyanColor]];
        return self;
    };
}

- (UILabel *(^)())aText
{
    return ^(){
        [self setText:@"这还是一个label"];
        return self;
    };
}

- (void)setAFont:(UILabel *(^)())aFont{}
- (void)setAText:(UILabel *(^)())aText{}
- (void)setATextColor:(UILabel *(^)())aTextColor{}
```
后来发现调用的时候还是要这样
![chian-4](http://7xsnb0.com2.z0.glb.clouddn.com/chian-4.png)

尝试了一下，最后一个调用能去掉括号，报警告，能运行，但是没效果

所以我想为什么不直接调用属性

```
@property(nonatomic,strong) UILabel *bText;
@property(nonatomic,strong) UILabel *bFont;
@property(nonatomic,strong) UILabel *bTextColor;
```

getter && setter

```
- (UILabel *)bText
{
    [self setText:@"labellabel"];
    return self;
}

- (UILabel *)bTextColor
{
    [self setTextColor:[UIColor purpleColor]];
    return self;
}

- (UILabel *)bFont
{
    [self setFont:[UIFont systemFontOfSize:13.0f]];
    return self;
}

- (void)setBFont:(UILabel *)bFont{}
- (void)setBText:(UILabel *)bText{}
- (void)setBTextColor:(UILabel *)bTextColor{}
```
调用

```
UILabel *label3 = [[UILabel alloc] initWithFrame:CGRectMake(100, 400, 200, 100)];
(void)label3.bTextColor.bText.bFont;
[self.view addSubview:label3];
```
如果调用不加(void)还是会报警告
如果想像Masonry那样的链式编程可以这样写

```
UILabel *label3 = [[UILabel alloc] initWithFrame:CGRectMake(100, 400, 200, 100)];
label3.bTextColor.bText.kFont(30);
[self.view addSubview:label3];
```

这样就不会报警告,整洁清晰

