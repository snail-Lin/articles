---
layout: post
title:  "Swift URL encode"
date:   2017-06-15
categories: Swift
excerpt: "根据RFC文档，统一资源标志符（URI）中的特殊字符需要进行编码传输，而统一资源定位符（URL）作为URI的子集，也需要对其进行编码"
tag:
- Swift
- URLEncode
comments: true
---

### 前言
在WEB前端开发，服务器后台开发，或者是客户端开发中，对URL进行编码是一件很常见的事情，但是由于各个年代的RFC文档中的内容一直在变化，一些年代久远的代码就对URL编码和解码的规则和现在的有一些区别。  

在1994年订制的RFC1738文档中，对字符串中的除了`- _ .`之外的所有非字母数字字符都替换成百分号(%)后跟两位十六进制数，十六进制数中字母必须为大写。  

在2005年定义的RFC3986中，将针对`- _.~`四个字符之外的所有非字母数字字符进行百分号编码。当然
根据URL的类型不同，有也一部分预留字符不需要进行编码，例如查询的`URL`中可以包含`? /`字符，不需要转义。更详细文档的可以查看[RFC 3986](http://www.ietf.org/rfc/rfc3986.txt)。

### Swift url encode
`addingPercentEncoding(withAllowedCharacters:`是iOS7之后出现的新API用于`url encode`。
#### 标准转码
所有类型的URL中,`"-_.~"`都不应该被转码
```swift
var str = "-_.~"
var encodeStr = str.addingPercentEncoding(withAllowedCharacters: .urlHostAllowed)
print(encodeStr ?? "") // -_.~
```
```swift
var str = "#"
var encodeStr = str.addingPercentEncoding(withAllowedCharacters: .urlHostAllowed)
print(encodeStr ?? "") // %23
```
#### CharacterSet
`CharacterSet`是一个结构体，`CharacterSet.urlHostAllowed`等预制类型包含了所有不需要被转码的字符，反过来说就是指明了需要被转码的字符。`CharacterSet`类中提供了一些常用的URL转码的类型:

```
* CharacterSet.urlHostAllowed: 被转义的字符有  "#%/<>?@\^`\{\|\}
* CharacterSet.urlPathAllowed: 被转义的字符有  "#%;<>?[\]^`\{\|\}
* CharacterSet.urlUserAllowed: 被转义的字符有   #%/<>?@\^`\{\|\}
* CharacterSet.urlQueryAllowed: 被转义的字符有  "#%<>[\]^`\{\|\}
* CharacterSet.urlPasswordAllowed 被转义的字符有 "#%/:<>?@[\]^`\{\|\}
```

为什么说`CharacterSet.urlHostAllowed`包含的是所有不需要被转码的字符，可以用两句代码验证:
```swift
let unicode = "1".unicodeScalars.flatMap{ $0 }[0]

print(CharacterSet.urlHostAllowed.contains(unicode)) //输出TRUE
```
所以，自定义转码字符的集合应该取反字符集:
```swift
let str2 = "#/%/<>?@"
let custom = CharacterSet(charactersIn: "#").inverted
let result = str2.addingPercentEncoding(withAllowedCharacters: custom) ?? ""

print(result) //输出 %23/%/<>?@
```
可以看到只有`#`被编码。在一些特殊的需求中会用到自定义编码集合，例如BASE64转码后的URL编码。

### Objective-C url encode
API调用都是一样的,不过网上流传的比较多的是用的`C API`

```swift
NSString *ciphertext = @"saf#*&";        
NSCharacterSet *set = [[NSCharacterSet characterSetWithCharactersInString:@"!*'();:@&=+$,/?%#[]"] invertedSet];
        
NSString *resultString = [ciphertext stringByAddingPercentEncodingWithAllowedCharacters: set];
```

C API 
```swift
NSString *ciphertext = @"saf#*&";
NSString *encodedStr = (NSString *)CFBridgingRelease(CFURLCreateStringByAddingPercentEscapes
                                                     (kCFAllocatorDefault,
                                                      (CFStringRef)ciphertext,
                                                      NULL,
                                                      CFSTR("!*'();:@&=+$,/?%#[]"),
                                                      kCFStringEncodingUTF8));
```

