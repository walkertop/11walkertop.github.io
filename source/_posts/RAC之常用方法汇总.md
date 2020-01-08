---
title: RAC之常用方法汇总
date: 2016-03-21 12:06:10
tags: RAC
---

# 一、iOS内部对不同事件的处理
iOS中对不同事件作出响应时，会用不同的方式来处理这些业务逻辑。
比如按钮的点击使用action，ScrollView滚动使用delegate，属性值改变使用KVO，通知等系统提供的方式。
虽然说是对事件做出相应，但iOS内部需要用不同的方法，时常用起来非常的繁琐。其实这些事件，都可以通过RAC处理。

# 二、RAC的核心介绍
RAC内部的核心类是`RACSiganl`。
RACSiganl:信号类,一般表示将来有数据传递，只要有数据改变，信号内部接收到数据，就会马上发出数据。

`ReactiveCocoa`为事件提供了很多处理方法，处理起来非常简单，而且代码放在一起，就不需要跳到对应的方法里。非常便于管理，也很符合我们开发中`高聚合，低耦合`的思想。

在介绍RAC底层时，我们已经对RAC的实现原理做了说明，本文会介绍RAC对不同事件的各种处理方式。

# 三、简单介绍不同的方法

## ReactiveCocoa开发中常见用法

### 3.1 代替代理.

`rac_signalForSelector：`用于替代代理。
### 3.2代替KVO :

`rac_valuesAndChangesForKeyPath：`用于监听某个对象的属性改变。
### 3.3 监听事件:

`rac_signalForControlEvents：`用于监听某个事件。
### 3.4 代替通知:

`rac_addObserverForName:`用于监听某个通知。
### 3.5 监听文本框文字改变:

`rac_textSignal:`只要文本框发出改变就会发出这个信号。
### 3.6 处理当界面有多次请求时，需要都获取到数据时，才能展示界面
`rac_liftSelector:withSignalsFromArray:Signals:`当传入的Signals(信号数组)，每一个signal都至少sendNext过一次，就会去触发第一个selector参数的方法。
使用注意：几个信号，参数一的方法就几个参数，每个参数对应信号发出的数据。
## 四、代码演示

```
 // 1.代替代理
    // 需求：自定义redView,监听红色view中按钮点击
    // 之前都是需要通过代理监听，给红色View添加一个代理属性，点击按钮的时候，通知代理做事情
    // rac_signalForSelector:把调用某个对象的方法的信息转换成信号，就要调用这个方法，就会发送信号。
    // 这里表示只要redV调用btnClick:,就会发出信号，订阅就好了。
    [[redV rac_signalForSelector:@selector(btnClick:)] subscribeNext:^(id x) {
        NSLog(@"点击红色按钮");
    }];

  // 2.KVO
    // 把监听redV的center属性改变转换成信号，只要值改变就会发送信号
    // observer:可以传入nil
    [[redV rac_valuesAndChangesForKeyPath:@"center" options:NSKeyValueObservingOptionNew observer:nil] subscribeNext:^(id x) {
        NSLog(@"%@",x);
    }];

  // 3.监听事件
    // 把按钮点击事件转换为信号，点击按钮，就会发送信号
    [[self.btn rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {
       NSLog(@"按钮被点击了");
    }];

  // 4.代替通知
    // 把监听到的通知转换信号
    [[[NSNotificationCenter defaultCenter] rac_addObserverForName:UIKeyboardWillShowNotification object:nil] subscribeNext:^(id x) {
        NSLog(@"键盘弹出");
    }];

  // 5.监听文本框的文字改变
   [_textField.rac_textSignal subscribeNext:^(id x) {
       NSLog(@"文字改变了%@",x);
   }];

  // 6.处理多个请求，都返回结果的时候，统一做处理.
    RACSignal *request1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        // 发送请求1
        [subscriber sendNext:@"发送请求1"];
        return nil;
    }];

    RACSignal *request2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        // 发送请求2
        [subscriber sendNext:@"发送请求2"];
        return nil;
    }];

    // 使用注意：几个信号，参数一的方法就几个参数，每个参数对应信号发出的数据。
    [self rac_liftSelector:@selector(updateUIWithR1:r2:) withSignalsFromArray:@[request1,request2]];
}
// 更新UI（该方法有要求，有多少个信号就要求有多少个参数，参数的内容就是发送的数据。）
- (void)updateUIWithR1:(id)data r2:(id)data1
{
    NSLog(@"更新UI%@,%@",data,data1);
}
```

# 文末：
这里只是对RAC常用方法合集的简单描述和基础使用的介绍，至于RAC信号的原理，可以参考作者的[《史上最全RAC之信号类源码解析》](http://betteris.top/2016/03/17/%E5%8F%B2%E4%B8%8A%E6%9C%80%E5%85%A8ReactiveCocoa-RAC-%E4%B9%8B%E4%BF%A1%E5%8F%B7%E7%B1%BB%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)。
至于RAC的bind(绑定)，map（映射），concat（组合）等高级用法，可参考作者的其它文章。


