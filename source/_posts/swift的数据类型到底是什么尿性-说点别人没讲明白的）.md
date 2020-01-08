---
title: swift的数据类型到底是什么尿性 (说点别人没讲明白的）
date: 2017-08-22 21:03:09
tags: swift
---

## 文初：
如果你对swift的些许了解只局限在

* swift中的类型使用struct取代class
* 多了Optional可选类型

这些最基础的认知，而对其底层设计的原因和原理了解甚少，那这篇文章会给你新的视角，让你更好的理解和使用。
为了让你能更宏观的理解swift在类型设计方面的初衷，本文会也会从更加宏观的角度着手。

其实swift的类型，从大的层面有两种类型：

* Named Types
* Compound Types

## Named Types 

其具体包含如下：

```
   * class 
   * struct 
        {
            * String
            * Int
            * Float
            * Bool
            * ...
        }
    * enum
    * protocol
```


### swift类型的新特点（区别与OC）：
    
```

* 没有特定的根类型（No dedicated type root ）
* 类型通过遵守协议的方式，而非继承来实现扩展（ Type conforms to protocols instead of heavy inheritance ）
* 低耦合（Super loose coupling ）
* 概念分离（Nice separation of concerns ）
* 结构清晰（Clean architecture ）
```
对比OC，所有的对象类型都继承自NSObject，其本质是Class类型。
为了保证继承的完整性，又多出了meta class等概念。Class中通过继承来实现的方法的扩展。

比如：NSArray及NSMutableArray类型
NSMutableArray是NSArray的子类。子类NSMutableArray特有的`addObject:`等方法是通过继承NSArray,在子类中单独实现。

![OC通过继承扩展方法](http://oiu13lwmh.bkt.clouddn.com/15033951098490.jpg)

这种设计本身无可挑剔，但OC过于依赖继承来实现方法的扩展，导致整个体系特别臃肿，过于耦合。



```
@interface AClass: NSObject

- (void)aMethod;
- (void)bMethod;

- (void)oneMethod;
- (void)twoMethod;

@end

//BClass想拥有aMethod和bMethod方法，不需要oneMethod和twoMethod,在OC一般都继承实现
@interface BClass: AClass

@end
```

而swift不过度依赖Class，其扩展了Struct，Enum, Protocol，并给其更高的权限（可更加灵活的添加属性，定义方法等），让其变成一等公民。

swift底层是结构体，结构体不是对象类型，自然没办法通过继承来实现方法扩展。那其方法实现是通过何种方式完成的呢？
答案是协议，不同的协议包裹了不同的方法，通过遵守不同的协议来扩展特定的方法。

对于上述需求，在swift中一般通过如下方法实现

```
// swift通过protocol来扩充方法，通过遵守不同的协议来获取不同方法，更偏重组合，而非继承
protocol FirstProtocol {
    func aMethod()
    func bMethod()
}

protocol SecondProtocol {
    func oneMethod()
    func twoMethod()
}

class AClass: FirstProtocol,SecondProtocol {
    //实现两个方法
}

class BClass: FirstProtocol {
    //实现两个方法
}
```

比如Array类型，其通过继承不同的协议来获取不同的方法。
![Array遵守的协议](http://oiu13lwmh.bkt.clouddn.com/15033835889832.jpg)





```
* 类比理解：

继承更讲究出身，尤其OC是单继承，只能有一个父类，你想获取父类制定的方法，必须继承父类。
而面向协议则更加灵活，不同的protocol封装了不同的技能包，某个类型想要获取对应方法，只需遵守该protocol就ok了。而且一个类型可以遵守多个协议，更加灵活。而且Enum,Struct都可以通过遵守协议来获取方法。
```

--------

## Compound Types (复合类型）
* Tuples（元组）
* Function（函数）

### Tuples Types
Tuples（元组）是swift的新类型。其成员不同，Tuple类型就不同。

```
var aTuple = (x: 10, y: 20)
// 此时就会报错，x的类型只能是Int
// aTuple = (x: "name", y: 50) 
```

当Tuples作为函数的返回值类型，就可以返回多个值

```
let nameDictionary: [Int: String] = [001: "Jimmy"]
let ageDictionary: [Int: Int] = [001: 20]

func getInformationFrom (ID: Int) -> (name: String, age: Int) {
    let name = nameDictionary[ID] ?? "找不到对应的名字"
    let age = ageDictionary[ID] ?? 0
    
    return (name, age)
}
```


### Function Types


swift将Function（函数）提升为一等类型

```
let nameDictionary: [Int: String] = [001: "Jimmy"]
let ageDictionary: [Int: Int] = [001: 20]

func getInformationFrom (ID: Int) -> (name: String, age: Int) {
    let name = nameDictionary[ID] ?? "找不到对应的名字"
    let age = ageDictionary[ID] ?? 0
    
    return (name, age)
}
// 用aPerson接受这个函数，类型为一个元组类型,值为(name: "Jimmy", age: 20)
let aPerson = getInformationFrom(ID: 001)
```

### Blocks VS Closures（闭包）
swift中的closure类似于OC的block，但还是有些区别。

![Blocks VS Closures](http://oiu13lwmh.bkt.clouddn.com/Blocks%20VS%20Closures.png)

##### 关于两者语法的不同，可以参考
[Block Syntax](http://fuckingblocksyntax.com/)  and [Closure Syntax](http://fuckingclosuresyntax.com)


### Function VS Closures
A closure is actually a higher usage of a function type. Above, we just have two types of types; the closure is defined quite similarly to the function itself. This is because it takes the parameters and the area and the return type just as the type signage. This makes functions and closures siblings.


## Optional可选类型

swift中的Optional底层就是一个带泛型的枚举类型。

其源码抽象出来如下：

```
public enum Optional<Wrapped> : ExpressibleByNilLiteral {
    case none
    case some(Wrapped)
    
    public init(_ some: Wrapped)
    public init(nilLiteral: ())
}
```


```
// 此时定义中的Wrapped就是String类型
var myName : String? = "Walker"
	print(myName)
	print(myName!)

	if let name = myName {
		print(name)
	}
```

