---
title: 轻量化ViewController，读文章做的总结
date: 2015-10-13 22:37:22
tags:
- iOS
- Objective-C
categories:
- 学习笔记
---


轻量化ViewControllers顾名思义就是把ViewController的代码进行简化让控制器更简单更清晰

<!-- more -->

### 把DataSource和其他protocol分离出控制器

因为做项目很多控制器都TableViewController，所以必须要有数据源(TableViewDataSource)输入，一般DataSource的3个方法都在控制器里
我们的目的就是要把他分离出来，方法就是自定义一个类，然后控制器调用这个类并且这个数据源类是通用的，别的TableViewController或者有定义了tableView/collectionView的控制器都可以调用自定义ArrayDataSource类

1.定义一个Block，用于传递cell和数据

```
typedef void (^TableViewCellPassBlock)(id cell, id item);
```

2.定义一个方法给控制器调用，用于初始化ArrayDataSource类（需要暴露在.h文件中）

```
@property (nonatomic, strong) NSArray *items;
@property (nonatomic, copy) NSString *cellIdentifier;
@property(nonatomic, copy) TableViewCellPassBlock passBlock;
/**

 *  初始化数据源的方法
 *  @param anItems         用于接受数据的数组
 *  @param aCellIdentifier 接收cell的ID
 *  @param aPassBlock      接收传递的Block
 */
 - (id)initWithItems:(NSArray *)anItems cellIdentifier:(NSString *)aCellIdentifier passBlock(TableViewCellPassBlock)aPassBlock {
    self = [super init];
	if(self){
		self.items = anItems;
		self.cellIdentifier = aCellIdentifier;
		self.passBlock = [aPassBlock copy];
	}
	return self;
}
```

3.然后就是数据源的方法

```
/** 这个方法找出数组的数据 后面要用到*/
- (id)itemAtIndexPath:(NSIndexPath *)indexPath
{
	return self.items[(NSUInteger) indexPath.row];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
	return self.items.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
	UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier: self.cellIdentifier forIndexPath: indexPath];
	id item = [self itemAtIndexPath: indexPath];
	self.passBlock(cell, item);
	return cell;
}
```

ArrayDataSource类就这样定义完成了
接下来是控制器的调用
只需要写以下几行代码便可以设置tableView的数据源，不用每个控制器都写那三个方法

```
//数组传数据是接下来要简化的，后面会详细说明
NSArray *arr = [AppDelegate sharedDelegate].returnModel.returnData;
//使用Block回调，传递cell和模型数据
TableViewCellPassBlock passBlock = ^(TestModelCell *cell,TestModel *model){
	//测试方法，只是传递model给cell，这个方法用分类写的，自定义cell的分类
	[cell setName: model];
};
//初始化并设置tableView的数据源
self.dataSource = [[ArrayDataSource alloc] initWithItems: arr cellIdentifier:@"TestModelCell" passBlock: passBlock];
self.tableView.dataSource = self.dataSource;
```

### 二、将业务逻辑放到model中

一些跟控制器无关的代码可以放到模型中
测试代码我写了一个TestModel，只有一个属性和一个初始化方法

```
@property (nonatomic, copy) NSString *name;

- (instancetype)initWithName:(NSString *)name;
```

然后又写了一个model，因为要返回数据所以我叫ReturnModel
在.h文件中暴露的方法

```
//类方法，创建的时候用这个方法
+ (instancetype)returnModel;

//这个方法用于返回数据
- (NSArray *)returnData;
```

然后在.m文件中

```
+ (instancetype)returnModel
{
    return [[self alloc] init];
}

- (id)init
{
    self = [super init];
    if (self) {
	//调用方法执行业务逻辑的方法
	[self doSomething];
    }
    
    return self;
}

//可以在这个方法中执行与控制器无关的业务逻辑
- (void)doSomething
{
    NSLog(@"try to do something");
}

//在这个方法中返回模型数据，数据略

- (NSArray *)returnData
{
　　NSArray *array = [[NSArray alloc] initWithObject:...];
　　return array;
}
```

思路就是在返回数据的模型类中执行业务逻辑，在创建类的时候执行
在AppDelegate中添加方法和属性，并且暴露在.h文件中

```
+ (instancetype)sharedDelegate
{
	return [UIApplication sharedApplication].delegate;
}

@synthesize returnModel = _returnModel;

- (ReturnModel *)returnModel
{
	if (_returnModel == nil) {
		_returnModel = [ReturnModel returnModel];
	}
	return _returnModel;
}
```

所以控制器直接调用方法得到数据

```
NSArray *arr = [AppDelegate sharedDelegate].returnModel.returnData;
```

这时候控制器已经很简化了
可以把网络请求也放在model层

### 参考

[Obj中国的#1](https://objccn.io/issue-1-1/)

### 探索

文中只写了3钟方法，后来我在用的时候要用到其他数据源方法的时候想，为什么不把其他方法也写进去。
把数据源完全分离开来。
例如 Section的title
接口方法

```
- (void)setSelectIndexTitle:(NSArray *)selectIndexTitles;
```

用一个数据接收便可

```
- (void)setSectionTitle:(NSArray *)groupTitles
{
	self.groupTitles = groupTitles;
}
```

然后在titleForHeaderInSection中调用即可

```
- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section
{
	NSString *title = self.groupTitles[section];
	return title;
}
```

