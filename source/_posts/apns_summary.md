---
title: APNS原理总结
date: 2016-07-29 21:51:44
tags:
- APNS
categories:
- 学习笔记
---

### 什么是APNs
先说一下远程推送，一般我们有自己的服务器，在这个过程中是Provider的角色，如图，推送从我们的服务器到我们的APP的过程就是要通过APNs来发送

<!-- more -->

![2016-07-29_10:34:17.jpg](http://7xsnb0.com1.z0.glb.clouddn.com/2016-07-29_10:34:17.jpg)

APNs(Apple Push Notification service)是远程推送功能的核心，通过APNs客户端和苹果服务器建立一个长连接，推送也是通过这个长连接发送到客户端上

### deviceToken
deviceToken是设备的一个标识符，属于你这款APP装在你这个设备上的标识符，即每个APP在每一个不同的设备上都有着不同的deviceToekn，通过注册远程推送服务，APNs会返回给你的APP的deviceToken，如图

![2016-07-29_09:22:33.jpg](http://7xsnb0.com1.z0.glb.clouddn.com/2016-07-29_09:22:33.jpg)

在项目的AppDelegate里面有一个方法，如果成功注册了便可以接收到deviceToken

```
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
```
deviceToken不是唯一确定的，当你的设备更新了系统然后deviceToken就会改变

### 安全架构
为了保证安全性，APNs用连接信任(connection trust)和token信任(token trust)来控制通信入口，你要用APNs则必须用通过这两种验证。

#### 连接信任
连接信任第一个作用是保证APNs连接的provider是苹果已经同意可通信的，然后第二个是保证与APNs连接的设备的合法性，第二步是APNs处理的，你所要处理的是provider与APNs之间的连接安全性。

##### 服务器与APNs之间的连接信任
每个服务器都必须要有唯一的provider证书和私钥，都是用来验证连接的。provider证书就是在开发者官网申请的。

服务器通过TLS验证和APNs连接，HTTPS中用的也是TLS协议，即四步握手，首先初始化TLS连接，即provider(服务器)发送请求给APNs，APNs服务器返回APNs证书（即公钥）给provider，然后服务端收到后生成provider证书后再返回给APNs，APNs收到后验证后即可以建立TLS连接，不过APNs用的不是HTTPS，而是HTTP/2，具体过程如下图

![2016-07-29_22:13:00.jpg](http://7xsnb0.com1.z0.glb.clouddn.com/2016-07-29_22:13:00.jpg)

##### APNs与设备之间的连接信任
设备和provider一样，都有私钥和设备证书，当设备激活了就保存在钥匙串(keychain)中，但是这部分的连接信任你是不用管的，由APNs负责，也是通过TLS来验证，过程和provider和APNs相同

![2016-07-29_23:35:06.jpg](http://7xsnb0.com1.z0.glb.clouddn.com/2016-07-29_23:35:06.jpg)

#### Token信任
Token信任就是保证通知从合法的起点推送到合法的终点，Token信任即要用到上面说的deviceToken，deviceToken提供给provider，然后之后你的provider每次发送要推送的通知都要携带deviceToken，这就是token信任

APNs会用token钥匙去保证通知来源（即你的服务器）的合法性，用包含在deviceTokenl里面的device ID去确定目标设备的身份，过程如下图

![2016-07-29_23:41:02.jpg](http://7xsnb0.com1.z0.glb.clouddn.com/2016-07-29_23:41:02.jpg)

### 参考资料
图全部来自官方文档，很多关键词都是我自行翻译

苹果官方文档-Apple Push Notification Service

