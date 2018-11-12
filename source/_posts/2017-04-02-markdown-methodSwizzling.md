---
layout: post
title:  "使用Method Swizzling出现的一些问题"
date:   2017-03-31
categories: Objective-C
excerpt: "Objective-C是一门动态语言，在运行时可以替换方法的实现，这是一把'双刃剑'"
tag:
- Method Swizzling 
- Objective-C
comments: true
---

## 前言
最近门店反馈了*海底捞APP*在使用时会偶然出现一些重复下单的问题，经排查是因为同事没有加上校验码的，按钮被连续点击之后会连发两次报文，因为没有校验码，导致后端无法判断是否是同一次订单而偶然出现下单重复的原因。  
按钮被暴力点击，通常使用遮罩视图或者在点击之后关闭按钮响应来解决。但是在一些旧设备上，遮罩视图出现的比较慢，而且即使在第一次点击之后设置`btn.enable = false`，也会出现按钮被连续点击响应多次事件情况。就连微信也会偶尔出现同一个界面被push两次的问题。  

经过考虑，代码改动量最小的情况下是添加一个`Category`，使用`Method Swizzling`为所有按钮添加响应锁定方法。

## Method Swizzling
`Method Swizzling`的原理很简单，只要交换两个`SEL`对应的`IMP`指针即可，使用`method_exchangeImplementations`即可实现交换。这里为UIButton添加一个类别，实现逻辑是在按钮第一次点击之后设置一个标志位代表按钮已经被点击，不会影响第一次的响应速度，在一段时间后再将标志位置为可点状态，代码如下：
```objc
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        SEL originalS = @selector(sendAction:to:forEvent:);
        SEL swizzledS = @selector(safetySendAction:to:forEvent:);
        
        Method originalMethod = class_getInstanceMethod(class, originalS);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledS);
        
        method_exchangeImplementations(originalMethod, swizzledMethod);
    });
}
- (void)setClickedTime:(NSTimeInterval)clickedTime {
    objc_setAssociatedObject(self, @selector(clickedTime), @(clickedTime), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (NSTimeInterval)clickedTime {
    return [objc_getAssociatedObject(self, _cmd) doubleValue];
}

- (void)setIsClicked:(BOOL)isClicked {
    objc_setAssociatedObject(self, @selector(isClicked), @(isClicked), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (BOOL)isClicked {
    return [objc_getAssociatedObject(self, _cmd) boolValue];
}

- (void)safetySendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
    if (self.clickedTime == -1) {
        [self safetySendAction:action to:target forEvent:event];
        return;
    }
    if (self.isClicked) {
        return;
    }
    self.isClicked = YES;
    NSInteger sec = self.clickedTime == 0 ? 0.7 * 1000 : self.clickedTime * 1000;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, sec * NSEC_PER_MSEC), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        self.isClicked = NO;
    });
    [self safetySendAction:action to:target forEvent:event];
}
```
头文件中添加一个`clickedTime`属性，用于设置按钮锁定时间
```objc
@interface UIButton (ClickSafety)
@property (nonatomic, assign) NSTimeInterval clickedTime;
@end
```
在`Category`中添加属性无法自动生成`setter`和`getter`方法，这里使用了`runtime`中的`objc_setAssociatedObject`函数，为UIButton动态的添加一个属性,因为`objc_setAssociatedObject`无法绑定非`Object`的类型，所以其中做了一个转换，虽然clickedTime属性为`NSTimeInterval`，但实际上还是用`NSNumber`保存的属性值，`setter`和`getter`方法中进行了类型转换。

