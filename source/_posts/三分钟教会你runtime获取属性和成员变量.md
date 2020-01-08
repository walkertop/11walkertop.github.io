---
title: 三分钟教会你runtime获取属性和成员变量
date: 2016-05-09 10:10:59
tags: runtime
---

## 目录
> - 成员变量和属性到底是什么？
> - 怎么通过runtime获取属性？
> - 怎么通过runtime获取成员变量？ 
> - 成员变量和属性的区别？
> - 实际应用场景是什么？




## 成员变量
### 1、成员变量的定义 
> Ivar: 实例变量类型，是一个指向`objc_ivar`结构体的指针
> `typedef struct objc_ivar *Ivar;`


### 2、相关函数
>// 获取所有成员变量
>`class_copyIvarList`
>// 获取成员变量名
>`ivar_getName`
>// 获取成员变量类型编码
>`ivar_getTypeEncoding`
>// 获取指定名称的成员变量
>`class_getInstanceVariable`
>// 获取某个对象成员变量的值
>`object_getIvar`
>// 设置某个对象成员变量的值
>`object_setIvar`

**说明：**
`property_getAttributes`函数返回`objc_property_attribute_t`结构体列表，`objc_property_attribute_t`结构体包含`name`和`value`，常用的属性如下：

属性类型  `name`值：T                                     `value：`变化
编码类型  `name`值：C(copy) &(strong) W(weak)空(assign) 等 `value：`无
非/原子性 `name`值：空(atomic) N(Nonatomic)                `value：`无
变量名称  `name`值：V  	                                  `value：`变化

使用`property_getAttributes`获得的描述是`property_copyAttributeList`能获取到的所有的`name`和`value`的总体描述，如 T@"NSDictionary",C,N,V_dict1

### 3、实例应用

```
<!--Person.h文件-->
@interface Person : NSObject
{
    NSString *address;
}
@property(nonatomic,strong)NSString *name;
@property(nonatomic,assign)NSInteger age;
```
--------

```
//遍历获取Person类所有的成员变量IvarList
- (void) getAllIvarList {
    unsigned int methodCount = 0;
    Ivar * ivars = class_copyIvarList([Person class], &methodCount);
    for (unsigned int i = 0; i < methodCount; i ++) {
        Ivar ivar = ivars[i];
        const char * name = ivar_getName(ivar);
        const char * type = ivar_getTypeEncoding(ivar);
        NSLog(@"Person拥有的成员变量的类型为%s，名字为 %s ",type, name);
    }
    free(ivars);
}

```

--------
```
<!--打印结果-->
2016-06-15 20:26:39.412 demo-Cocoa之method swizzle[17798:2565569] Person拥有的成员变量的类型为@"NSString"，名字为 address 
2016-06-15 20:26:39.413 demo-Cocoa之method swizzle[17798:2565569] Person拥有的成员变量的类型为@"NSString"，名字为 _name 
2016-06-15 20:26:39.413 demo-Cocoa之method swizzle[17798:2565569] Person拥有的成员变量的类型为q，名字为 _age 
```
## 属性
### 1、属性的定义
> `objc_property_t`：声明的属性的类型，是一个指向`objc_property`结构体的指针
> `typedef struct objc_property *objc_property_t;`


### 2、相关函数

> // 获取所有属性
> `class_copyPropertyList`
> 说明：使用`class_copyPropertyList`并不会获取无`@property`声明的成员变量
> // 获取属性名
> `property_getName`
> // 获取属性特性描述字符串
> `property_getAttributes`
> // 获取所有属性特性
> `property_copyAttributeList` 


