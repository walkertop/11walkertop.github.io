---
title: 史上最全ReactiveCocoa(RAC)之信号类源码解析
date: 2016-03-17 12:01:13
tags: RAC
---


信号signal是RAC的绝对核心，所有的操作都是围绕着信号来处理的。
比如：创建信号，订阅信号，发送信号是消息发送的核心步骤。
常见的三个信号类为：

> * RACSignal
> * RACSubject
> * RACReplaySubject

# 一、RACSignal
## 代码实现：
```
// 1.创建信号
    RACSignal *siganl = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
       // 注：block在此仅仅是个参数，未被调用，
       //当有订阅者订阅信号时会调用block。
// 2.发送信号
        [subscriber sendNext:@1];
        // 如果不在发送数据，最好发送信号完成，内部会自动调用[RACDisposable disposable]取消订阅信号。
        return nil;
    }];
// 3.订阅信号,才会激活信号.
    [siganl subscribeNext:^(id x) {
        // block调用时刻：每当有信号发出数据，就会调用block.
        NSLog(@"接收到数据:%@",x);
    }];
```

--------

## 使用步骤：

### 1.创建信号

```
+(RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe;```
```
内部原理：
* 1.1 方法解析：方法名为createSignal：其返回值类型为RACSignal类的实例变量。参数为一个名字为didSubscribe的block。
* 1.2 方法参数didSubscribe解析：(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {}的返回值是RACDisposable类型的实例变量，参数为id类型且遵守且遵守RACSubscriber协议的subscriber。
* 1.3 创建信号，首先把didSubscribe这个block保存到信号中，但不会触发。此时didSubscribe仅仅作为方法的参数，并没有被触发，所以信号也仅仅是一个冷信号，block内部不会执行。（具体执行见下文）

源码解析：
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
//创建了一个RACDynamicSignal类的信号
RACDynamicSignal *signal = [[self alloc] init];
//将代码块保存到信号里面（但此时仅仅是保存，没有调用）
signal->_didSubscribe = [didSubscribe copy];
return [signal setNameWithFormat:@"+createSignal:"];
}
```
--------

### 2.订阅信号
>（激活信号，冷信号编程热信号）
> - (RACDisposable *)subscribeNext:(void (^ )(id x))nextBlock；

```
* 2.1当信号被订阅，也就是调用signal的subscribeNext:nextBlock，
* 2.2nextBlock内部创建了订阅者subscriber，并且把nextBlock保存到subscriber中。
//  RACSignal.h
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock {
NSCParameterAssert(nextBlock != NULL);
 //内部创建了RACSubscriber（订阅者）类的实例对象o，并且将nextBlock保存到o中，在返回值出执行o,实际也是执行了nextBlock。
RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:NULL completed:NULL];
return [self subscribe:o]; //内部执行了nextBlock，具体见下文
}

+ (instancetype)subscriberWithNext:(void (^)(id x))next error:(void (^)(NSError *error))error completed:(void (^)(void))completed {
 RACSubscriber *subscriber = [[self alloc] init];
//将block保存到subscriber中
 subscriber->_next = [next copy];
 subscriber->_error = [error copy];
 subscriber->_completed = [completed copy];

 return subscriber;
}

//执行
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
 NSCParameterAssert(subscriber != nil);

 RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
 subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];
//判断有无self.didSubscribe,有则执行该self.didSubscribe，意味着将订阅者subscriber发送过去
 if (self.didSubscribe != NULL) {
  RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
   RACDisposable *innerDisposable = self.didSubscribe(subscriber);
   [disposable addDisposable:innerDisposable];
  }];

  [disposable addDisposable:schedulingDisposable];
 }
 
 return disposable;
}

```

--------

### 3.发送信号
> 订阅信号时，将subscriber传递给didSubscribe的参数subscriber
> 
> 利用[subscriber sendNext:@1]方法发送信号。
> 

```
//  RACSubscriber.h
// These callbacks should only be accessed while synchronized on self.
@property (nonatomic, copy) void (^next)(id value);

//名为next的block是返回值为void，参数为id类型的value，在sendNext:内部，将next复制给nextBlock，执行该方法后，subscribeNext:的block参数才会被调用。

- (void)sendNext:(id)value {
 @synchronized (self) {
  void (^nextBlock)(id) = [self.next copy];
  if (nextBlock == nil) return;
  //执行nextBlock，发送value
  nextBlock(value);
 }
}
```

