---
title: 解释automaticallyAdjustsScrollViewInsets，extendedLayoutIncludesOpaqueBars，edgesForExtendedLayout的不同
date: 2017-01-19 12:36:23
tags:
---


从iOS7开始，控制器就默认添加了全屏属性，因此，你就有更多的方式去操作view的布局，具体用到的属性包括：

* edgesForExtendedLayout
* automaticallyAdjustsScrollViewInsets
* extendedLayoutIncludesOpaqueBars

-------

## `edgesForExtendedLayout`

我们可以根据以上属性设置view的铺满样式。
想象一下，默认情况下，我们从一个普通的UIViewController跳转到一个UINavigationController，view默认的展示样式是从导航栏底部开始。
但是你可以通过设置`edgesForExtendedLayout`为不同类型来控制view的样式(top, left, bottom, right)。

可以看一下例子：
```
UIViewController *viewController = [[UIViewController alloc] init];
viewController.view.backgroundColor = [UIColor redColor];
UINavigationController *mainNavigationController = [[UINavigationController alloc] initWithRootViewController:viewController];
```

你没有设置`edgesForExtendedLayout`的值，其默认的值是`UIRectEdgeAll`，所以view是延伸到整个屏幕的高度。
效果如下图：
![](https://i.stack.imgur.com/MOB6v.png)

如你所见，红色背景延伸到了状态栏（status bar）下面。
假若你将 `edgesForExtendedLayout`的值设置为`UIRectEdgeNone`，意味着你告诉view不要讲其扩展到整个屏幕。
其效果如下:

```
UIViewController *viewController = [[UIViewController alloc] init];
viewController.view.backgroundColor = [UIColor redColor];
viewController.edgesForExtendedLayout = UIRectEdgeNone;
UINavigationController *mainNavigationController = [[UINavigationController alloc] initWithRootViewController:viewController];
```
![](https://i.stack.imgur.com/ojAvO.png)

-------

关于另外一个属性`automaticallyAdjustsScrollViewInsets`.
这个属性属于UIScrollView或包含UIScrollView的控制器（比如UITableView继承自UIScrollView，UIWebView中也包含UIScrollView）。
如果你想要你的view从导航栏底部开始，但是在滑动时，让其穿透到导航栏的底部。
在这种情况下，如果你将`edgesForExtendedLayout`设置为`UIRectEdgeNone`，虽然可以从导航栏底部开始，但滑动时无法穿透到导航栏底部。

怎么办呢？

这时候就显示出`automaticallyAdjustsScrollViewInsets`的作用了。
如果你将`edgesForExtendedLayout`的值设置为`UIRectEdgeAll`，`automaticallyAdjustsScrollViewInsets`设置为YES(`edgesForExtendedLayout`默认为`UIRectEdgeAll`，`automaticallyAdjustsScrollViewInsets`默认就是YES),就能实现上述需求。
具体如下图：
![](https://i.stack.imgur.com/9Iapl.png)
上图是将`edgesForExtendedLayout`设置为`UIRectEdgeAll`，`automaticallyAdjustsScrollViewInsets`默认就是NO的情况)。

------

下图是将`edgesForExtendedLayout`设置为`UIRectEdgeAll`，`automaticallyAdjustsScrollViewInsets`默认就是YES的情况)(也就是系统默认情况)
![](https://i.stack.imgur.com/VVQHQ.png)

--------

关于另外一个属性。
字面意思是：是否延伸到包含不透明的状态栏。

* `extendedLayoutIncludesOpaqueBars`
这个值是一个补充,默认值是NO;
默认的苹果的状态栏（status bar）是透明的。如果状态栏不透明，这个试图就不回扩展到不透明的状态栏底部，除非将其值设置为YES
This value is just an addition to the previous ones. If the status bar is opaque, the views won't be extended to include the status bar too, unless this parameter is YES.
所以如果状态栏不透明，即使你设置`edgesForExtendedLayout` 为 `UIRectEdgeAll`，`extendedLayoutIncludesOpaqueBars`为NO(默认如此)，view不会延伸到状态栏底部的。
 
--------
#### 怎么判断UIScrollView在使用？
iOS会抓取控制器view的第一个子视图，（也就是index = 0），如果是UIScrollView或者UIScrollView的子类，就可以使用上文描述的属性。

如果视图是普通的UIView，可以添加一个线来解决。
```
self.navigationController.navigationBar.translucent = NO;
```