## 出现的问题
### UIDatePicker滑动后崩溃
为`UIButton`添加的`Category`会导致其它类出问题，这个是没意料到情况，经过排除之后找出了原因。 
`Class`的结构是
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
方法调用首先会查询`cache`中是否有对应的方法,若没有则查询`methodlists`中保存的方法列表，若还是查询不到对应的方法，则根据`super_class`指针查询父类中的方法，依次遍历下去查询直至根类。  
**sendAction:to:forEvent:是在UIControl类中定义的**,所以对应的`Method`是保存在父类即`UIControl`的`methodlists`下。而`Method`的结构体是这样的:
```objc
typedef struct objc_method *Method;
 
struct objc_method {
    SEL method_name                 OBJC2_UNAVAILABLE;  // 方法名
    char *method_types                  OBJC2_UNAVAILABLE;
    IMP method_imp                      OBJC2_UNAVAILABLE;  // 方法实现
}
```
可以看到SEL和IMP在此结构体中建立了映射关系，`method_exchangeImplementations`交换的是两个选择子对应的`IMP`指针。  
在上面的代码中，即使我们表面上替换的只是`UIButton`的`sendAction:to:forEvent:`方法，但由于`sendAction:to:forEvent:`的`SEL`被定义在`UIControl`的`methodLists`指向的其中的一个`objc_method`结构体。导致了所有继承于`UIControl`的子类都同一个方法都被指向了自定义方法`safetySendAction:to:forEvent:`的实现，由于其它子类没有动态的添加`safetySendAction:to:forEvent:`这个方法，所以出现了`unrecognized selector sent to`错误，从而导致了程序崩溃。

### 多个target情况下无法响应
如果一个按钮使用`addTarget`添加多个target对象和多个响应事件，在按钮点击之后首先会调用响应`sendAction:to:forEvent:`方法将事件分发到第一个添加的target对象，然后继续调用`sendAction:to:forEvent:`通知所有的target。  
因为在代码里添加了标志位判断导致第二个target接收不到响应信息而出现了一些Bug，所以多个target的情况也必须进行处理，由于这种情况很少见，所以给了一个简单的处理,直接略过即可：
```objc
    if (self.allTargets.count > 0) {
        [self safetySendAction:action to:target forEvent:event];
        return;
    }
```

经过完整的测试，这次代码再没有出现任何其它的问题。

## 最终代码
```objc
#import <objc/runtime.h>
@implementation UIControl (ClickSafety)
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        SEL originalS = @selector(sendAction:to:forEvent:);
        SEL swizzledS = @selector(safetySendAction:to:forEvent:);
    
        Method originalMethod = class_getInstanceMethod(class, originalS);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledS);
    
        method_exchangeImplementations(originalMethod, swizzledMethod);
    });
}
- (void)setClickedTime:(NSTimeInterval)clickedTime {
    objc_setAssociatedObject(self, @selector(clickedTime), @(clickedTime), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
- (NSTimeInterval)clickedTime {
    return [objc_getAssociatedObject(self, _cmd) doubleValue];
}
- (void)setIsClicked:(BOOL)isClicked {
    objc_setAssociatedObject(self, @selector(isClicked), @(isClicked), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
- (BOOL)isClicked {
    return [objc_getAssociatedObject(self, _cmd) boolValue];
}
- (void)safetySendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
    if (self.allTargets.count > 0) {
        [self safetySendAction:action to:target forEvent:event];
        return;
    }
    
    if (self.clickedTime == -1) {
        [self safetySendAction:action to:target forEvent:event];
        return;
    }
    
    if (self.isClicked) {
        return;
    }
    self.isClicked = YES;
    NSInteger sec = self.clickedTime == 0 ? 0.5 * 1000 : self.clickedTime * 1000;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, sec * NSEC_PER_MSEC), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        self.isClicked = NO;
    });
    [self safetySendAction:action to:target forEvent:event];
}
@end
```

## 总结
使用`Method Swizzling`可以在不改动原代码的情况下动态的替换一些方法，这固然是一个简便的方法，但也容易出现一些未知的问题。使用`Method Swizzling`要遵循两个原则：
*  替换的方法尽量要在`原方法被定义的类中`中实现，而不是在子类中实现
*  在替换方法中调用`原方法`的实现

Objective-C有很多的动态特性，`Method Swizzling`也只是其中之一，用好Objective-C的动态特性，将会获益良多。


