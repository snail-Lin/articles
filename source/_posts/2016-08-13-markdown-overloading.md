---
layout: post
title:  "深入了解Swift中的重载"
date:   2016-08-13
categories: Swift
excerpt: "拥有同样名称，参数或返回类型不同的多个方法互相称为重载方法"
tag:
- Swift 
- 重载
comments: true
---

## 一、运算符的重载
本文中使用的Swift的版本为3.0
### 自定义运算符
如果swift库中没有定义的运算符，需要使用关键字`infix`,这里先自定义一个次方运算符
```swift
precedencegroup NthrootPrecedence {
     //左结合
     associativity: left
     //优先级需要大于乘法
      higherThan: MultiplicationPrecedence
}

infix operator ^&: NthrootPrecedence

func ^&(lhs: Double, rhs: Double)  -> Double {
     return pow(lhs, 1.0 / rhs)
}

print(8 ^& 3) //8开3次方，输出2.0
print(4 ^& 2) //4开2次方，输出2.0
```    
### 重载自定义的次方运算符
目前次方运算符只能用于`Double`类型，运算结果也是Double类型。如果要直接使用在自定义类中，我们可以为其定义一个重载。
#### 先定义一个测试类
```swift    
class Figure {
    var count: Double
    var nthroot: Double
        
    init(count: Double, nthroot: Double) {
         self.count = count
            self.nthroot = nthroot
        }
    
        convenience init(count: Double) {
            self.init(count: count, nthroot: 2)
       }
    }   
```
#### 重载运算符方法
```swift
func ^&(lhs: Figure, rhs: Figure) -> Figure {
     //这里直接调用刚才定义的^&方法，Swift会自动编译合适的方法
     return Figure(count:lhs.count ^& rhs.nthroot)
}

let f1 = Figure(count: 16, nthroot: 2)
let f2 = Figure(count: 8, nthroot: 4)
let f3 = f1 ^& f2

print("count:\(f3.count), nthroot:\(f3.nthroot)")//这里输出count:2.0, nthroot:2.0
```
`Figure`对象已经能够直接使用次方根运算符直接计算

## 二、泛型约束的重载
### 定义一个泛型函数
在Swift中，经常会写一些泛型函数，重载函数可以使其用途更为广泛。
这里先定义一个泛型函数，作用是计算数组中所有数字的和，返回值为`Int`类型。
```swift    
extension Sequence where Iterator.Element: SignedInteger {

    func combineCount() -> Int {
    
      return self.reduce(0) {
        if let count = $1 as? Int {
            return $0 + count
         }
        return 0
       }
     }
}

print([1, 2, 3, 4].combineCount()) //输出10
```    
如果是`Double`类型的数组
```swift    
print([1.0, 2.0].combineCount()) //调用失败
```    
这里编译器将会报错，因为当前函数没有定义`Double`类型的参数，所以重载`combineCount`函数，使其可以用在`Double`类型数组上

### 重载泛型函数
```swift
extension Sequence where Iterator.Element: BinaryFloatingPoint {

     func combineCount() -> Double {
        return self.reduce(0) {
            if let count = $1 as? Double {
                 return $0 + count
             }
         return 0
        }
     }
}
    
print([1.0, 2.0 , 3.0, 4.0].combineCount()) //输出10.0
```    
## 总结

在调用函数时，Swift的类型检查器找到一个最精确的重载函数
**重载的使用是在编译期间静态决定的**，编译器会根据变量的`静态类型`来决定要调用哪个重载函数。而在Objective-C中，方法调用是在运行时根据调用对象的`动态类型`来决定的

来看一段代码
```swift
let arrs: [[Double]] = [[1, 2, 3], [1.0, 2.0, 3.0]]
    for array in arrs {
       print("result: \(array.combineCount())")
}
/*
    输出结果
    result: 6.0
    result: 6.0
*/
```
上面说道，Swift的类型检查器找到一个最精确的重载函数，应该输出6和6.0而不是6.0和6.0。这是因为现在数组内的所有元素的静态类型都是Double类型，所以对应的重载函数也应该是Double类型的重载函数。
如果将arrs后的[[Double]]类型定义去掉，数组内部的元素调用combineCount函数将会报错，因为这时候arrs的类型为`[Array\<Any\>]`,其内部元素的静态类型为Any，所以调用combineCount函数会报错,如下图所示
    
![](/assets/img/overloading/1.png)

所以我们在使用重载函数的时候，要多注意Swift中的`静态编译`。


