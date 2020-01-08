---
title: 深入runtime探究KVO
date: 2015-09-29 12:18:27
tags: runtime, KVO
---


## 前言
Objective-C 中的键(key)-值(value)观察(KVO)并不是什么新鲜事物,它来源于设计模式中的`观察者模式`,其基本思想就是:

>一个目标对象管理所有依赖于它的观察者对象,并在它自身的状态改变时主动通知观察者对象。这个主动通知通常是通过调用各观察者对象所提供的接口方法来实现的。观察者模式较完美地将目标对象与观察者对象解耦。

在 Objective-C 中有两种使用键值观察的方式:手动或自动,此外还支持注册依赖键(即一个键依赖于 其他键,其他键的变化也会作用到该键)。下面将一一讲述这些,并会深入 Objective-C 内部一窥键值 观察是如何实现的。

--------
## 观察者（Observer）
>**观察者（Observer）**
>定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并自动更新。
在 `Cocoa Touch` 框架中通知和 `KVO` 都实现了观察者模式。**通知是由一个中心对象为所有观察者提供变更通知，`KVO` 是被观察的对象直接向观察者发送通知。**
如上图，`Subject` 的值改变时，通知观察者 `ObserverA`，`ObserverB`，`ObserverC`，我的数据改变了，依赖我的你们需要更新状态了。
>`被观察者不需要知道有多少个观察者和观察者的更新细节，降低被观察者和观察者之间的耦合`。

![observer.png](http://upload-images.jianshu.io/upload_images/1467716-696c1f5f6fb7cab7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 运用键值观察
### 1,注册与解除注册
如果我们已经有了包含可供键值观察属性的类,那么就可以通过在该类的对象(被观察对象)上调用名 为 NSKeyValueObserverRegistration 的 category 方法将观察者对象与被观察者对象注册与解除 注册:

```
- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath opt ions:(NSKeyValueObservingOptions)options context:(void *)context;
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath;
```

这两个方法的定义在 `Foundation/NSKeyValueObserving.h` 中,`NSObject`,`NSArray`,`NSSet` 均实现了以上方法,因此我们不仅可以观察普通对象,还可以观察数组或结合类对象。在该头文件中,我们还 可以看到 NSObject 还实现了 `NSKeyValueObserverNotification` 的 `category` 方法(更多类似方 法,请查看该头文件):

```
 - (void)willChangeValueForKey:(NSString *)key;
 - (void)didChangeValueForKey:(NSString *)key;
```
>**注意：**不要忘记解除注册,否则会导致资源泄露。

### 2,设置属性
将观察者与被观察者注册好之后,就可以对观察者对象的属性进行操作,这些变更操作就会被通知给观察 者对象。注意,只有遵循 KVO 方式来设置属性,观察者对象才会获取通知,也就是说遵循使用属性的 setter 方法,或通过 key-path 来设置:

```
//必须是set方法
target.age = 30;
[target setAge:30];
[target setValue:[NSNumber numberWithInt:30] forKey:@"age"];
```

###  3,处理变更通知
观察者需要实现名为 `NSKeyValueObserving` 的 `category` 方法来处理收到的变更通知:

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context;
```

>**注意：**在这里`change` 这个字典保存了变更信息,具体是哪些信息取决于注册时 的 `NSKeyValueObservingOptions`。

## KVO的内部实现原理：
KVO是基于`runtime`机制实现的，当某个类的属性对象第一次被观察时，系统就会在运行期间动态地创建该类的一个派生类，在这个派生类中重写基类的任何被观察属性的`setter`方法。派生类在被重写的setter方法内实现真正的通知机制
如果原类为`Person`，那么生成的派生类名为`NSKVONotifying_Person`。

我们知道，每一个类中都有一个`isa`指针指向当前类，所有系统就是在当一个类的对象第一次被观察的时候，系统就会偷偷将`isa`指针指向动态生成的派生类，从而在被监听属性赋值时被执行的是派生类的`setter`方法
键值观察通知依赖于`NSObject` 的两个方法: `willChangeValueForKey:` 和 `didChangevlueForKey:`;在一个被观察属性发生改变之前， `willChangeValueForKey:` 一定会被调用，这就 会记录旧的值。而当改变发生后，`didChangeValueForKey:` 会被调用，继而 `observeValueForKey: ofObject: change: context:` 也会被调用。
>**补充：**KVO的这套实现机制中苹果还偷偷重写了class方法以“欺骗”外部调用者，让我们误认为还是使用的当前类，然后系统将这个对象的 isa 指针指向这个新诞生的派生类,因此这个对象就成为该派生类的对象了,因而在该对象上对 `setter` 的调 用就会调用重写的 `setter`,从而激活键值通知机制。此外,派生类还重写了 `dealloc` 方法来释放资源。

## 写在最后

技术学习绝不能孤胆英雄独闯天涯，而应在一群人的交流碰撞，享受智慧火花的狂欢。
希望我的文章能成为你的盛宴，也渴望你的建议能成为我的大餐。



