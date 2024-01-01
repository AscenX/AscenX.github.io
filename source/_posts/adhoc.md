---
title: Ad Hoc测试
date: 2015-10-27 19:04:40
tags:
- 开发调试
categories:
- 学习笔记
---

由于最近某个项目需要给别人测试，使用的是Ad Hoc方法
首先登录开发者官网配置证书

<!-- more -->

#### 1.添加Certificates，从电脑获取certSigningRequest然后添加进去
#### 2.在Identifiers里面的App IDs添加要调试App的Bundle ID和名字
#### 3.在Devices里面添加要给别人测试的手机的UDID
前几步都和真机调试一样，简单说明一下

#### 4.在Provisioning Profiles 里面添加profiles的时候选择Ad Hoc
![Ad_Hoc](http://7xsnb0.com2.z0.glb.clouddn.com/756114-20151027100118310-497055954.png)

continue
　　选择要调试的App的App ID

continue
　　选择开发者

continue
　　选择要调试的手机（可以多选）

continue
　　输入保存的证书的名字，然后Generate，生成之后要下载下来 　　

#### 5.保存下来证书之后，打开Xcode，双击证书
#### 6.在Xocde-Preferences-Account里面添加刚才设置证书的开发者帐号，如果已添加就跳过此步骤
#### 7.然后在项目PROJECT-Build Settings-Code Signing里面的Provisioning Profile里面添加下载的Ad Hoc证书
如果没有这个选项的话，再次双击生成的Ad Hoc证书

Provisioning Profile上面的开发者全部选择设置了证书的开发者帐号

![Code_Signing](http://7xsnb0.com2.z0.glb.clouddn.com/756114-20151027101624154-609807275.png)

连接真机，看是否可以正常运行

#### 8.Xcode菜单栏中选择Product-Archive，即会弹出Organizer窗口（已经Archive过了可以从菜单栏Window-Organizer打开）
然后选择Export，选择Save For Ad Hoc Deployment，然后Next

![organizer](http://7xsnb0.com2.z0.glb.clouddn.com/756114-20151027102331247-1201210840.png)

#### 9.会弹出窗口让你选择开发者，选择设置过证书的开发者
#### 10.导出
![device_support](http://7xsnb0.com2.z0.glb.clouddn.com/756114-20151027102524200-945274110.png)

#### 11.选择Next
![summary](http://7xsnb0.com2.z0.glb.clouddn.com/756114-20151027102748544-523448479.png)

1框框内会显示配置好的Ad Hoc证书，如果显示*，则出现问题，请登录开发者官网查看Ad Hoc证书是否valid（如果invalid则edit一下为valid）

如果显示正确，就直接Next

#### 12.然后就生成APP的ipa了，这个程序就可以拿去给别人测试了。


