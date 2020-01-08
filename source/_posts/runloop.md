---
title: 对，我就是RunLoop
date: 2016-06-08 18:37:28
tags: iOS RunLoop
---

```
#以下是RunLoop和iOS搬砖程序员的访谈#
```



> "请问你是？"

 "不用请问，我就是RunLoop"

> “你好，我是iOS开发者，我听说过你，不过抱歉，对你的名声我早有耳闻，只是不很熟悉。”

 ”嗯，不难理解。毕竟我在幕后，你在台前，我是说句不妄言的话，没有我，你们就别想玩的转。“
> ”哦 ？ 这么说的话，我确实很好奇，请问你能不能介绍下你自己！


## RunLoop的概念
人如其名，我就是RunLoooooooooooooooop,像是一个死循环，不停的跑圈，永不懈怠。除非程序不启动，或者你们代码写的太差，以至于crash,我才不得不停止了。

> 可是程序启动与否和你有什么关系？

 程序启动伊始，有一段代码：
```
int main(int argc, char * argv[]) {
     @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([YourAppDelegate class]));
    }
}
```

这段代码永远不会执行结束，不然程序也停止了。所以我伴随着程序的启动，一直存在。
当然，我不可能无意义的瞎跑。我贯穿整个程序，在奔跑的过程中帮忙处理各种事情。

---

> 我主观上听明白了你的意思，但冒昧的说一句，除了瞎跑我还真不知道你到底做了啥？

我理解，我理解，毕竟你们花费大量的时间和UIKit和Foundation的各种类打交道，丝毫不顾及我的存在感。但你有没有好奇过，你的事件响应，各种手势识别，定时器都是怎么传递的啊？
> 抱歉，你这么一问，我确实欠考虑了。

对啊，所以这就是我从幕后走向台前的目的。写代码不能只看表面，还要挖挖本质。话说回来，官方点说呢，我就是消息机制的处理模式，从线程start到线程end，一直在循环检测，检测inputSource(如点击，双击等操作)同步事件，检测timeSource同步事件，检测到输入源后会执行处理函数，首先会产生通知，CoreFunction向线程添加runLoop Observers来监听事件，意在监听事件发生时来做处理。

## RunLoop和线程之间的关系
> 麻烦停一下，我只知道很多事情是线程来做的，比如页面刷新交给主线程，异步线程来下载。但听你的口气，这些功劳都是你的了？

对，从表面看来你们都是在操作线程，但这只是表面。我和线程是绑定在一起的。每个线程都有一个对应的 Runloop 对象。当然不同线程的RunLoop是有区别的，主线程（也是你们常说的UI线程）的 Runloop 会在应用启动时完成启动，其他线程的 Runloop 默认并不会启动，只是在需要使用时，你们手动启动。

## RunLoop不同的Mode
> 你的意思是，每一个线程都有一个RunLoop，但默认情况下只有主线程的才会开启。

对，不仅每个线程都有RunLoop，而且每一个 RunLoop都包含若干个 Mode，这么给你解释吧，你肯定玩过LOL吧，知道里面有`鞋子`的装备吧。
> 这个倒忘不掉。

那就好说了，正常来说，每个鞋子都有自己偏重的功能属性，不同英雄都只会买一种鞋子对不对。我也一样，我虽然有多个Mode，但就像穿鞋一样，每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 RunLoop，再重新指定一个 Mode 进入。这就像换鞋子一样，必须先脱掉旧鞋才能穿上新鞋。
> 当然，鞋子不同，有的偏重法术，有的偏重攻速，你所谓的Mode是怎么区分不同的？

 其实每个 Mode 包含若干个 `Source`/`Timer`/`Observer`，这些都属于Mode的item，item不同，Mode也不同，RunLoop分为五类。

```
NSDefaultRunLoopMode    //大多数工作中默认的运行方式
NSConnectionReplyMode    //使用这个Mode去监听NSConnection对象的状态，我们很少需要自己使用这个Mode
NSModalPanelRunLoopMode    //在Model Panel情况下去区分事件(OS X开发中会遇到)
NSEventTrackingRunLoopMode    //跟踪来自用户交互的事件（比如UITableView上下滑动）
NSRunLoopCommonModes    //这是一个伪模式，其为一组run loop mode的集合
```

虽然模式很多，但iOS中公开暴露出来的只有 NSDefaultRunLoopMode 和 NSRunLoopCommonModes。 NSRunLoopCommonModes 实际上是一个 Mode 的集合，默认包括 NSDefaultRunLoopMode 和 NSEventTrackingRunLoopMode。

