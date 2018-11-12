---
layout: post
title:  "关于Swift中的异常处理"
date:   2017-03-05
categories: Swift
excerpt: "Swift中可以将方法标记为throws，用来表示方法可能抛出异常，编译器会验证调用对象是否捕获异常"
tag:
- Swift 
- throws
- 异常处理
comments: true
---

**本文中使用的Swift版本为3.0**

# 抛出异常
Swift中可以将方法标记为`throws`，写法是在函数或者方法的返回箭头前加上`throws`关键字，用来表示方法可能抛出异常，编译器会验证调用对象是否捕获异常
```swift    
fun thisIsThrowFunction() throws -> Void
```
如果我们需要将一个`Foundation object`转换成一个`Generate JSON data`，你可能会写以下代码:
```swift
let dic: [String: String] = ["key": "value"]
let data = JSONSerialization.data(withJSONObject: dic, options: JSONSerialization.WritingOptions.prettyPrinted)
```    
你会发现编译器将会报错 `Call can throw`的错，这是因为这个方法被标记为`throws`，必须处理方法抛出的异常，处理异常需要使用`try`关键字，这在Swift 1.0版本中是没有的，在Swift 2.0中才加入了try关键字专门用来异常处理。

# 处理异常

## 一、使用try关键字处理异常
*  使用try关键字处理异常的方式有两种，一种是使用`try!`告诉编译器调用将不会出现异常情况，即不处理异常，使用`try!`是一件很危险的事，一旦方法抛出异常，将会使客户端出现一些未知的错误甚至于崩溃。所以和强制解包一样，尽量不要去使用try！，否则会出现难以预料的情况。
*  另一种便是使用`do { try function() } catch {}`处理可能出现的异常情况

将上面的代码改为
```swift
do {
     let dic = ["json": "String"]
     let data = try JSONSerialization.data(withJSONObject: dic, options: JSONSerialization.WritingOptions.prettyPrinted)
    } catch (let error){
     print(error.localizedDescription)
      //在这里处理所有异常情况  
 }
```    
 使用catch的方式和模式匹配一样，可以根据出现的异常情况做出不同的处理方式，这里先定义一个`enum`类型
```swift 
enum Faild: Error {
     case error1
     case error2
}  
do {
    let dic = ["json": "String"]
    let data = try JSONSerialization.data(withJSONObject: dic, options: JSONSerialization.WritingOptions.prettyPrinted)
     } catch Faild.error1 {
            //如果捕获的异常为error1，在这里处理
     } catch Faild.error2 {
            //如果捕获的异常为error2，在这里处理
     } catch (let error){
            print(error.localizedDescription)
            //这里处理除了error1和error2出现的其他所有异常情况
}
```
在上面这段代码中，我们用catch匹配了三种可能出现的异常，每种出现的异常对应不同的处理方法，**但实际上代码永远不会执行error1和error2中的情况**，因为`JsonFaild`是自定义错误类型，默认的系统函数并不会抛出此类异常，接下来我们来定义一个可以抛出自定义错误的`throws`方法
```swift 
enum Faild: Error {
    case notString
    case notNumberString
}       
func convertToNumber(obj: Any) throws -> Double {
    if let string = obj as? String {
        if let result = Double(string) {
             return result
        } else {
            throw Faild.notNumberString
        }
     }else {
            throw Faild.notString
    }
}
```
 调用此函数的时候我们便可以这样处理,**要注意的是同时调用三个throws标记的方法可以不必写3个do catch，只需要一次即可，编译器会正确处理**
```swift 
 do {
    let result1 = try convertToNumber(obj: [1])
    let result2 = try convertToNumber(obj: "number")
    let result3 = try convertToNumber(obj: "1.2")
} catch Faild.notString {
        //在调用convertToNumber(obj: [1])方法时，将会捕获异常跳转到这里，在这里编写处理异常的函数
} catch Faild.notNumberString {
        //在调用convertToNumber(obj: "number")方法时，将会捕获异常跳转到这里，在这里编写处理异常的函数
} catch {
        //注意:最后的catch必须写，否则编译器会报错，所有捕获的其它异常类型必须在此处理
}
```              
    
## 二、将错误传递给上一级调用者
在一个函数内部如果调用了throws标记的函数，可以将当期函数也标记为throws，就可以将错误传递给上一级调用者，例如
```swift
func function() throws {
    _ = try convertToNumber(obj: "123")
}
```
# 总结

*现在Swift的错误处理的语法已经很许多常见的语言一样，我们可以简单的使用`do catch`来模式匹配异常情况并正确处理。一个不同的地方便是在Swift中，我们只能将函数标记为`throws`，并不能在函数接口中定义将会抛出的错误类型，只能在函数内部指定抛出的异常类型，所以如果不是自己定义的throws函数，调用者并不能知道当前函数调用的函数会抛出哪一种类型的异常，将来的Swift版本可能会改进这一点。*



