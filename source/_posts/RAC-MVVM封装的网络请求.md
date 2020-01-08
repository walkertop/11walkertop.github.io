---
title: RAC+MVVM封装的网络请求
date: 2016-03-22 12:11:29
tags: RAC MVVM
---

### 1.MVVM 代替 MVC 原因
无论MVC还是MVVM，包括其他设计模式，核心目的是为了提高代码的简洁性，降低耦合度。
简单点说就是让专门的人去做专门的事情。
比如MVC模式中，
>* M (model)
>* V (view)
>* C (controller)

但在MVC中，通过网络请求回来的数据会放到Model中，作为数据源来调用和处理。
但还是存在控制器中文件很大的问题。很多业务逻辑都写到了控制器上了，不利于程序之间的解耦，而且在比较大的项目中，代码的可读性也比较差，而MVVM的引入大大减少了这个问题，会让C释放释放出来，关于视图方面的业务逻辑交给VM处理，C只用来解决控制器之间的连接问题。

### 2.RAC如何处理和传输数据
那在RAC中怎么处理和传送数据呢？
RAC最核心的内容是信号。我们可以把网络请求回来的数据通过信号传递和发送出去。
我们把网络请求回来的数据叫做`responseObject`。
基于RAC(  想深入探究 RAC 原理可点击[史上最全ReactiveCocoa(RAC)之信号类源码解析](http://www.jianshu.com/p/7cf4754cebee))的知识，我们让订阅者发送数据，然后让信号接收数据，便完成数据的传递。
同时RAC中有`RACCommand`的类，负责处理事件。
所有总体可以分为三步：

* 1. 网络请求，获得数据`responseObject`；
* 2. 订阅者将`responseObject`发送出去；
* 3. 信号订阅信号（接收发送处理的数据）。

### 3.代码实例
至于代码层面是怎么解决这三个方面的问题呢?

我们假设一个使用场景：
豆瓣上有开放的API，当我们查询图书的时候，当搜索"美女"关键词的时候，会出现很多关于美女的图书。
然后将其显示在tableView上。

主要的业务逻辑包括：
> * 1. 通过AFN请求数据
> * 2. 将请求回来的数据传递给控制器
> * 3. 控制器的tableView完成数据的显示


#### 3.1AFN请求数据

```
        AFHTTPRequestOperationManager *mgr = [AFHTTPRequestOperationManager manager];
        [mgr GET:@"https://api.douban.com/v2/book/search" parameters:@{@"q":@"美女"} success:^(AFHTTPRequestOperation * _Nonnull operation, NSDictionary * _Nonnull responseObject) {
            NSLog(@"发送成功");
            NSArray *dictArray = responseObject[@"books"];
            [subscriber sendNext:dictArray];

```

--------

#### 3.2请求回来的数据传给控制器

在MVC中，通常会将请求回来的数据`responseObject`进行初步的处理，放到`model`模型中，然后`tableView`的数据源也来自于model，最终完成`tableView`的绘制和展示。

这里我们通过MVVM的方式，将网络请求的业务逻辑放到VM中处理。在V中实现`tableView`的数据源方法。

至于事件的处理就交给RAC来解决。RAC中有一个类`RACCommond`，来处理事件。
#### 3.3具体代码
处理的代码如下：


```
//通过RACCommand获取数据
- (void)getData {
    self.requestCommand = [[RACCommand alloc]initWithSignalBlock:^RACSignal *(id input) {
        RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
            
        AFHTTPRequestOperationManager *mgr = [AFHTTPRequestOperationManager manager];
        [mgr GET:@"https://api.douban.com/v2/book/search" parameters:@{@"q":@"美女"} success:^(AFHTTPRequestOperation * _Nonnull operation, NSDictionary * _Nonnull responseObject) {
            NSLog(@"发送成功");
            NSArray *dictArray = responseObject[@"books"];
            [subscriber sendNext:dictArray];

            } failure:^(AFHTTPRequestOperation * _Nonnull operation, NSError * _Nonnull error) {
                NSLog(@"发送失败");
            }];
            return nil;
        }];
        return signal;
    }];
}
```


```
//控制器接受传回来的数据
- (void)getDataFromRequestVM {
    RACSignal *signal = [self.requestVM.requestCommand execute:nil];
    
    [signal subscribeNext:^(id x) {
        self.dataArray = x;
        [self.tableView reloadData];
    }];
}
```

### 4.注意：
为了避免外部修改，可以使用readOnly
以上操作可以分步处理，也可以通过RACCommand的类来处理。

### 5.github地址
[demo下载地址](https://github.com/walkertop/RAC-MVVM-)

