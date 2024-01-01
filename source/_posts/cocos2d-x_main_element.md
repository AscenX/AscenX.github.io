---
title: cocos2d-x_main_element
date: 2016-09-13 13:01:02
tags:
- Cocos2d-x
categories:
- 学习笔记
---

### 主要组件
核心组件就是 场景(Scene)，结点(Node)，精灵(Sprite)，菜单(Menu)，动作(Action)

<!-- more -->

如图

![](http://7xsnb0.com1.z0.glb.clouddn.com/2016-09-13-15%3A44%3A16.jpg)

### 导演 Director
一个游戏只有一个导演，负责控制流程和告诉接收者(recipient)怎么做，想象一下你是一个生产的执行者，告诉导演怎么做
导演一个平常的任务就是控制场景的替换和改变，导演在代码里一般是一个单例

这是一个经典的游戏流程
![](http://7xsnb0.com1.z0.glb.clouddn.com/2016-09-13-15%3A45%3A19.jpg)

对于游戏来说，我们就是一个导演，负责导演这个游戏

### 场景 Scene
场景是用渲染器(renderer)画出来的，渲染器负责渲染屏幕上的精灵和其他东西

### 场景图(Scene Graph)
场景图是一种用来安排图形场景的数据结构，包含一个结点(Node)，而结点是在场景图形树（数据结构）里面

![](http://7xsnb0.com1.z0.glb.clouddn.com/2016-09-13-15%3A48%3A16.jpg)

场景图形是一棵树，那么你可以遍历他，Cocos2d-x用中序遍历算法来遍历树

![](http://7xsnb0.com1.z0.glb.clouddn.com/2016-09-13-15%3A48%3A47.jpg)

这里要提到一个z轴的概念(z-order)，每个node都有一个z轴，z轴是用来决定渲染的顺序，值越大越后渲染
我们把在场景树左子树的结点（Node）的值设为负数(nagative)，右子树的值为正数(positive)

场景(Scene）是结点(Node)的一个集合

在代码中，构建场景图可以用`addchild()`;

```
//设置z-order的值为-2，放在左子树
scene->addChild(title_node, -2);

//如果不设置值，则默认0
scene->addChild(label_node);

//设置值为1，则放在右子树
scene->addChild(sprite_node, 1);
```

### 精灵 Sprites
所有游戏都有精灵这种东西，相当于游戏的角色

![](http://7xsnb0.com1.z0.glb.clouddn.com/2016-09-13-15%3A50%3A41.jpg)

精灵很容易创建，它们有一些可配置的属性，例如 位置，比例，旋转度，透明度，颜色等

```
//创建一个精灵
auto mySprite = Sprite::create(“mysprite.png”);

//改变属性
mySprite->setPosition(Vec2(500,0));
mySprite->setRotation(40);
mySprite->setScale(2.0);
mySprite->setAnchorPoint(Vec2(0,0));
```

设置属性这部分就不说了，很容易理解

### 动作 Actions
创建一个场景，添加了精灵只是其中一部分我们所需要做的，一个游戏要成为游戏，我们要让他动起来！
动作允许结点在时间空间上变化，你可以把精灵从一个点移动到另一个点，当完成的时候还可以设置回调
你可以创建一系列（Sequence）的动作在一个Node上执行

```
auto mySprite = Sprite::create(“Blue_Font1.png”);

//在两秒内 向右移动50像素，向上移动10像素
auto moveBy = MoveBy::create(2, Vec(50,10));
mySprite->runAction(moveBy);

//把精灵在两秒内移动到这个位置
auto moveTo = MoveTo::create(2, Vec2(50,10));
mySprite->runAction(moteTo);
```

### Sequences and Spawns
把一系列（Sequences）动作让一个精灵执行，Cocos2d-x有几种方法

我们先来看一下，如何用Sequences

```
auto mySprite = Node::create();

auto moveTo1 = MoveTo::create(2, Vec2(50,10));

auto moveBy1 = MoveBy::create(2, Vec2(100, 10));

auto moveTo2 = MoveTo::create(2, Vec2(150, 10));

//创建一个延迟
auto delay = DelayTime::create(1);

mySprite->runAction(Sequence::create(moteTo1, delay, moveBy1, delay.clone(), moveTo2, nullptr));
```

这样就能让一个精灵按顺序执行一系列的动作，那如何同时执行呢？
那就是Spawn，Spawn可以让所有动作同时执行

```
auto myNode = Node::create();

auto moveTo1 = MoveTo::create(2, Vec2(50,10));
auto moveBy1 = MoveBy::create(2, Vec2(100, 10));
auto moveTo2 = MoveTo::create(2, Vec2(150, 10));

myNode->runAction(Spawn::create(moveTo1,moveBy1, moveTo2, nullptr));
```

当你的角色有大量动作要同时执行的时候便可以这样用

### 父子关系 Parent Child Relationship
Cocos2d-x用父子关系来描述 属性的改变是父节点对子节点作用的

下图是一个蓝精灵和它的两个孩子

![](http://7xsnb0.com1.z0.glb.clouddn.com/2016-09-13-15%3A54%3A45.jpg)

当父精灵的旋转度改变的时候 子精灵的也会改变

![](http://7xsnb0.com1.z0.glb.clouddn.com/2016-09-13-15%3A55%3A03.jpg)

```
auto myNode = Node::create();

//旋转
myNode->setRotation(50);
```

当然，如果你改变父精灵的缩放，子精灵的也会改变

![](http://7xsnb0.com1.z0.glb.clouddn.com/2016-09-13-15%3A55%3A39.jpg)

### 输出消息 Logging as a way to output messages
有时候，当你的app运行起来了，你需要看到控制台的输出，可以用log();
例如

```
//简单的字符串
log("Thi would be outputted to the console");

//字符串和变量
string s = “My variable”;
log("string s %s", s);

double dd = 42;
log(“double is %f”, dd);

int i = 5;
log(“integer is %d”, i);

float f = 2.0f;
log(“float is %f”, f);

bool b = true;
if(b == true)
     log("bool is true");
else
     log(“bool is false”);
```
     
当然也可以用c++的std::cout，虽然log()的输出格式更简单操作

### 参考资料

[Cocos2d-x官方文档](http://cocos2d-x.org/docs/cocos2d-x/en/index.html)


