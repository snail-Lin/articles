---
layout: post
title:  "制作基于Swift与Objective-C混编的Framework"
date:   2017-06-01
categories: Swift
excerpt: "可以将编译生成的二进制文件，头文件，资源文件统一打包为Framework，方便共享代码实现而不必开放所有源代码"
tag:
- Swift
- Objective-C
- Framework
comments: true
---

### 前言
最近公司正在和另外一个公司合作开发一个项目，一个APP中的一部分功能需要由我来实现，大部分的功能由另一个公司完成。所以需要将代码打包成一个Framework，将实现的接口暴露出来给他们调用即可，这样代码就可以分为两部分，方便各自公司以后的代码维护。  
考虑到功能有一小部分和之前公司的项目重合，之前的代码是经过多年的迭代已经是非常稳定的了，再进行Swift的重构并不划算。最好还是旧功能部分用之前的代码，新功能部分用Swift编写，这样就需要打造一个Swift与Objective-C的Framework，项目到完成期间还是经历了不少小坑。现在网上大部分的教程已经并不适用，所以写一篇详细的文章以供他人参考。

### CoCoa Touch Framework 和 Cocoa Touch Static Library 的区别
首先要理解`Static Library(静态库)`和`Dynamic Library(静态库)`的区别。  
静态链接库会将模块编译合并到应用中，会增大应用程序的体积，不需要其它的依赖。如果有多个应用使用了同一个静态库，那么就会有多个同样的静态库冗余在内存中。  
动态库顾名思义就是在运行时候查找并载入到内存里，多个程序可以共用同一个动态库，例如`UIKit`就是一个动态=库，不存在冗余现象。
#### Static Library VS Framework
`Static Library`仅能包含编译后的代码，即`.a`文件，其它的文件只能使用`Bundle`打包。而且Swift代码不支持编译为`Static Library`。 
 
Framework中同样的也包含编译后的二进制文件，还包含了其他的资源:

* 头文件 - 也包含Swift symbols所生成的头文件，如 Alamofire-Swift.h。
* 所有资源文件的签名 - Framework被嵌入应用前都会被重新签名。
* 资源文件 - 像图片等文件。
* Dynamic Frameworks and Libraries - 参见Umbrella Frameworks
* Clang Module Map 和 Swift modules - 对应处理器架构所编译出的Module文件
* Info.plist - 该文件中说明了作者，版本等信息。

Cocoa Touch Framework 实际上可以分为三个部分的内容: `Header`、`动态链接库`、`资源文件`。  
  
  
**因为iOS中的沙盒机制，即使使用了动态链接库，也不能跨APP间共享动态库，但是iOS8之后出现了`App Extension`，可以在APP和Extension之间共享动态库, 所以`CoCoa Touch Framework`仅能用于iOS8之后的版本，这也是为什么Carthage不支持iOS7的原因。  
还有一个体现动态链接库的方式就是在你的Framework代码中中引用第三方库，在打包Framework时并不会将第三方库的Framework文件合并到自己的Framework包中，但是使用你提供的Framework的应用中必须引用同样的第三方库，应用打包时只会集成一份第三方库的Framework文件，不会出现冗余文件**

如果你的应用只需要支持iOS8之后的iOS版本，推荐使用`Cocoa Touch Framework`
![](/assets/img/framework/framework1.png)

### Swift与Objective-C代码的互相调用
创建一个`Cocoa Touch Framework`，

分别创建一个Objective-C和Swift的类，其中内容如下. 
OCDemoView.h
```swift
@interface OCDemoView : UIView
@property (nonatomic, strong) UILabel *titleLabel;
@end
```
OCDemoView.m
```swift
#import "OCDemoView.h"

@implementation OCDemoView

@end
```
SwiftDemoView.swift
```swift
import UIKit

public class SwiftDemoView: UIView {

    public let titleLabel: UILabel = UILabel()

}
```
#### Objective-C中调用Swift代码
1.注意将`类`和`属性`声明为`pubulic`或者`open`，只有暴露出来的类才能被`Objective-C`代码调用，最终打包的Framework也会暴露出这些Swift类。  
2.和正常的应用一样，Objective-C代码中使用Swift类需要导入`bridge`头文件，默认的Bridge头文件名为`项目名-Swift.h`,导入方式和普通的应用不太一样，需要使用`Modules`导入，调用代码如下:
```swift
#import "DemoManager.h"
#import <DemoFramework/DemoFramework-Swift.h>

@implementation DemoManager

- (void)test {
    SwiftDemoView *view = [[SwiftDemoView alloc] init];
}

@end
```

#### Swift中调用Objective-C代码
在`umbrella Header`里导入需要调用的类的头文件:
![](/assets/img/framework/framework6.png)
并且在`Build Phases`的`Headers`下将头文件拖入`Public`列表下:
![](/assets/img/framework/framework5.png)
调用代码:
```swift
import UIKit

public class SwiftDemoView: UIView {
    ...
    func test() -> Void {
        let view = OCDemoView()
    }
}

```
#### 调用方式的缺点
上面的方式已经实现了Objective-C代码和Swift代码的代码的互相调用，但是有一个缺点就是暴露出了Objective-C文件的接口，混编的Framework，最终暴露给外部的接口应该是纯Swift的接口，Objective-C类的具体实现应该被隐藏。为了实现两种代码的互相调用而暴露出了不应该有的文件，得不偿失。

