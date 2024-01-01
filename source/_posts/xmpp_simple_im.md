---
title: xmpp_simple_im
date: 2016-03-10 20:01:05
tags:
- Objective-C
- XMPP
categories:
- 学习笔记
---

### 搭建服务器和数据库

搭载服务器我用的是openfire，数据库用的是mysql

这里推荐两篇文章

配置mysql，用的是`mysql workbench`

http://justsee.iteye.com/blog/1753467

配置服务器 `Openfire`

http://www.cnblogs.com/xiaodao/archive/2013/04/05/3000554.html

<!-- more -->

先配置好数据库然后配置服务器
然后两个都打开

![xmpp-1](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-1.png)

数据库也打开

![xmpp-2](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-2.png)

下一个XMPP客户端，就是用来测试的
我下的是Adium

![Adium](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-3.png)

[这里下](https://xmpp.org/software/clients.html)

### 配置

然后在Adium里面添加帐号，服务器要用openfire设置好的127.0.0.1，端口用5222

这里openfire里面的服务器端口，5222是客户端连接服务器

![server](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-4.png)

登录成功后可以在openfire里面看到用户已经在线

![online](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-5.png)

然后另一客户端，我直接用代码，觉得麻烦没做界面

简单实现
设置XMPPStream

```
- (XMPPStream *)stream
{
    if (!_stream) {
        _stream = [[XMPPStream alloc] init];
        [_stream addDelegate:self delegateQueue:dispatch_get_current_queue()];
    }
    return _stream;
}
```
然后设置帐号，只是测试，所以直接在viewDidLoad里面写了

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self.stream setMyJID:[XMPPJID jidWithString:@"test123@127.0.0.1"]];
    [self.stream setHostName:@"127.0.0.1"];
    [self.stream setHostPort:5222];
    
    NSError *error = nil;
    [self.stream connectWithTimeout:1.0f error:&error];
    
    if (error) {
        NSLog(@"connectWithTimeout : %@",error);
    }
}
```

然后就是验证密码，下面的方法是上线，这些是代理方法，记得设置XMPPStream的代理

```
- (void)xmppStreamDidConnect:(XMPPStream *)sender
{
    NSError *error = nil;
    [self.stream authenticateWithPassword:@"test123" error:&error];
    if (error) {
        NSLog(@"authenticateWithPassword : %@",error);
    }
}


- (void)xmppStreamDidAuthenticate:(XMPPStream *)sender
{
    XMPPPresence *presence = [XMPPPresence presence];
    [self.stream sendElement:presence];
}
```

这是接受信息的方法，我没做界面，直接打印出来接受的信息

```
- (void)xmppStream:(XMPPStream *)sender didReceiveMessage:(XMPPMessage *)message
{
    NSString *msg = [[message elementsForName:@"body"] lastObject];
    NSLog(@"%@",msg);
}
```

如果验证失败的话，会调用这个方法

```
- (void)xmppStream:(XMPPStream *)sender didNotAuthenticate:(DDXMLElement *)error
{
    NSLog(@"didNotAuthenticate : %@",error);
}
```

后来在openfire查看用户名必须带 服务器名

![serverlist](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-6.png)

例如我的帐号是test123，设置JID的用户名就是`test123@127.0.0.1`

```
[self.stream setMyJID:[XMPPJID jidWithString:@"test123@127.0.0.1"]];
```

然后运行，没有报错，在openfire显示已经在线

![online2](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-7.png)

然后就可以开始聊天了，下面的是刚发的，上面的是之前的聊天记录

发了文字和一个链接

![chat](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-8.png)

然后Xcode输出

![](http://7xsnb0.com2.z0.glb.clouddn.com/xmpp-9.png)

即时通讯最简单的就这样，退出APP后openfire后台就会显示下线