--------
## RACSignal原理流程图：
![RACSignal实现原理图.png](http://upload-images.jianshu.io/upload_images/1467716-d6ea285e8b921f99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


--------
## RACSignal总结：
三步骤（先创建信号，然后订阅信号，最后执行didSubscribe内部的方法）顺序是不能变的。
>
>RACSignal底层实现：
>
    * 1.创建信号，首先把didSubscribe保存到信号中，还不会触发。
    * 2.当信号被订阅，也就是调用signal的subscribeNext:nextBlock
      2.1 subscribeNext内部会创建订阅者subscriber，并且把nextBlock保存到subscriber中。
       2.2 subscribeNext内部会调用siganl的didSubscribe
    * 3.siganl的didSubscribe中调用[subscriber sendNext:@1];
    * 3.1 sendNext底层其实就是执行subscriber的nextBlock
    
--------

# 二、RACSubject
> RACSubject:信号提供者，自己可以充当信号，又能发送信号。

先来简单回顾下，RACSignal类中发送和订阅信号是两步完成的。

```
//发送信号的为遵守RACSubscribe协议的对象subscriber完成发送
[subscriber sendNext:@1];
//订阅信号的为RACSignal的实例对象
[siganl subscribeNext:^(id x) {}；
```

* 疑问：有没有办法让一个对象既能发送也可以发送消息呢？

其实很简单，只要让具有RACSignal对象遵守RACSubscribe协议，就既能发送，又能订阅信号了。
这个类就是RACSubject。虽然和RACSignal一样都具有订阅和发送信号的能力，但其内部原理不同。
下文是对RACSubject类进行剖析。

    
## 代码实现
> 调用顺序不变，但是可以创建多个订阅者，并发送信号；
 
```
 // 1.创建信号
    RACSubject *subject = [RACSubject subject];   
// 2.订阅信号（这里可以创建多个订阅者）
    [subject subscribeNext:^(id x) {
        // block调用时刻：当信号发出新值，就会调用.
        NSLog(@"第一个订阅者%@",x);
    }];
    [subject subscribeNext:^(id x) {
        // block调用时刻：当信号发出新值，就会调用.
        NSLog(@"第二个订阅者%@",x);
    }];
     [subject subscribeNext:^(id x) {
        // block调用时刻：当信号发出新值，就会调用.
        NSLog(@"第三个订阅者%@",x);
    }];
// 3.发送信号
    [subject sendNext:@"1"];
```
## 源码解析

### 1.创建信号

```
//  RACSubject.m
+ (instancetype)subject {
 return [[self alloc] init];
}

- (id)init {
 self = [super init];
 if (self == nil) return nil;
 
 _disposable = [RACCompoundDisposable compoundDisposable];
 
 _subscribers = [[NSMutableArray alloc] initWithCapacity:1];
 
 return self;
}

```

### 2.订阅信号
> 不同类型的信号，创建订阅者的方式不同，RACSignal订阅信号时，调用了形影的block。
> 
> 而RACSubject订阅信号的实质就是将内部创建的订阅者保存在订阅者数组self.subscribers中，仅此而已。订阅者对象有一个名为nextBlock的block参数。

```
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock {
 NSCParameterAssert(nextBlock != NULL);
 
 RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:NULL completed:NULL];
 return [self subscribe:o];
}

- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
 NSCParameterAssert(subscriber != nil);

 RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
 subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];

 NSMutableArray *subscribers = self.subscribers;
 @synchronized (subscribers) {
 //将订阅者保存在订阅者数组中
  [subscribers addObject:subscriber];
 }
 
 return [RACDisposable disposableWithBlock:^{
  @synchronized (subscribers) {
   // Since newer subscribers are generally shorter-lived, search
   // starting from the end of the list.
   NSUInteger index = [subscribers indexOfObjectWithOptions:NSEnumerationReverse passingTest:^ BOOL (id<RACSubscriber> obj, NSUInteger index, BOOL *stop) {
    return obj == subscriber;
   }];

   if (index != NSNotFound) [subscribers removeObjectAtIndex:index];
  }
 }];
}
```

### 3.发送信号
> 底层实现是：
>  
  1. 先遍历订阅者数组中的订阅者;
  2. 后执行订阅者中的nextBlock;
  3. 最后让订阅者发送信号。
 

```
//  RACSubject.m

- (void)sendNext:(id)value {
//顺序A：遍历保存在数组中的订阅者对象
 [self enumerateSubscribersUsingBlock:^(id<RACSubscriber> subscriber) {
//顺序E：调用订阅者，执行sendNext:方法
  [subscriber sendNext:value];
 }];
}

//顺序B:遍历subscribers数组中的订阅者对象，执行block。
- (void)enumerateSubscribersUsingBlock:(void (^)(id<RACSubscriber> subscriber))block {
 NSArray *subscribers;
 @synchronized (self.subscribers) {
  subscribers = [self.subscribers copy];
 }
//顺序C:遍历数组，取出subscriber的block，并且执行。
 for (id<RACSubscriber> subscriber in subscribers) {
//顺序D,执行block
  block(subscriber);
 }
}
```

--------

## RACSubject原理流程图：

![RACSubject原理图 .png](http://upload-images.jianshu.io/upload_images/1467716-f2b4f5f91210d0ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



--------

## 总结

```
RACSubscriber:表示订阅者的意思，用于发送信号，这是一个协议，不是一个类，只要遵守这个协议，并且实现方法才能成为订阅者。通过create创建的信号，都有一个订阅者，帮助他发送数据。

RACDisposable:用于取消订阅或者清理资源，当信号发送完成或者发送错误的时候，就会自动触发它。
```
> RACSubject:底层实现和RACSignal不一样。

* 1.调用subscribeNext订阅信号，只是把订阅者保存起来，并且订阅者的nextBlock已经赋值了。
    
* 2.调用sendNext发送信号，遍历刚刚保存的所有订阅者，一个一个调用订阅者的nextBlock。
* 3.由于本质是将订阅者保存到数组中，所以可以有多个订阅者订阅信息。

使用场景:不想监听某个信号时，可以通过它主动取消订阅信号。
RACSubject:信号提供者，自己可以充当信号，又能发送信号。

使用场景:通常用来代替代理，有了它，就不必要定义代理了。

> **缺点：**
> 还是必须先订阅，后发送信息。订阅信号就是创建订阅者的过程，如果不先订阅，数组中就没有订阅者对象，那就通过订阅者发送消息。

--------

# 三、RACReplaySubject

## 介绍
上文中提到了，RACSubject要求先订阅，后发送信号。
**##****代码实现：**

```
// 1.创建信号
    RACReplaySubject *subject = [RACReplaySubject subject];
// 2.订阅信号
    [subject subscribeNext:^(id x) {
        NSLog(@"%@",x);
    }];
    // 遍历所有的值,拿到当前订阅者去发送数据
// 3.发送信号
    [subject sendNext:@1];
    // RACReplaySubject发送数据:
    // 1.保存值
    // 2.遍历所有的订阅者,发送数据
    
    // RACReplaySubject:可以先发送信号,在订阅信号
```
## 使用步骤：

### 1.创建信号
```
//  RACReplaySubject.m

+ (instancetype)subject {
 return [[self alloc] init];
}

- (instancetype)init {
 return [self initWithCapacity:RACReplaySubjectUnlimitedCapacity];
}
//此时调用的子类RACReplaySubject的初始化方法
- (instancetype)initWithCapacity:(NSUInteger)capacity {
 self = [super init];
 if (self == nil) return nil;
 
 _capacity = capacity;
//会用_valuesReceived这个数组保存值value
 _valuesReceived = (capacity == RACReplaySubjectUnlimitedCapacity ? [NSMutableArray array] : [NSMutableArray arrayWithCapacity:capacity]);
 
 return self;
}
```

### 2.订阅信号
> 遍历拿到保存在数组中的所有值，然后调用subscriber发送信号。

```
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock {
 NSCParameterAssert(nextBlock != NULL);
 
 RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:NULL completed:NULL];
 return [self subscribe:o];      //创建订阅者的方式不同
}

//RACReplaySubject类对象执行[self subscribe:o]
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
 RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];

 RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
  @synchronized (self) {
//先遍历self.valuesReceived的所有的值，后让subscriber调用sendNext:发送信号。
   for (id value in self.valuesReceived) {
    if (compoundDisposable.disposed) return;

    [subscriber sendNext:(value == RACTupleNil.tupleNil ? nil : value)];
   }

   if (compoundDisposable.disposed) return;

   if (self.hasCompleted) {
    [subscriber sendCompleted];
   } else if (self.hasError) {
    [subscriber sendError:self.error];
   } else {
    RACDisposable *subscriptionDisposable = [super subscribe:subscriber];
    [compoundDisposable addDisposable:subscriptionDisposable];
   }
  }
 }];

 [compoundDisposable addDisposable:schedulingDisposable];

 return compoundDisposable;
}
```

### 3.发送信号

```
- (void)sendNext:(id)value {
 @synchronized (self) {
//重点：发送信号的时候，会先将值value保存到数组中，
  [self.valuesReceived addObject:value ?: RACTupleNil.tupleNil];
//调用父类发送（先遍历订阅者，然后发送值value）
  [super sendNext:value];
  
  if (self.capacity != RACReplaySubjectUnlimitedCapacity && self.valuesReceived.count > self.capacity) {
   [self.valuesReceived removeObjectsInRange:NSMakeRange(0, self.valuesReceived.count - self.capacity)];
  }
 }
}

```
## RACReplaySubject原理图

![RACReplaySubject原理图.jpg](http://upload-images.jianshu.io/upload_images/1467716-df43b214a0a047c4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## RACReplaySubject总结
RACReplaySubject是RACSubject的子类。

RACReplaySubject使用步骤:

```ubject使用步骤:

// 1.创建信号 [RACSubject subject]，跟RACSiganl不一样，创建信号时没有block。
// 2.可以先订阅信号，也可以先发送信号。
  // 2.1 订阅信号 - (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock
  // 2.2 发送信号 sendNe
```


