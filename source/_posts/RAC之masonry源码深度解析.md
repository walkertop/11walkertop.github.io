---
title: RAC之masonry源码深度解析
date: 2016-03-19 11:52:58
tags:
---
-------

写在前面：
本文不是讲解masonry的基础使用，而是希望借着masonry的源码解析给大家渗透链式编程的思想和展示其具体实现。
现在RAC（ReactiveCocoa）很火，借着这个成熟的案例让大家窥其一斑，作者在此抛砖引用，供大家交流参考。

# **一、NSLayoutConstraint约束**

实际iOS用`NSLayoutConstraint`对控件进行约束。比如：想要让子控件的顶部距离父控件顶部10pt，添加约束的实际条件就是满足`subView.top = superView.top * 1 + 10`这个公式就可以了。
`NSLayoutConstraint`的实际就是对该公式的代码解释，代码如下:

```
    NSLayoutConstraint *topConstraint = [NSLayoutConstraint constraintWithItem:subView
                                 attribute:NSLayoutAttributeTop
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:self.view
                                 attribute:NSLayoutAttributeTop
                                multiplier:1.0
                                                                   constant:padding.top];
    [self.view addConstraints:topConstraint];
```

但我们需要对控件的top,bottom,left,right进行约束就特别麻烦。在OC中有一个库`Masonry`对`NSLayoutConstraint`进行了封装，********（****Swift****中使用****SnapKit****，****SnapKit****其实就是****Masonry****的****Swift****版本，实现思路大体一致。）********

# 二：masonry介绍
masonry是iOS布局控件的轻量级框架。其原理是通过链式调用的方式对`NSLayoutConstraint`进行封装，简化了控件的约束方式。

抓住两头：
其实massory最终还是利用苹果官方提供的`NSLayoutConstraint`，只是利用链式编程的方式进一步封装。

> 接下来思考两个问题
>1. 怎么通过封装？
>2. 链式编程来实现约束的添加的？

接下来我们就对masonry的封装做进一步解释。
## 1.masonry添加约束的代码实现

```
- (void)viewDidLoad {
    [super viewDidLoad];
    UIView *subView = [[UIView alloc]init];
    subView.backgroundColor = [UIColor purpleColor];
    
    //先添加控制，后设置约束，不然找不到约束的依赖，会报错。
    [self.view addSubview:subView];
    [subView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.top.equalTo(@20);
        make.right.bottom.equalTo(@-10);
    }];
}
```

## 2.masonry方法执行步骤解析：

> * 子控件调用`mas_makeConstraints`方法，`mas_makeConstraints`方法有个block参数（返回值为void,参数为`MASContraintMaker`的实例对象make）；
> * block作为方法的参数就是隐式调用（block并没有真正调用，需要在方法内部，block()调用一次，才会真正执行block）；
> * block的有一个MASContraintMaker类的实例make作为参数，让make去添加约束；
> * MASContraintMaker类中有个可变数组的属性，用于保存约束；
> * 执行`mas_makeConstraints`传入进行的block；
> * 遍历数组中的约束，完成约束的安装；

--------

以上只是文字描述了执行的大致步骤，具体的代码实现是怎么样的呢？
我们接下里通过三个问题来展开。
## 3.疑问
```
> 1. make的点语法代表什么意思？
> 2. 为什么可以连续用点语法？
> 3. 具体代码解析是什么样的？
```

--------

### 问题一：make的点语法代表什么意思？
`make.left.top.equalTo(@20);`

实质就是`MASContraintMaker`类的实例对象make调用了属性的getterter方法。
扒开源码我们会看到

```
@interface MASConstraintMaker : NSObject

@property (nonatomic, strong, readonly) MASConstraint *left;
@property (nonatomic, strong, readonly) MASConstraint *top;
//省略了bottom,right，baseline等属性。

@end
```
--------

```
//getter方法，返回的是MASConstraint对象，getter方法调用 addConstraintWithLayoutAttribute:
- (MASConstraint *)left {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeLeft];
}

//返回的是MASConstraint对象，接着调用constraint: addConstraintWithLayoutAttribute:方法
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}

- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
    MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
    if ([constraint isKindOfClass:MASViewConstraint.class]) {
        //replace with composite constraint
        NSArray *children = @[constraint, newConstraint];
        MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
        compositeConstraint.delegate = self;
        [self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
        return compositeConstraint;
    }
    if (!constraint) {
        newConstraint.delegate = self;
        
        //
        [self.constraints addObject:newConstraint];
    }
    return newConstraint;
}
```