> 这里就有点抽象了，这么多Mode你是怎么选择的呢？

主线程的 RunLoop 里有两个预置的 Mode：NSDefaultRunLoopMode和 NSEventTrackingRunLoopMode。这两个 Mode 都已经被标记为"Common"属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。
> 但如果我想滑动ScrollView时，不影响Timer咋办？

 既然在DefaultMode下，不影响Timer，TrackingRunLoopMode下能滑动，两者都不影响，就是两种模式都要就ok了，iOS中的commonModeItems就是就是将DefaultMode和TrackingRunLoopMode组合在一起了，因此只用切换到commonModeItems就ok了。

---

## RunLoop的内部逻辑
> 哈哈，果真我对你了解的太不到位了，原来你是超神的存在。

不不不，超神不至于，其实我也只是给系统跑腿罢了。但我的一举一动也要接受管理，不能随便乱来。
> 谁还能管的了你啊？

怎么管不了，能力大，责任大。我要时时刻刻接受系统的监督。你们对我的了解可能从NSRunLoop开始的，但这实际只是OC对我简单的封装，我的底层是C语言库CFRunLoop，这里面有一个叫CFRunLoopObserverRef的观察者，也就是前面我给你提到的`Observer`，当我的状态发生改变时，观察者就会记录我的变化。

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```

> 这个倒不难理解，毕竟UIView，UIController，UIApplication都有类似的管理。

## RunLoop与NSTimer的准时触发

> 可是RunLoop与NSTimer有什么关系呢？

NSTimer其实是一种资源，但它要想起作用必须添加到RunLoop中。
> NSTimer会是准时触发事件吗？

timer不是一种实时的机制，会存在延迟，而且延迟的程度跟当前线程的执行情况有关。
> 这个怎么理解？

正常情况下，你指定一个事件2秒之后触发，但若是此时恰好有一个大规模的连续耗时运算，那timer的执行必然要等到该连续事件处理结束才会开始执行，此时你就无法保证NSTimer的准时触发了。当然这只是针对于一次执行的timer，
```
[NSTimer scheduledTimerWithTimeInterval:1 target:testObject2 selector:@selector(timerAction:) userInfo:nil repeats:YES];
```
对于重复性事件，情况也不一样。比如一个程序，你设置了周期性1秒触发，但是有个耗时事件用时两秒，此时就无法准确触发，并且以后会随着这个延迟继续延迟。

## RunLoop的相关实战

> 说了那么多，我大约感受到你的神奇魔力了，但还是过于抽象和偏理论，有没有具体的实例来彰显你的存在感啊！

那是必然，那我虎躯一震，抖一抖我的黑魔法。给你说几个具体的场景吧。
### AutoreleasePool的真谛
问你个问题：从MRC的手动管理内存，到ARC的自动管理，其关键因素是什么？
> 因为多了AutoreleasePool，自动释放池。只是我也不知道AutoreleasePool背后到底帮助我们做了什么？

哈哈，其实这和我RunLoop有很大的关系，在App启动后，会在主线程度的RunLoop帮我们创建两个Observer。
第一个 Observer 只监视了一个事件：监听事件在Entry(即将进入Loop)期间，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。
第二个 Observer 监视了两个事件：在BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的，被这些inputSource和timeSource包裹。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。
> 你这么一说我就理解了，难怪之前有人告诉我，内存的释放是在每次RunLoop结束之后呢。

这算是一个场景，当然还有其它的。

### 关于`performSelecter:afterDelay: `方法
> 好的，你接着说。

 你有没有遇到过这样的场景，我在子线程中执行`performSelecter:`，一切ok，但加了延时，执行`performSelecter:afterDelay: `方法时，却愣是没反应？
> 对对对，后来别人告诉我在子线程开了RunLoop就ok了，至于为何，我现在还是云里雾里的。

其实这和NSTimer有关，当调用 NSObject 的 `performSelecter:afterDelay:` 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。而子线程默认没有开启RunLoop，就无法执行timer事件，自然就不执行。
> 原来如此。

### GCD
> 既然聊到了线程问题，我想问下线程之间的通信问题。比如我在异步子线程执行了网络请求，想把请求回来的结果通过异步主线程`dispatch_async(dispatch_get_main_queue(), block)`，的方式将block回调给主线程，这应该也很你们有关系吧。

对的，对于子线程和主线程之间的通信。当调用 `dispatch_async(dispatch_get_main_queue()`, block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__() 里执行这个 block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

