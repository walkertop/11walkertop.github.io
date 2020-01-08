---
title: KVC和runtime字典转模型对比分析
date: 2016-06-19 00:25:33
tags: KVC Runtime
---

>本文分为两部分：
>
一：教你怎样一部获取成员属性（通过NSObject+autoLogProperty分类）
二：对比KVC和runtime两种字典转模型的方法并抽取一个分类

# **一：自定义分类，打印字典转模型的属性声明**
```
+ (void)createPropertyCodeWithDict:(NSDictionary *)dict
{
    NSMutableString *strM = [NSMutableString string];
    
    // 遍历字典
    [dict enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull propertyName, id  _Nonnull value, BOOL * _Nonnull stop) {
        NSString *code;
        //__NSCFString  <——>   NSString类型
        //__NSCFNumber  <——>   int类型
        //__NSCFArray  <——>    NSArray类型
        //__NSCFDictionary  <——>   NSDictionary类型
        //__NSCFBoolean  <——>   BOOL类型

        if ([value isKindOfClass:NSClassFromString(@"__NSCFString")]) {
            code = [NSString stringWithFormat:@"@property (nonatomic, strong) NSString *%@;",propertyName];
        }else if ([value isKindOfClass:NSClassFromString(@"__NSCFNumber")]){
            code = [NSString stringWithFormat:@"@property (nonatomic, assign) int %@;",propertyName];
        }else if ([value isKindOfClass:NSClassFromString(@"__NSCFArray")]){
            code = [NSString stringWithFormat:@"@property (nonatomic, strong) NSArray *%@;",propertyName];
        }else if ([value isKindOfClass:NSClassFromString(@"__NSCFDictionary")]){
            code = [NSString stringWithFormat:@"@property (nonatomic, strong) NSDictionary *%@;",propertyName];
        }else if ([value isKindOfClass:NSClassFromString(@"__NSCFBoolean")]){
            code = [NSString stringWithFormat:@"@property (nonatomic, assign) BOOL %@;",propertyName];
        }
        [strM appendFormat:@"\n%@\n",code];
    }];
    NSLog(@"%@",strM);
}
```
--------
## 1.**核心思想：**
 
     1.遍历自定义类中的成员变量
     
     2.将遍历获取的成员变量定为value,复制给类中的ivar.
     
## 2.**runtime字典转模型与KVC赋值的区别:**
     KVC是调用`setValue: forKey: `的方法，将系统的成员变量作为value，自定义的属性为key
     如果自定义的属性找不到就必须要调用     `- (void)setValue:(id)value forUndefinedKey:(NSString *)key;`
     来处理报错。
     但是runtime的字典转模型是将自定义属性生成的下划线成员变量变为key.
     `setValuesForKeysWithDictionary:`就不会出现找不到key而报错的问题了。

--------
# **两种字典转模型的代码：**

## 1.KVC方式字典转模型
```
+ (StatusModel *)statusWithDict:(NSDictionary *)dict
{
    StatusModel *statusModel = [[self alloc] init];
    // KVC
    [statusModel setValuesForKeysWithDictionary:dict];
    return statusModel;
}

// 解决KVC报错
- (void)setValue:(id)value forUndefinedKey:(NSString *)key
{
//可以为空，表示不处理，也可以为做一些转换操作
//    if ([key isEqualToString:@"XXX"]) {
//        _### = [value integerValue];
//    }
}
```
-------
## 2.runtime字典转模型
```
+ (instancetype)modelWithDict:(NSDictionary *)dict {
    id objc = [[self alloc] init];
    unsigned int count = 0;
    Ivar *ivarList = class_copyIvarList(self, &count);
    for (int i = 0; i < count; i++) {
        //获取属性名
        Ivar ivar = ivarList[i];
        
        //获取成员名
        NSString *propertyName = [NSString stringWithUTF8String:ivar_getName(ivar)];
        
        //获取key
        NSString *key = [propertyName substringFromIndex:1];   //取出下划线_
        //获取字典中的value
        id value = dict[key];
        
        //获取成员属性类型
        NSString *propertyType = [NSString stringWithUTF8String:ivar_getTypeEncoding(ivar)];

        //此处为二级转换，如果里面的为字典类型，且属性类型为二级模型的名字，不为NSDictionary
        if ([value isKindOfClass:[NSDictionary class]] && ![propertyType containsString:@"NS"]) { // 需要字典转换成模型
            // 转换成哪个类型
            // 打印出来的值为  @"@\"User\"", 截取成User
            // 字符串截取
            NSRange range = [propertyType rangeOfString:@"\""];
            propertyType = [propertyType substringFromIndex:range.location + range.length];
            //此时变为 User\"";,借着截取掉后面的\""
            range = [propertyType rangeOfString:@"\""];
            propertyType = [propertyType substringToIndex:range.location];
            
            // 获取需要转换类的类对象
            Class modelClass =  NSClassFromString(propertyType);
            
            if (modelClass) {
                value =  [modelClass modelWithDict:value];
            }
        }
        if (value) {
            // KVC赋值:不能传空
            [objc setValue:value forKey:key];
        }
    }
    return objc;
}

```

## 3.**具体使用**
```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSString *filePath = [[NSBundle mainBundle] pathForResource:@"status.plist" ofType:nil];
    NSDictionary *dict = [NSDictionary dictionaryWithContentsOfFile:filePath];
    
    //log方法属性
    NSArray *dictArr = dict[@"statuses"];
    [[self class] createPropertyCodeWithDict:dictArr[0]];            //打印StatusModel的属性
    [[self class] createPropertyCodeWithDict:dictArr[0][@"user"]];   //打印UserModel的属性
    
    NSMutableArray *statuses = [NSMutableArray array];
    // 遍历字典数组
    for (NSDictionary *dict in dictArr) {
//        KVC字典转模型，调用setValueForKeyWithDictionary:方法
//        StatusModel *statusModel = [StatusModel statusWithDict:dict];
        
//       runtime字典转模型，调用分类方法
        StatusModel *statusModel = [StatusModel modelWithDict:dict];
        [statuses addObject:statusModel];
    }
    NSLog(@"%@",statuses);
    self.dataArray = statuses;
}

//懒加载dataArray
- (NSMutableArray *)dataArray
{
    if (!_dataArray) {
        _dataArray  = [NSMutableArray array];
    }
    return _dataArray;
}
```

[点击下载demo](https://github.com/walkertop/Runtime-)

注意：demo中的工具类可以抽取使用