### 问题二：为什么可以连续用点语法？

>**链式编程的核心：**
每个点语法实际调用的getter方法，getter方法的返回值为实例对象本身，然后继续调用getter方法，就成为链式了。
结合代码进行具体解释：`make.left.top.equalTo(@10);`
我们对其分开解释：点语法返回的时一个新的约束newConstraint。

```
`make.left.top.equalTo(@10);`
//分开写就为
newConstraint1 = make.left;
newConstraint2 = newConstraint1.top;
newConstraint2.equalTo(@10);
```
--------

```
- (MASConstraint * (^)(id))equalTo {
    return ^id(id attribute) {
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}
```

### 问题三：整个方法的具体调用步骤是什么样的？

首先解释下`MASConstraintMaker`类：
> `MASConstraintMaker`类就是一个工厂类，负责创建`MASConstraint`类型的对象（依赖于`MASConstraint`接口，而不依赖于具体实现）

--------
>**粗略步骤：**
>1. UIView的类调用`mas_makeConstraints`方法
>2. `mas_makeConstraints`有个block参数，会做隐式回调
>3. 获得约束数组，通过install安装约束。

#### 1.`mas_makeConstraints`方法解析

用户是UIView调用扩展的`UIView+MASAdditions`分类的`mas_makeConstraints`方法来为当前视图添加约束的。
mas_makeConstraints方法的返回值是一个数组（NSArray）,数组中所存放的就是当前视图中所添加的所有约束。因为Masonry框架对NSLayoutConstraint封装成了MASViewConstraint，所有此处数组中存储的是MASViewConstraint对象。

接下来来看`mas_makeConstraints`的参数，`mas_makeConstraints`测参数是一个类型为`void(^)(MASConstraintMaker *)`的匿名block（也就是匿名闭包），该闭包的返回值为void, 并且需要一个`MASConstraintMaker`工厂类的一个对象。该闭包的作用就是可以让`mas_makeConstraints`方法通过该block给`MASConstraintMaker`工厂类对象中的`MAConstraint`属性进行初始化。
具体可以参考下面的代码及其注释:

```
//新建并添加约束
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
   //关闭自动添加约束，由我们手动添加约束
    self.translatesAutoresizingMaskIntoConstraints = NO;
    //实例化constraintMaker对象，来操作接下来的约束
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    //block作为参数，这里完成隐式调用，完成回调，通过block将constraintMaker对象回调给用户让用户对constraintMaker中的MAConstraint类型的属性进行初始化。换句话说block中所做的事情就是之前用户设置约束是所添加的代码，比如make.top(@10) == ( constraintMaker.top = 10 )。
    block(constraintMaker);
    //添加约束，但会Install的约束数组
    return [constraintMaker install];
}
```

#### 2. block参数的隐式回调
返回的值为一个block,block的返回值是MASConstraint类的实例对象，所以最终还是返回的MASConstraint类的实例对象。
```
- (MASConstraint * (^)(id))equalTo {
    return ^id(id attribute) {
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}
```

#### 3.约束安装install方法

实际的过程是：
> 1. 判断是否有约束，有就遍历约束，调用uninstall清空之前所有的约束
> 2. 无约束，就遍历数组的约束对象，然后调用install逐个安装
> 3. 调用系统的方法安装约束

```
- (NSArray *)install {
  //判断是否存在约束，存在就遍历所有约束，然后移除
    if (self.removeExisting) {
        NSArray *installedConstraints = [MASViewConstraint installedConstraintsForView:self.view];
        for (MASConstraint *constraint in installedConstraints) {
            [constraint uninstall];
        }
    }
     //不存在约束，就复制约束，然后遍历数组中的约束，完成安装。
    NSArray *constraints = self.constraints.copy;
    for (MASConstraint *constraint in constraints) {
        constraint.updateExisting = self.updateExisting;
        [constraint install];
    }
    [self.constraints removeAllObjects];
    return constraints;
}
```
## 文末：
以上是masonry。通过这个也是给大家渗透链式编程的思想。
可能很多人对block作为返回值比较难理解，但这是整个链式编程的核心。


