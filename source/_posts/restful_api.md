---
title: 基于RESTful API的设计模式小实践
date: 2017-02-19 20:27:29
tags:
- Golang
- RESTful API
categories:
- 学习笔记
---

## RESTful API

REST是REpresentational State Transfer的缩写，表现层状态转移，这里的表现层指的是资源
我们设计API的时候每一个url都要指向一个资源
<!-- more -->
例如现在有一个订单（资源）
http://example.com/api/v1/orders
而我们的操作由HTTP的请求方法来决定
v1为API版本号

- GET http://example.com/api/v1/orders
就是获取所有的订单
- GET http://example.com/api/v1/orders/123
获取ID为123的订单
- POST http://example.com/api/v1/orders
创建一个新的订单
- POST http://example.com/api/v1/orders/123
更新ID为123的订单
- PATCH http://example.com/api/v1/orders/123
修改ID为123的订单
- DELETE http://example.com/api/v1/orders/123
删除ID为123的订单
也可以在后面添加参数修改订单状态
- POST http://example.com/api/v1/orders/123/cancel
设置这个订单的状态为取消
- POST http://example.com/api/v1/orders/123/finish
设置这个订单的状态为完成
总之都是围绕 http://example.com/api/v1/orders 这个URL来设计，一句话总结就是使用URL来定位资源，使用HTTP请求方法来操作

### 小实践
接下来我用Go简单的写了个小服务器
定义了一个Orders类型，有Id，Name,Price三个属性

```
type Orders struct {
	Id    int
	Name  string
	Price int
}
```
RESTful相关处理url使用的是github.com/gorilla/mux这个库，非常简便

```
r.HandleFunc("/api/v1/orders/{id}", getOrders).Methods("GET")
r.HandleFunc("/api/v1/orders", getOrders).Methods("GET")
r.HandleFunc("/api/v1/orders", createAnOrders).Methods("POST")
r.HandleFunc("/api/v1/orders/{id}", updateOrders).Methods("PATCH")
r.HandleFunc("/api/v1/orders/{id}", delteOrders).Methods("DELETE")
```
数据库用的是postgreSQL，用了对象关系映射库gorm
然后简单的存了点模拟数据

![ascen.me_restful](http://7xsnb0.com1.z0.glb.clouddn.com/2017-02-19-072547.jpg)


服务器监听的是8080端口
然后我就不写前端了，直接使用curl命令在控制台操作

使用GET操作获得所有订单
```
curl localhost:8080/api/v1/orders
```

![ascen.me_restful](http://7xsnb0.com1.z0.glb.clouddn.com/2017-02-19-134318.jpg)

使用GET操作加上ID获取一个订单
```
//获取id为123的订单
curl localhost:8080/api/v1/orders/123
```

![ascen.me_restfule](http://7xsnb0.com1.z0.glb.clouddn.com/2017-02-19-134257.jpg)

使用POST创建订单

```
curl -X POST --data id=666 --data name="test123" --data price=998 localhost:8080/api/v1/orders
```

![ascen.me_restfule](http://7xsnb0.com1.z0.glb.clouddn.com/2017-02-19-134444.jpg)

这是服务器的输出

![ascen.me_restfule](http://7xsnb0.com1.z0.glb.clouddn.com/2017-02-19-134523.jpg)

数据库可以看到

![ascen.me_restfule](http://7xsnb0.com1.z0.glb.clouddn.com/2017-02-19-133845.jpg)

使用DELETE操作删除一个订单
```
curl -X DELETE localhost:8080/api/v1/orders/123
```
数据库更新

![ascen.me_restfule](http://7xsnb0.com1.z0.glb.clouddn.com/2017-02-19-134802.jpg)

使用PATCH操作修改订单的数据
```
curl -X PATCH --data name=haobang localhost:8080/api/v1/orders/234
```
数据库

![ascen.me_restfule](http://7xsnb0.com1.z0.glb.clouddn.com/2017-02-19-135159.jpg)





