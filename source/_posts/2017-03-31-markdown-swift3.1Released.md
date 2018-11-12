---
layout: post
title:  "Swift 3.1 的一些改动"
date:   2017-03-31
categories: Swift
excerpt: "Swift3.1已经发布，包含了一些对标准库的改进和完善"
tag:
- Swift 
- throws
- 异常处理
comments: true
---

  要体验Swift3.1，只要把Xcode升级到8.3即可，根据苹果的官方文档介绍，这次只是一个小更新，包含了一些对于标准库的改进和完善，也有一部分更新是针对Swift包管理器的改进。

## Sequence协议
  首先就是对于`Sequence`协议的改进，添加了两个关联协议，即使遵循Sequence协议，也不用实现这两个方法，因为Swift标准库中已经对`drop`和`prefix`这两个方法有了默认实现。
  ```swift
protocol Sequence {
    func drop(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> Self.SubSequence
    func prefix(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> Self.SubSequence
}
```
  Swift3.1要满足Sequence的协议，依然是提供一个返回迭代器的方法，如果迭代器的类型能够推断出，`Iterator`也不必声明。
```swift
protocol Sequence {
    associatedtype Iterator: IteratorProtocol
    func makeIterator() -> Iterator
}
```
查看更多:[SE-0045: Add prefix(while:) and drop(while:) to stdlib](https://github.com/apple/swift-evolution/blob/master/proposals/0045-scan-takewhile-dropwhile.md)  

## availability关键字
现在使用`@availability`可以标记出当前方法的Swift版本生命周期，如果要将一个方法在Swift3.1中标记为废弃，可以这样标记
```swift
@available(swift, obsoleted: 3.1)
class Foo {
  //...
}
```
查看更多:[SE-0141: Availability by Swift version](https://github.com/apple/swift-evolution/blob/master/proposals/0141-available-by-swift-version.md)
## 包管理器更新
软件包的依赖默认存储在工具管理的构建目录下，并且一个新的`swift package edit`命令将允许用户直接在软件包上进行编辑，避免了依赖更新，允许用户直接提交和推送软件包的更新。  
查看更多:[SE-0082: Package Manager Editable Packages](https://github.com/apple/swift-evolution/blob/master/proposals/0082-swiftpm-package-edit.md)

## Swift在Linux上实现的一些改进
*   Implementation of `NSDecimal`
*   Implementation of `NSLengthFormatter`
*   Implementation of `Progress`
*   Many improvements to `URLSession` functionality, including API coverage and optimized usage of `libdispatch`
*   Improved API coverage in `NSArray`, `NSAttributedString` and many others
*   Significant performance improvements in `Data`. See more details here
*   Improved JSON serialization performance
*   Memory leaks fixed in `NSUUID`, NSURLComponents and others
*   Improved test coverage, especially in `URLSession`

**还有更多的内容，苹果官方发布主页上都有详细的介绍->**
[Swift 3.1 Released!](https://swift.org/blog/swift-3-1-released/)

这次的改动并不会影响到绝大多数代码的语法，所以这次直接升级Xcode8.3后，即使有上百个Swift文件，编译器也没有任何报错需要修改的地方。没有像Swift2.3升级到Swift3.0后各种痛不欲生的语法更改和未知错误。这次的升级反而加快了Swift的编译速度，所以建议大家尽快升级Xcode8.3，也是为了尽快能适配iOS 10.3 SDK。  

在官方文章的最后，苹果还给出了Swift4将来的一部分改动和发布日期。**Swift4将在2017年秋季完成，并将提供更高的稳定性支持，包含对核心语言和标准库的增强，特别是在泛型和String类型部分有更大的改进。**对于Swift4，我个人非常期待能够早日发布。比较Swift这么多版本的迭代后，已经越来越趋于稳定，在不久的将来就会取代Objective-C，只不过国内的日程可能会比国外稍慢一些。

