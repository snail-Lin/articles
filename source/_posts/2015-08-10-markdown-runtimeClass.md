---
layout: post
title:  "Objective-C Runtime-类结构分析"
date:   2015-08-10
categories: Objective-C
excerpt: "Objective-C是一门动态语言，程序在运行时可以改变其结构，新的函数可以被引进，或者交换一个方法的实现，最常见的另一个动态语言是JavaScript"
tag:
- Objective-C 
- Runtime
comments: true
---


>Objective-C动态特性的实现主要依赖于Runtime库，Runtime库将C语言中的结构体封装为Objective-C中的对象，函数封装为对象方法，并且增加了一些额外的面向对象特性

## Objective-C对象结构
### 类的定义
Objective-C中绝大多数类的基类为NSObject，NSObject的定义如下
```objc
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```
其中有一个`Class`类型的isa属性，Class类型是在objc.h头文件中定义的
```objc
typedef struct objc_class *Class;
```
可以看到Class是一个结构体指针，指向`objc_class`结构体，所以直接去runtime.h头文件中查看结构体`objc_class`的定义
```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```
逐一分析结构体的每一个成员
>**name**:保存Objective-C类的类名  
>**isa**:在Objective-C中，类本身也是一个对象，对象的isa指针指向`metaClass`  
>**super_class**:字面意思即父类，如果是根类，`super_class`将指向NULL  
>**version**:类的版本信息，默认为0
>**info**:类信息，供运行期使用的一些标识位  
>**instance_size**:该类的实例变量大小   
>**ivars**:objc_ivar_list结构体，保存类的成员变量链表.   
>**methodLists**:指向类中定义的方法列表.   
>**cache**:缓存最近使用的方法，如果每次调用方法都需要遍历一次. methodLists，会很消耗性能，每次调用的方法地址将会被缓存在cache中    
>**protocols**:保存当前类遵循的协议列表    
  

### 类的实例结构
在runtime.h有这么一句注释*instance of a class*，即类的实例，是一个结构体类型
```objc
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```
在下边也找到了id类型的定义,可以看出id是一个指向`objc_object`结构体的指针
```objc
typedef struct objc_object *id;
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```
这个结构体中只有一个成员，即指向其`objc_class`的isa指针，向一个Objective-C对象发送消息时，会根据isa指针找到实例对象所属的类的结构体，先遍历objc_class的`cache`再遍历`methodLists`查询是否有对应的方法,如果没有找到，根据`super_class`成员找到父类的结构体，依次循环查找，直至找到对应的方法或者objc_class的`super_class`指向NULL后停止查找。  

### 元类
在Objective-C中可以向一个类发送消息，那么意味着类本身也是一个对象，也包含一个指向类的isa指针，而且isa指针也必须指向一个包含这些类方法的一个objc_class结构体，类的isa指针指向的类便称为`meta-class`  ,meta-class中存储着一个类的所有类方法，每个类都只有一个meta-class
meta-class中的isa指向基类的meta-class,而基类的isa指针指向自己，形成一个**闭环**。 

### 实例变量结构
在objc_class中ivars指针指向属性列表，是一个objc_ivar_list结构体，单个实例变量的类型为Ivar，定义如下
```objc
typedef struct objc_ivar *Ivar;
struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
}   
```
>**name**:变量名称
>**type**:变量类型
>**ivar_offse**:表示该变量距离存放对象的内存区域的起始地址有多远
使用@property增加一个属性时，都会在objc_ivar_list中添加一个成员变量的描述，即一个objc_ivar结构体，然后在methodLists中添加一个setter和getter方法，计算偏移量，setter从偏移量的位置开始赋值。

### 方法
#### SEL
在Objective-C中我们通常使用`@selector()`调用方法,@selector获取的结果实际上是一个SEL类型，也是一个结构体
```objc
typedef struct objc_selector *SEL;
typedef const struct objc_selector   
{  
    void *sel_id;  
    const char *sel_types;  
} *SEL;  
```
在Objective-C中不能存在2个同名的方法名，即使参数类型不同也不行，这就导致Objective-C不能像其他语言一样进行方法的重载。SEL中保存了根据方法名hash化的一个key值，能唯一代表一个方法，系统将根据SEL找到方法的`IMP`指针,SEL的存在只是为了加快方法的查询速度。

#### IMP指针
IMP是一个函数指针，指向方法实现的首地址，定义如下
```objc
id (*IMP)(id, SEL, ...)
```
第一个参数为指向self的指针，如果是实例方法，则是类实例的内存地址  
第二个参数为方法选择器SEL  

#### Method
Method用于定义类的方法
```objc
typedef struct objc_method *Method;
 
struct objc_method {
    SEL method_name                 OBJC2_UNAVAILABLE;  // 方法名
    char *method_types                  OBJC2_UNAVAILABLE;
    IMP method_imp                      OBJC2_UNAVAILABLE;  // 方法实现
}
```
可以看到SEL和IMP在此结构体中建立了映射关系，有SEL就可以在`methodLists`找到对应的IMP。

### Protocol
Protocol的定义如下
```objc
typedef struct objc_object Protocol;
```
由此可知protocol其实就是一个对象结构体`objc_object`

### Category
Category是一个指向分类的结构体指针，定义如下
```objc
typedef struct objc_category *Category;
 
struct objc_category {
    char *category_name                          OBJC2_UNAVAILABLE; 
    char *class_name                             OBJC2_UNAVAILABLE;
    struct objc_method_list *instance_methods    OBJC2_UNAVAILABLE;
    struct objc_method_list *class_methods       OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols         OBJC2_UNAVAILABLE;
}
```
>*category_name*:即分类名  
>*class_name*:分类所属的类名  
>*instance_methods*:和object_class中的methodLists一样，保存方法列表  
>*class_methods*:保存类方法列表  
>*protocols*:分类实现的协议列表  

**Objective-C中类，对象，协议等都是由C语言中的结构体封装而来，方法由C函数封装，是C语言的一个超集,和C++不同的地方在于，Objective-C只能单继承，而C++支持多重继承。**