#### 通过modulemap在Swift中导入Objective-C头文件
`modulemap`文件是对一个框架，一个库的所有头文件的结构化描述。语法可以参照官方文档。  
创建一个空文件，并且命名为`module.modulemap`，其中内容如下:
```swift
//module的名字可以随便起,这里为OCModule，导入module会用到
module OCModule [system] {
    
    //头文件的路径,如果头文件和module.modulemap不在同一个目录，需要写出完整的路径
    //例如 library/**.h
    header "OCDemoView.h"
    export *
}
```
创建module文件后必须在`Build Settings`的`Import Paths`选项下添加文件的路径，如图:
![](/assets/img/framework/framework7.png)
设置好之后，在`umbrella Header`将刚才导入的OCDemoView.h的代码删除，并且在`Headers`中重新将OCDemoView.h拖回`Project`列表，这时候在Swift中调用Objective-C代码需要添加导入`module`,使用`import OCModule`导入，完整代码:
```swift
import UIKit
import OCModule

public class SwiftDemoView: UIView {

    public let titleLabel: UILabel = UILabel()
    
    func test() -> Void {
        let view = OCDemoView()
    }
}

```
这样的调用方式便符合了Framework的封装性。
### 在Framework中使用Xib，图片等资源
#### 使用资源文件
在普通的应用中，加载图片的代码很简单
```swift
let image = UIImage(named: "a.png")
```
如果在Framework中也这么写的话，别人集成你的Framework后将会发现有图片的界面一片空白，即加载不到所需的图片。因为默认的资源库是集成Framework的应用的`Bundle`，而不是Framework下的`Bundle`,正确的调用代码是:
```swift
let innerBundle = Bundle(identifier: "com.hdl.lin.DemoFramework")
    
let image = UIImage(named: "a.png", in: innerBundle, compatibleWith: nil)
```
要注意这里创建`Bundle`的`identifier`参数不是项目名，而是Framework的`Bunlde Identifier`。

加载Xib的方式一样,不要使用默认的`Bundle`:
```swift
public class func viewFromeNib() -> SwiftDemoView {
    return Bundle(identifier: "com.hdl.lin.DemoFramework")?.loadNibNamed("SwiftDemoView", owner: self, options: nil)?[0] as! SwiftDemoView
}
```
这里只是写个简单的实例代码，在`Swift`中还是尽量不要使用强制解包。

加载其它资源：
```swift
let txt = Bundle(identifier: "com.hdl.lin.DemoFramework")?.path(forResource: "1", ofType: "txt")
```

#### 在Xib中加载图片
和普通的项目一样，XiB中设置图片直接添加图片,程序会正确加载Framework中的图片资源，不用另外设置其他选项。

### 创建Framework
选择`Generic iOS Device`设备编译，这样打包出的Framework可用于应用的真机调试和Release打包。
![](/assets/img/framework/framework8.png)

编译成功后，项目的`Products`会多出一个`Framework`文件，打开文件目录查看
![](/assets/img/framework/framework9.png)

可以看到Framework的目录结构，其中`DemoFramewok`就是代码编译后的二进制文件，和静态库的`.a`文件是一样的。
![](/assets/img/framework/framework10.png)

可以使用`lipo -info`命令查看当前库所支持的CPU架构，注意选择Framework下的二进制文件，而不是Framework本身。
![](/assets/img/framework/framework11.png)
可以看到其中不包含模拟器

选择模拟器再次编译，编译成功后，查看Framework的上一级目录,接下来就是合并这两个Framework
![](/assets/img/framework/framework12.png)

### 合并真机和模拟器的Framework
包含Swift代码的Framework和单独用Objective-C编写的Framework过程不太一样，除了合并`二进制文件`，还需要复制`*.swiftmodule`文件夹下的文件里的文件到一起。

#### 合并二进制文件
直接使用`lipo -create 真机路径 模拟器路径 -output 输出路径`命令合并二进制文件，将合并后的二进制文件重命名去掉后缀，覆盖原Framework下的文件。合并完后使用`lipo -info`命令查看是否正确。 
![](/assets/img/framework/framework14.png)

#### 合并*.swiftmodule文件夹下的文件
将文件复制过来即可,这样最终生成的Framework就可以用于真机和模拟器，你也可以在`Build Phases`下添加一个`Script Build Phase`脚本在编译后自动合并。
![](/assets/img/framework/framework13.png)

### Framework的一些优化
网上的大部分教程都提倡将`Dead Code Stripping`设置为`NO`,这是不合理的。  
这个选项设置为`YES`后，编译器会自动在生成的Framework中删除对象文件中不需要加载的符号，即不会包含任何不被执行的代码，有利于减少二进制文件的大小。

如果你需要减少Framework的体积，将`Strip Debug Symbols During Copy`选项设置为`YES`，这会在编译后的二进制文件中去掉不必要的`symbols（符号）`，将使用一个单独的符号文件来表示崩溃日志。一般情况下还是设置为`NO`。

尽量避免暴露非必要的接口，暴露的接口尽量简洁，方便他人调用，进行单元测试时也更简单。

