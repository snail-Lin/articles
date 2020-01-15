---
layout: post
title:  "libstdc++.6.dylib问题"
date:   2019-12-12
categories: Experience
excerpt: ""
tag: 
- Xcode
- iOS
- Experience
comments: true
---

苹果在 XCode10 和 iOS12 中移除了 `libstdc++` 这个库，由 `libc++` 取而代之，苹果建议大家使用经过了 llvm 优化过并且全面支持C++11的 `libc++` 库。

#### 正确的解决方案

- 将你的代码依赖调整为 libc++ ，重新编译
- 依赖包使用了 libstdc++ ，那么应该更新依赖包

#### 临时解决方案
如果依赖包也没有更新的话，那只能使用临时的解决方案

#### 临时解决方案
将`libstdc++`文件复制到相关文件夹下， 文件下载[libstdc++.zip](https://www.yuque.com/attachments/yuque/0/2019/zip/117839/1576839624885-43c1982a-f4e6-4c64-9056-01cecafa8ac5.zip?_lake_card=%7B%22uid%22%3A%221576839624678-0%22%2C%22src%22%3A%22https%3A%2F%2Fwww.yuque.com%2Fattachments%2Fyuque%2F0%2F2019%2Fzip%2F117839%2F1576839624885-43c1982a-f4e6-4c64-9056-01cecafa8ac5.zip%22%2C%22name%22%3A%22libstdc%2B%2B.zip%22%2C%22size%22%3A302920%2C%22type%22%3A%22application%2Fzip%22%2C%22ext%22%3A%22zip%22%2C%22progress%22%3A%7B%22percent%22%3A99%7D%2C%22status%22%3A%22done%22%2C%22percent%22%3A0%2C%22id%22%3A%22p2hoy%22%2C%22refSrc%22%3A%22https%3A%2F%2Fwww.yuque.com%2Fattachments%2Fyuque%2F0%2F2019%2Fzip%2F117839%2F1576839624885-43c1982a-f4e6-4c64-9056-01cecafa8ac5.zip%22%2C%22card%22%3A%22file%22%7D)

模拟器运行（XCode 10 ）
将`CoreSimulator` 文件夹下的内容复制到以下路径

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/usr/lib/
```

模拟器运行（XCode 11 ）
将`CoreSimulator` 文件夹下的内容复制到以下路径

```
 /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/usr/lib/
```

模拟器编译
将`iPhoneSimulator` 文件夹下的内容复制到以下路径

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/
```

iOS真机调试
将 `iPhoneOS` 文件夹下的内容复制到以下路径

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/usr/lib/
```

MacOS 运行
将 `MacOSX` 文件夹下的内容复制到以下路径

```
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/lib/
```

#### 临时方法之模拟器崩溃解决
首先要知道iOS模拟器的路径
1.下载路径

```
~/Library/Caches/com.apple.dt.Xcode/Downloads
```

**2.Xcode自带模拟器**


Xcode10

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes
```


Xcode11

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Library/Developer/CoreSimulator/Profiles/Runtimes
```

**3.Xcode升级后，原有的模拟器将会放置在这个路径**
**
```
/Library/Developer/CoreSimulator/Profiles/Runtimes
```


**解决崩溃**，需要将 `CoreSimulator` 文件下的内容 复制到模拟器的 `Runtimes` 目录下，直接右键 `Runtimes` 显示包内容，然后粘贴文件即可。
