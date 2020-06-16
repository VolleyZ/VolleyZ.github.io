---
layout:     post
title:      底层知识点
subtitle:   mj底层知识记录
date:       2020-06-16
author:     Volley
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - 底层原理
---

##一些底层相关面试题

* [OC对象本质](#object-essence)
* [KVO](#kvo)
* [Category分析](#category)

####objc源码：[https://opensource.apple.com/source/objc4/](https://opensource.apple.com/source/objc4/)

**了解底层首先要知道oc文件转c++命令：xcrun -sdk -iphoneos clang  -arch arm64 -rewrite-objc xxx.m**

##<a name="object-essence"></a>OC对象本质

通过点击Class进入内部，我们可以发现Class对象是一个指向`objc_class`结构体的指针。

我们首先看一下`objc_class`结构体：

```
struct objc_class {
    Class isa;  
    Class super_class;
    cache_t cache;             //方法缓存
    class_data_bits_t bits;//用于获取具体的类信息class_rw_t *data = bits & FAST_DATA_MASK
}
```

> 注：`class_rw_t`是个保存类信息的结构体,如下

```
struct class_rw_t {
    uint32_t flags;
    uint16_t version;
    uint16_t witness;

    const class_ro_t *ro;   //存储编译时当前类已经确定的属性、方法、协议的信息

    method_array_t methods;      //方法列表
    property_array_t properties; //属性列表
    protocol_array_t protocols;  //协议列表

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
}
```

> 注：`class_ro_t`是个保存编译时类已确定的信息的结构体,如下

```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize; //instance对象占用的内存空间
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;  //类名
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;  //成员变量列表

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
}
```

##### *1. 一个NSObject对象占用多少内存？*
> 系统分配了16个字节给NSObject对象（通过malloc_size函数获得）     
> 但NSObject对象内部只使用了8个字节的空间（64bit环境下，可以通过class_getInstanceSize函数获得）

    
##### *2. 对象的isa指针指向哪里？*
> instance对象的isa指向class对象。  
> class对象的isa指向meta-class对象。    
> meta-class对象的isa指向基类的meta-class对象
    
    
##### *3. OC的类信息存放在哪里？*
> 对象方法、属性、成员变量、协议信息，存放在class对象中     
> 类方法，存放在meta-class对象中  
    (因为这些编译器只需编译一次就可以知道类里面有什么东西了)   
    ----------------------------------  
> 成员变量的具体值，存放在instance对象里

    
##### *4. class对象superclass的指针作用，meta-class对象的superclass指针的作用*
```
@interface Person: NSObject
- (void)run;
+ (void)personClassMethod;
@end
Class Student: Person
@end

Student *student = [[Student alloc] init];
[student run];
[Student personClassMethod];
```
>
> 调用方法其实就是给对象发消息
>
**class对象superclass指针:**
实例方法是存在类对象里的，首先先通过student实例对象的isa找到student自己的类对象，然后通过Sutdent类对象的superclass指针找到Person类对象,然后就调起Person类对象里面的run方法
>
**meta-class:**
首先类方法时存放在元类对象里的，当Student的class要调用Person的类方法时，首先通过Student的class的isa找到Student的meta-class，然后通过superclass找到Person的meta-class，最后找到类方法实现进行调用

##### *5. 简述OC方法的调用过程*
> oc是动态语言，在运行时方法会被动态转为消息发送，即：objc_msgSend(receiver, selector)。在方法调用的时候，其实就是给某个对象发送消息。   
1.runtime会根据对象的isa找到对象所属的类，然后在类的方法列表里寻找，如果找到相符的方法的话，就调用这个方法的实现。     
2.如果在当前所属类找不到的话，就通过superclass指针向父类的方法列表中寻找方法并调用。     
3.如果在最顶层的父类（一般是NSObject）中找不到的话，程序会抛出异常unrecognized selector sent to xxx，

## <a name="kvo"></a>KVO
##### *1. iOS用什么方式实现一个对象的KVO？（KVO本质、原理是什么？）*
* 利用runtime的API（`objc_allocateClassPair`和`objc_registerClassPair`）动态生成一个子类，并且让instance的isa指向这个全新的子类（NSKVONotifying_XXX）
* 当修改instance对象的属性时，会调用Foundation的`_NSSetXXXValueAndNotify`函数

> 而`_NSSetXXXValueAndNotify`函数里会调用:

```
1. willChangeValueForKey:       
2. 父类原来的setter方法        
3. didChangeValueForKey:   
内部会触发监听器（Observer）的监听方法（observeValueForKeyPath: ofObject:(id)object change: context:）
```
KVO本质其实就是重写了set方法。  
KVO的原理其实就是把addObserver的instance的isa指向了NSKVONotifying_XXX的类，而这个类里面重写了set方法的实现。

##### *2. 如何手动触发KVO？*
> 手动调用`willChangeValueForKey:`和`didChangeValueForKey:`

##### *3. 直接修改成员变量会触发KVO吗？*
> 不会。因为只接修改成员变量（_age = 2）不会触发set方法。


## <a name="category"></a>Category分析

##### *1. Category的加载处理过程。*
> 1.通过Runtime加载某个类的所有分类数据。  
> 2.把所有Category的方法、熟悉、协议数据，合并到一个大数组中，后面参与编译的Category数据，会在数组的前面。     
> 3.将合并后的分类数据（方法、熟悉、协议）插入到类原来数据的前面。     
> 注：也就是说调用方法时默认会找到最新添加的Category的方法调用。（可以通过xcode的compile sources修改编译顺序）

##### *2.分类的实现原理。*
> Category在编译后会生成一个`category_t`对象，有多个分类的时候就会有多个`category_t`类型的对象，在编译阶段分类里面的东西跟类的东西是分开的，在运行的时候，分类的东西会合并到类(如：对象方法等)或者元类（如：类方法）里面去。

分类编译成`struct category_t`，里面存储着分类的对象方法，类方法，属性，协议信息。结构如下：

```
struct _category_t {
	const char *name;                              //分类名称，如：Person
	struct _class_t *cls;                          // 0
	const struct _method_list_t *instance_methods; //对象方法列表
	const struct _method_list_t *class_methods;    //类方法列表
	const struct _protocol_list_t *protocols;      //协议列表
	const struct _prop_list_t *properties;         //属性列表
};
```