### 3、实例应用
```
<!--Person.h文件-->
@interface Person : NSObject
{
    NSString *address;
}
@property(nonatomic,strong)NSString *name;
@property(nonatomic,assign)NSInteger age;
```
--------
```
//遍历获取所有属性Property
- (void) getAllProperty {
    unsigned int propertyCount = 0;
    objc_property_t *propertyList = class_copyPropertyList([Person class], &propertyCount);
    for (unsigned int i = 0; i < propertyCount; i++ ) {
        objc_property_t *thisProperty = propertyList[i];
        const char* propertyName = property_getName(*thisProperty);
        NSLog(@"Person拥有的属性为: '%s'", propertyName);
    }
}
```
--------
```
<!--打印结果-->
2016-06-15 20:25:19.653 demo-Cocoa之method swizzle[17778:2564081] Person拥有的属性为: 'name'
2016-06-15 20:25:19.653 demo-Cocoa之method swizzle[17778:2564081] Person拥有的属性为: 'age'
```
## 应用具体场景

### 1、Json到Model的转化

在开发中相信最常用的就是接口数据需要转化成Model了（当然如果你是直接从Dict取值的话。。。），很多开发者也都使用著名的第三方库如`JsonModel`、`Mantle`或`MJExtension`等，如果只用而不知其所以然，那真和“搬砖”没啥区别了，下面我们使用runtime去解析json来给Model赋值。

原理描述：用runtime提供的函数遍历Model自身所有属性，如果属性在json中有对应的值，则将其赋值。

核心方法：在NSObject的分类中添加方法：

```
- (instancetype)initWithDict:(NSDictionary *)dict {
 
    if (self = [self init]) {
        //(1)获取类的属性及属性对应的类型
        NSMutableArray * keys = [NSMutableArray array];
        NSMutableArray * attributes = [NSMutableArray array];
        /*
         * 例子
         * name = value3 attribute = T@"NSString",C,N,V_value3
         * name = value4 attribute = T^i,N,V_value4
         */
        unsigned int outCount;
        objc_property_t * properties = class_copyPropertyList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            objc_property_t property = properties[i];
            //通过property_getName函数获得属性的名字
            NSString * propertyName = [NSString stringWithCString:property_getName(property) encoding:NSUTF8StringEncoding];
            [keys addObject:propertyName];
            //通过property_getAttributes函数可以获得属性的名字和@encode编码
            NSString * propertyAttribute = [NSString stringWithCString:property_getAttributes(property) encoding:NSUTF8StringEncoding];
            [attributes addObject:propertyAttribute];
        }
        //立即释放properties指向的内存
        free(properties);
 
        //(2)根据类型给属性赋值
        for (NSString * key in keys) {
            if ([dict valueForKey:key] == nil) continue;
            [self setValue:[dict valueForKey:key] forKey:key];
        }
    }
    return self;
 
}

```
读者可以进一步思考：

如何识别基本数据类型的属性并处理
空（nil，null）值的处理
json中嵌套json（Dict或Array）的处理

尝试解决以上问题，你也能写出属于自己的功能完备的Json转Model库。

### 2、快速归档

有时候我们要对一些信息进行归档，如用户信息类UserInfo，这将需要重写`initWithCoder`和`encodeWithCoder`方法，并对每个属性进行`encode`和`decode`操作。那么问题来了：当属性只有几个的时候可以轻松写完，如果有几十个属性呢？那不得写到天荒地老.

原理描述：用runtime提供的函数遍历Model自身所有属性，并对属性进行`encode`和`decode`操作。

核心方法：在Model的基类中重写方法：


```
- (id)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super init]) {
        unsigned int outCount;
        Ivar * ivars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            Ivar ivar = ivars[i];
            NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
            [self setValue:[aDecoder decodeObjectForKey:key] forKey:key];
        }
    }
    return self;
}
```

``` 
- (void)encodeWithCoder:(NSCoder *)aCoder {
    unsigned int outCount;
    Ivar * ivars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar ivar = ivars[i];
        NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
        [aCoder encodeObject:[self valueForKey:key] forKey:key];
    }
}

```
### 3、访问私有变量

我们知道，OC中没有真正意义上的私有变量和方法，要让成员变量私有，要放在m文件中声明，不对外暴露。如果我们知道这个成员变量的名称，可以通过runtime获取成员变量，再通过`getIvar`来获取它的值。

方法：

```
Ivar ivar = class_getInstanceVariable([Model class], "_str1");
NSString * str1 = object_getIvar(model, ivar);

```






