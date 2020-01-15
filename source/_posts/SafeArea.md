---
layout: post
title:  "Safe Area"
date:   2019-11-02
categories: Experience
excerpt: ""
tag: 
- iOS
- Safe Area
- Experience
comments: true
---

iOS 11 开始引入了 Safe Area 这个概念。

#### UIView 中的 Safe Area
iOS 11 中 UIViewController 的 `topLayoutGuide` 和 `bottonLayoutGuide` 两个属性被 UIView 中的 `Safe Area` 属性替代。表示相对于边缘的距离，如果不使用Auto Layout布局而是通过计算frame来布局的时候，这个属性来辅助计算安全距离。

```swift
@available(iOS 11.0, *)
open var safeAreaInsets: UIEdgeInsets { get }

@available(iOS 11.0, *)
open func safeAreaInsetsDidChange()
```


#### UIViewController 中的 Safe Area

在 iOS 11 中 UIViewController 有一个新的属性

```swift
@available(iOS 11.0, *)
open var additionalSafeAreaInsets: UIEdgeInsets
```

安全区域改变时的回调方法

```swift
// UIView
@available(iOS 11.0, *)
open func safeAreaInsetsDidChange()

//UIViewController
@available(iOS 11.0, *)
open func viewSafeAreaInsetsDidChange()
```

当 `UIViewController` 的子视图覆盖了嵌入的子 view controller 的视图的时候。比如说， 当 UINavigationController 和 UITabbarController 中的 bar 是半透明(translucent) 状态的时候, 就有 additionalSafeAreaInsets

#### safeAreaLayoutGuide
`safeAreaLayoutGuide`继承`UILayoutGuide`，可用于AutoLayout布局，可以基于这个属性，设置AutoLayout的约束。

```swift
UILayoutGuide *safeGuide = self.view.safeAreaLayoutGuide;
NSLayoutConstraint *topConstraint = [label.topAnchor constraintEqualToAnchor:safeGuide.topAnchor];
```

总结就是：

- 它是UIView的一个只读属性，意味着所有UIView对象都有并且是系统帮我们创建好的
- 它继承UILayoutGuide，有layoutFrame意味着它能代表一块区域
- 它代表的区域避开了诸如导航栏、tabbar或者其他有可能挡住你这个UIView对象显示的所有父view，意味着你的view对象只要相对另一个view的safeLayoutGuide做布局就不用担心她被奇奇怪怪的东西挡住
- 对于控制器的view的safeAreaLayoutGuide，他的区域同样避开了statusbar或其他有可能挡住view显示的东西，我们甚至可以用控制器的additionalSafeAreaInsets属性，来额外指定inset
- 如果view完全在父view的安全区域内，或者view不在视图层级或屏幕上，那么view的safeAreaLayoutGuide区域其实和view自身是一样大的
