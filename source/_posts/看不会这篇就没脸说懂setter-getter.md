---
title: 看不会这篇就没脸说懂setter/getter
date: 2016-05-19 15:37:38
tags: setter/getter
---
## 背景
`setter/getter`是一个类最基本的东西，任何一门面向对象的语言，都有这个概念，C++、java等等。因为`setter/getter`是对面向对象语言封装的最基本的支持。

## OC中的setter/getter特点和变化
OC的`setter/getter`和其他面向对象的语言没有什么不同。只不过，添加了一些自己的特性。
## LLVM编译器下的`setter/getter`的实质。
我们都知道`@property`实质帮我们做了`setter`、`getter`和生成`_属性名`的三部操作，具体情况是怎样的呢，下面三部将给你一一揭示`@peroperty`的实质。

### `Step1:`成员变量

首先查找是否有以` _属性名 `命名的成员变量。

如果有，默认对其进行set和get；

如果没有，则隐式生成以` _属性名 `命名的成员变量；

实际LLVM编译器会隐形的帮我们创建一个`_属性名`的成员变量。

<不过注意，编译器会先检测有无相关成员变量，有不创建，无才创建，下文会有详细说明>

### `Step2:`系统默认实现`setter`方法
代码如下：

```
- (void)setName:(NSString *)name {
    _name = name;
}
```

### `Step3:`系统默认实现`getter`方法
代码如下：

```
- (NSString *)name {
    return _name;   //调用getter方法返回的是下划线的成员变量。
}
```

--------
## 更改@property的`_属性`
虽然系统默认帮助我们生成了_属性名的成员变量，假如我们并不想如此，而是把系统的_属性更改为制定的成员变量。接下来怎操作呢？
实质上是由两种方法的，下面一一介绍:

### 方式一：
重写`setter/getter`方法，将` _属性名 ` 更改为我们想要的名字；

### 方式二:
直接使用`@synthesize 属性名 = 指定属性名`；
两种创建方式的代码如下：

```
@interface ViewController () {
    
    NSString *mySynthesizeString2;   //synthesize方法
    NSString *setterString;          //setter的string
}
@property(nonatomic,copy) NSString *myString1;

@property(nonatomic,copy) NSString *myString2;  //@synthesize

@property(nonatomic,copy) NSString *myString3;

@end

@implementation ViewController

@synthesize myString1;          //没有_myString1

@synthesize myString2 = mySynthesizeString2;    //将成员变量变为mySynthesizeString2

- (void)viewDidLoad {
    [super viewDidLoad];   
}

/**
 *  通过重写setter方法改变成员变量的值
 *
 *  @param myString2 将自定义的成员变量setterString赋值给myString2，
 *  以后调setter实际获得的是setterString，  _myString2 已不存在
 */
- (void)setMyString2:(NSString *)myString2 {
    setterString = myString1;
}

- (NSString *)myString2 {
    return setterString;
}
```

## 关于`@property`的属性注意事项

> 使用属性注意事项：
> 
> 1、当属性名与成员变量名一样时，如果我们想保证成员变量有值，那么就需要在`.m`中加入`@synthesize` 变量名；
> 
> 2、当属性名与成员变量名一样时，如果我们对成员变量的值不强求，但我们又想打印所赋的值，这时在`.m`里可以使用(`_属性名`) 或者`self.属性名`；
> 
> 3、当定义一个属性时，会首先查找是否有以 _属性名 命名的成员变量，如果有，默认对其进行`setter/getter`，如果没有，则隐式生成以`_属性名`命名的成员变量；
> 
> 4、当我们使用属性时成员变量可以省略。
(当`.h`文件中的成员变量名不省略时 `.m`文件中的`@synthesize` 也不能省略！当成员变量名省略时`@synthesize`也可以省略）

## 扩展关于setter,getter,readonly,readwrite

### （1）设置访问方法的名字
默认的getter和setter器的名称是和变量名关联的，一定是setVirableName和virableName，比如上面的变量age，setter是setAge，getter是age。
可以通过设置@property中的setter和getter属性来修改setter和getter器的方法名。
`getter=getterName`
`setter=setterName`
 
举个例子：`@property (getter=show1,setter=show2:)int age;`//现在，它的getter和setter的方法名字就变了  
注意：如果你设置了readonly属性的话，那么你就不应该设置setter属性，要不然会给出一个编译器的警告。
### （2）设置只读或读写
下面两个属性很好理解，
`readwrite`：表示既有getter，也有setter
`readonly`：表示只有getter，没有setter
这两个属性是互相排斥的，只能存在一个。

