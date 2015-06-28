---
date: 2015-06-29 01:13:31
layout: post
title: NSJSONSerialization中各参数说明
category: ios
tags: ios
keywords: ios NSJSONSerialization NSJSONReadingOptions NSJSONWritingOptions NSJSONReadingMutableLeaves
description:
---

##用法

####jsonString to object

```objectivec
NSString *jsonString = @"{\"test\":\"abc\"}"
NSData* data = [self dataUsingEncoding:NSUTF8StringEncoding];
NSError* error = nil;
id result = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingMutableContainers error:&error];
NSLog(@"result %@", result);
```

####object to jsonString

```objectivec
NSDictionary *dict = @{@"test":@"abc"};
NSError *error = nil;
NSData *data = [NSJSONSerialization dataWithJSONObject:dict options:NSJSONWritingPrettyPrinted error:&error];
NSString *jsonString = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
NSLog(@"jsonString %@", jsonString);
```

##NSJSONReadingOptions说明

####先看定义

```objectivec
typedef NS_OPTIONS(NSUInteger, NSJSONReadingOptions) {
    NSJSONReadingMutableContainers = (1UL << 0),
    NSJSONReadingMutableLeaves = (1UL << 1),
    NSJSONReadingAllowFragments = (1UL << 2)
} NS_ENUM_AVAILABLE(10_7, 5_0);
```

####再看官方文档的说明

`NSJSONReadingMutableContainers`<br>
Specifies that arrays and dictionaries are created as mutable objects.

`NSJSONReadingMutableLeaves`<br>
Specifies that leaf strings in the JSON object graph are created as instances of NSMutableString.

`NSJSONReadingAllowFragments`<br>
Specifies that the parser should allow top-level objects that are not an instance of NSArray or NSDictionary.


####解释一下

`NSJSONReadingMutableContainers`<br>
意思是返回的数组和字典都是**NSMutableArray**和**NSMutableDictionay**类型，固可以作后续修改。

`NSJSONReadingMutableLeaves`<br>
意思是返回的的字符串是**NSMutableSting**类型，这里实践后是不行的，网上查了很多资料说是苹果的**bug**，到目前ios8中也没有修复。

`NSJSONReadingAllowFragments`<br>
意思是需要格式化的json字符串最外层可以不是数组和字典，只要是正确的json格式就行。

####总结

如果需要后续进行修改，则需要使用`NSJSONReadingMutableContainers`，返回可变类型，其他的选项中的数字和字典都是不可变的。

如果需要兼容所有的json格式，则使用`NSJSONReadingAllowFragments`，因为使用`NSJSONReadingMutableContainers`和`NSJSONReadingMutableLeaves`转换最外层不是字典或数组的字符串会报错，导致转换失败，即使是正确的json字符串（如`"test"`,`123`,`false`等）。

至于`NSJSONReadingMutableLeaves`，在bug修复前没有使用的必要了，既不能返回可变类型，也不能兼容所有的json格式。

PS：是否正确的json字符串，苹果是这样定义的

The top level object is an NSArray or NSDictionary.<br>
All objects are instances of NSString, NSNumber, NSArray, NSDictionary, or NSNull.<br>
All dictionary keys are instances of NSString.<br>
Numbers are not NaN or infinity.

最外层必须是数组或者字典（可以看出`NSJSONReadingAllowFragments`是个例外）<br>
所有的对象必须是`NSString`、`NSNumber`、`NSArray`、`NSDictionary`、`NSNull`的实例<br>
所有字典的键值key必须是NSString类型<br>
数字对象不能是非数值或无穷

---

##NSJSONReadingOptions说明

####先看定义

```objectivec
typedef NS_OPTIONS(NSUInteger, NSJSONWritingOptions) {
    NSJSONWritingPrettyPrinted = (1UL << 0)
} NS_ENUM_AVAILABLE(10_7, 5_0);
```

####再看官方文档的说明

`NSJSONWritingPrettyPrinted`<br>
Specifies that the JSON data should be generated with whitespace designed to make the output more readable. If this option is not set, the most compact possible JSON representation is generated.

意思就是生成的json字符串是带换行和缩进的，更加易读。如果要生成单行不带缩进的就直接设0。<br>
（PS:这里有点小坑，直接多写个枚举`NSJSONWritingSingleLine`多好，简单易读，If this option is not set这里说的选项不设置，害我猜了半天什么意思，还以为有另外一个方法是不需要穿参数的，结果没有...）