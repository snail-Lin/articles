---
layout: post
title:  "Objective-C中的消息转发机制"
date:   2015-10-05
categories: Objective-C
excerpt: "在Objective-C中，调用哪个方法都是在运行期决定的，编译器在编译时无法确认类中是否有某个方法的实现，当对象接收到无法解读的消息后，就会启用“消息转发”机制"
tag:
- Objective-C 
- 消息转发
comments: true
---

 接触过Objective-C的人都曾经遇到过`unrecognized selector sent to`错误,意思是向对象发送了一个无法识别的消息，从而导致了程序崩溃。在崩溃之前，这条消息经过了一条**完整的消息转发通道**让接收者处理当前接收到的消息。

## 消息转发
如果对象接收到无法处理的方法，即遍历`objc_class`的`objc_method_list`后无法找到对应的方法的SEL后，系统将转发这个消息，分为三个步骤，第一个步骤询问类是否能够处理这个消息，在这一阶段处理消息的一般做法就是动态的为类或类实例添加一个方法，称为`动态方法解析`。第二个步骤让处理消息的类寻找是否有其它的对象能够处理。第三个步骤系统将会封装消息的所有细节到NSInvocation对象中，让接收者寻找其它的方法处理这个消息，例如为消息添加一些参数，改变SEL，或者转发给其它对象。
### 动态方法解析
在对象接收到无法处理的实例方法后，首先将调用所属类的一个类方法
```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel
```
接收到无法处理的类方法，则调用
```objc
+ (BOOL)resolveClassMethod:(SEL)sel
```
此方法的参数即为无法处理的选择子，返回一个BOOL类型，表示是否可以增加一个方法用于处理此sel。在此方法中可以为类动态添加一个函数以处理这个未知的选择子。通常使用此方法的情景是为了实现一个dynamic属性，不使用关联对象也可以动态的为类添加一个属性。
```objc
id autoGetter(id self, SEL _cmd) {
    //setter方法实现
}
void autoSetter(id self, SEL _cmd, id value) {
    //编写Setter方法实现
}
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSString *selString = NSStringFromSelector(sel);
    if ([selString containsString:@"dynamicName"]) {
        /*
         i -> 返回值类型int，若是v则表示void
         @ -> 参数id(self)
         : -> SEL(_cmd)
         */
        if ([selString hasPrefix:@"set"]) {
            class_addMethod(self, sel, (IMP)autoSetter, "v@:@");
        }else {
            class_addMethod(self, sel, (IMP)autoSetter, "@@:");
        }
    }
    return [super resolveInstanceMethod:sel];
}
```

### 询问是否有其它接收者能够处理
如果经过`resolveInstanceMethod`对象还是无法处理该消息，则调用
```objc
- (id)forwardingTargetForSelector:(SEL)aSelector
```
询问是否有其它对象能够处理，通过此方法可以模拟`多重继承`的某些特性，在对象内部，还有一系列其它对象，NSString便是一个`类簇`，NSString会根据方法的不同，选择不同的内部对象来处理方法并返回，在外界看来，像是该对象本身处理并执行了此方法。

### 消息封装
如果经过前两个消息转发依然不能处理消息的话，Objective-C会将消息的所有细节封装于NSInvocation对象，包含SEL，target，以及方法参数。调用以下方法来转发消息内容
```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation
```
在此方法中，我们可以改变调用的target，让消息在另一个对象上进行调用，那么就和`forwardingTargetForSelector`一样，仅仅改变了接收者。在此方法中，更多的做法是改变消息的内容，比如追加参数或者改变选择子等等，或者在需要某方法需要使用超类调用的时候，可以将`anInvocation`转交给超类处理。  
在此方法调用之前，我们必须覆写`methodSignatureForSelector`方法，可以使用函数类型描述来创建一个`NSMethodSignature`方法签名对象，否则系统将不会调用最后一步的`forwardInvocation`方法将消息的所有细节进行封装。
```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [NSMethodSignature signatureWithObjCTypes:"v@:"];
}
```
如果覆写了`forwardInvocation`但是没有处理消息，则最终消息的信息将会被抛弃，系统不会抛出异常。  
如果没有覆写`forwardInvocation`方法，则无法处理消息的对象将会调用`doesNotRecognizeSelector:(SEL)aSelector`抛出异常。
所以最终的消息转发流程是
![](/assets/img/overloading/forwadSelector.png)


