---
layout: post
title:  "在Swift中优雅的使用正则表达式"
date:   2017-05-08
categories: Swift
excerpt: "正则表通常被用来检索、替换那些符合某个模式(规则)的文本，在Swift中使用NSRegularExpression来实现，但是API并不是那么优雅"
tag:
- regular expression 
- Swift
comments: true
---

### 前言
在开发过程中使用正则表达式过滤和判断文本是很常见的做法。Swift中的正则表达式使用起来很不方便，需要使用`NSRegularExpression`创建一个操作正则的对象，而`JavaScript`中使用正则表达式的方式就颇为优雅，判断只需要一句代码
```JavaScript
//使用JavaScript判断是否是数字
var isNumber = (/^[0-9]*$/g).test("123424");
```
在Swift中，也可以实现这样简洁的代码甚至于更简洁

### 1.封装
NSRegularExpression的用法和Objective-C中使用方法一样,在Swift中还可以使用`map`等高级函数可以使代码更为简短。
```swift
struct Regex {
    
    let reg: NSRegularExpression
    
    init?(regString text: String, options: NSRegularExpression.Options = []) {
        do {
            self.reg = try NSRegularExpression(pattern: text, options: options)
        } catch {
            return nil
        }
    }
    
    func match(with text: String, options: NSRegularExpression.MatchingOptions = []) -> [String] {
        
        let results = self.reg.matches(in: text, options: options, range: NSRange(location: 0, length: text.characters.count))
        return results.map{ (r) -> String in
            //这里使用NSString的原因是无法将NSRange转换为Range<String.Index>类型
            let str = text as NSString
            return str.substring(with: r.range)
        }
    }
    
    func test(with text: String, options: NSRegularExpression.MatchingOptions = []) -> Bool {
        return self.match(with: text, options: options).count > 0
    }
}
```
虽然还是必须使用`NSString`作为`temp`类型，但是使用起来还是方便多了
```swift
if let result = Regex("1")?.test(with: "11"), result == true {
    print("字符串符合条件")
}
if let results = Regex("^[0-9]*$")?.match(with: "1234") {
    print("搜索结果: \(results)")
}
```
`NSRegularExpression.Options`的各个选项的作用为:
```objc
NSRegularExpressionCaseInsensitive             = 1 << 0,     // 不区分大小写字母模式

NSRegularExpressionAllowCommentsAndWhitespace  = 1 << 1,     // 忽略掉正则表达式中的空格和#号之后的字符

NSRegularExpressionIgnoreMetacharacters        = 1 << 2,     // 将正则表达式整体作为字符串处理

NSRegularExpressionDotMatchesLineSeparators    = 1 << 3,     // 允许.匹配任何字符，包括换行符

NSRegularExpressionAnchorsMatchLines           = 1 << 4,     // 允许^和$符号匹配行的开头和结尾

NSRegularExpressionUseUnixLineSeparators       = 1 << 5,     // 设置\n为唯一的行分隔符，否则所有的都有效。

NSRegularExpressionUseUnicodeWordBoundaries    = 1 << 6      // /使用Unicode TR#29标准作为词的边界,否则所
```
`NSRegularExpression.MatchingOptions`的各个选项的作用为:
```objc
NSMatchingReportProgress         = 1 << 0,       // 找到最长的匹配字符串后调用block回调

NSMatchingReportCompletion       = 1 << 1,       // 找到任何一个匹配串后都回调一次block

NSMatchingAnchored               = 1 << 2,       // 从匹配范围的开始进行极限匹配

NSMatchingWithTransparentBounds  = 1 << 3,       // 允许匹配的范围超出设置的范围

NSMatchingWithoutAnchoringBounds = 1 << 4        // 禁止^和$自动匹配行开始和结束
```

#### 重载match方法使其可用于数组过滤
在之前写过一篇[关于Swift重载](https://www.yihanv.com/markdown-overloading/)的文章。  
想要将正则表达式作用于数组可以简单的重载`match`方法,将参数类型改为`[String]`,添加重载方法。
```swift
    func match(with text: [String], options: NSRegularExpression.MatchingOptions = []) -> [String] {
        return text.map{ self.match(with: $0) }.reduce([], { $0 + $1 })
    }
```
使用效果如下:
```swift
if let results = Regex("^[0-9]*$")?.match(with: ["1234", "3456", "789"]) {
    print("搜索结果: \(results)")//搜索结果: ["1234", "3456", "789"]
}
```

### 2.自定义运算符
如果是Swift库中没有定义过的运算符，需要先定义操作符的属性例如优先级等等。这里定义一个运算符
```swift
func =~(input: String, reg: String) -> Bool {
    if let result = Regex(reg)?.test(with: input) {
        return result
    }
    return false
}
```
要注意result不管是`false`还是`true`都不会执行`return false`语句，`if`仅仅判断的是`result`是否为`nil`。  
这样我们就可以使用自定义的表达式来过滤字符串
```swift
if "123" =~ "^[0-9]*$" {
    print("符合条件")
}
```
同样的可以重载操作符的方法，以达到可以直接过滤字符串数组的效果

### 3.扩展
#### 为String扩展方法
创建一个Regex的方法看起来也不是那么整洁，所以我们直接扩展String类型，为String添加使用正则表达式的过滤方法。
```swift
extension String {
    func match(with reg: String,
               options: NSRegularExpression.Options = [],
               matchOptions: NSRegularExpression.MatchingOptions = []) -> [String] {
        if let result = Regex(reg, options: options)?.match(with: self, options: matchOptions) {
            return result
        }
        return []
    }
    
    func test(with reg: String,
              options: NSRegularExpression.Options = [],
              matchOptions: NSRegularExpression.MatchingOptions = []) -> Bool {
        if let result = Regex(reg, options: options)?.test(with: self, options: matchOptions) {
            return result
        }
        return false
    }
}
```
使用扩展方法:
```swift
if "12334".test(with:"^[0-9]*$") {
    print("字符串符合条件")
}
```

#### 为字符串序列扩展方法
```swift
extension Sequence where Iterator.Element == String {
    func match(with reg: String,
               options: NSRegularExpression.Options = [],
               matchOptions: NSRegularExpression.MatchingOptions = []) -> [Iterator.Element] {
        let reg = Regex(reg, options: options)
        return self.map{ reg?.match(with: $0, options: matchOptions) }.flatMap{ $0 }.reduce([], {$0 + $1})
    }
}
```
使用方式:
```swift
let results = ["123", "456", "aa"].match(with: "^[0-9]*$")
print(results) //输出["123", "456"]
```

### 总结
其实这一篇和正则表达式的关系并不是很大，最重要的是想说明`自定义类型`，`自定义操作符`，和`类的扩展`这3种方式编写API的效果的区别在哪里，最终想要用哪一种方法，仁者见仁智者见智，没有绝对的使用方式。在不同的情况下使用不同的方式，可以使代码更为优雅。


