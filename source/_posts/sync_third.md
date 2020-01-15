---
layout: post
title:  "修改依赖包代码并同步更新"
date:   2019-12-30
categories: Experience
excerpt: "每一门语言或者开发平台都有其依赖的生态，随着生态发展，都会出现相应的依赖管理工具"
tag:
- Experience
comments: true
---

### 1.依赖包管理
每一门语言或者开发平台都有其依赖的生态，随着生态发展，都会出现相应的依赖管理工具，目前前端项目对应的依赖管理工具如下：

| 语言 | 平台 | 依赖管理工具 |
| --- | --- | --- |
| Objective-C、Swift | iOS | CocoaPods、Carthage |
| Kotlin、Java | Android | Maven、Gradle |
| JavaScript、TypeScript | Web | NPM |


### 2.依赖包的修改
在开发过程中，经常会出现需要修改依赖包代码的情况，如果将依赖包的源代码直接拖拽到项目工程中进行修改，会有致命的缺点：

- 依赖包的源代码繁多，有些甚至超过了项目中的业务代码，长期下来，会导致整个项目工程代码混乱，维护困难
- 依赖包的源代码无法同步原作者的更新，对依赖包最熟悉的人，原作者无出其二，直接将源代码拖拽到项目工程，会导致依赖包更新困难，长期下来，便会疏于更新，导致项目留下安全或兼容隐患。

正确的做法应该是**克隆源代码仓库，在自己的仓库上进行所需要修改，并且保持和原作者同步，拉取最新代码。**
那么使用依赖管理工具直接引用`Git仓库`作为`依赖包`便是重点。

---


#### 2.1 NPM使用Git仓库作为依赖包源地址
以`weex-ui`为例，安装命令为
```bash
npm i weex-ui -S
```

package.json文件便会自动生成依赖包信息，或直接添加依赖信息，使用 `npm i` 命令更新
```bash
"dependencies": {
    "weex-ui": "^0.6.16"
 }
```

如果weex-ui的某个布局不符合项目风格，需要修改其中的组件代码，正确的步骤如下：

**1.找到weex-ui的源码仓库**
[https://github.com/apache/incubator-weex-ui](https://github.com/apache/incubator-weex-ui)

**2.clone一份并上传到自己的仓库，创建一个新分支进行代码修改**

```shell
#将远程仓库克隆到本地
git clone git@github.com:apache/incubator-weex-ui.git
cd incubator-weex-ui
#添加一个名为haidilao的远程仓库
git remote add haidilao git@git.haidilao.com:linyihan/incubator-weex-ui.git
#将本地仓库镜像到远程仓库
git push haidilao --mirror
#创建一个名为my_dev的分支并切换到新分支上
git checkout -b  my_dev
```

**3.修改package.json文件，将weex-ui版本号替换为Git仓库地址，修改后的配置如下：**

```bash
"dependencies": {
  "weex-ui": "git+ssh://git@git.haidilao.com:linyihan/incubator-weex-ui.git#my_dev"
}
```
其中的`Git+ssh://`表示使用将weex-ui的源改为Git仓库的地址，并且使用`ssh`方式克隆仓库。
最后面的`#my_dev`表示使用`my_dev`分支下的代码。

**4.运行 `npm i` 命令更新项目依赖即可**

---


#### 2.2 Carthage使用Git仓库作为依赖包源地址
以EFQRCode为例，因EFQRCode不支持Swift5版本（目前已经适配，可以直接使用），所以需要修改依赖包的源代码以便其支持Swift5。

不要直接修改Carthage文件下源代码。

**正确的修改步骤如下：**
**
**1.找到EFQRCode的源码仓库**
[https://github.com/EFPrefix/EFQRCode](https://github.com/EFPrefix/EFQRCode)

**2.clone一份并上传到自己的仓库，创建一个新分支进行代码修改、适配Swift5**

```shell
#将远程仓库克隆到本地
git clone git@github.com:EFPrefix/EFQRCode.git
cd EFQRCode
#添加一个名为haidilao的远程仓库
git remote add haidilao git@git.haidilao.com:linyihan/EFQRCode.git
#将本地仓库镜像到远程仓库
git push haidilao --mirror
#创建一个名为my_dev的分支并切换到新分支上
git checkout -b  my_dev
```

**3.修改Cartfile文件配置**
```shell
git "git@github.com:snail-Lin/EFQRCode.git" "my_dev"
```

即将 `github` 改为 `git` ，表示直接使用git远程仓库作为依赖包的源地址，最后面的 `my_dev` 为Git分支名称。

4.**运行 **`**carthage update --platform iOS **`**命令更新项目依赖即可**

---


#### 2.3 CocoaPods使用Git仓库作为依赖包源地址
前3个步骤和《Carthage使用Git仓库作为依赖包源地址》一样。

**修改 `Podfile` 文件配置**
```shell
pod 'EFQRCode', :git => 'git@github.com:snail-Lin/EFQRCode.git', :branch => 'my_dev'
```

**运行 `pod update` 更新项目依赖即可**

---


#### 2.4 Gradle使用Git仓库作为依赖包源地址
需要配和 `jitpack` 进行 `Android libraries` 的发布，可以使用你自己的Github仓库作为依赖包源地址。
以logger依赖为例

**1.首先找到logger的源码仓库，fork一份到自己的仓库**
[https://github.com/orhanobut/logger](https://github.com/orhanobut/logger)

**2.进入**[**https://jitpack.io/**](https://jitpack.io/)**，登录GItHub账号，选择自己的logger仓库进行发布**
![image.png](https://cdn.nlark.com/yuque/0/2019/png/117839/1576655983343-45b17603-386e-4ffc-95c1-5c2640889ed5.png#align=left&display=inline&height=57&name=image.png&originHeight=128&originWidth=1166&size=12460&status=done&style=none&width=523)

**3.选择版本发布**
![image.png](https://cdn.nlark.com/yuque/0/2019/png/117839/1576656298339-0e9fcf77-d54b-4907-b1eb-eb0cd65a26d0.png#align=left&display=inline&height=287&name=image.png&originHeight=652&originWidth=1180&size=55111&status=done&style=none&width=520)

**4.点击Get it，等待发布成功，自动生成配置代码**

```bash
	allprojects {
		repositories {
			...
			maven { url 'https://jitpack.io' }
		}
	}
```

```bash
	dependencies {
	        implementation 'com.github.snail-Lin:logger:Tag'
	}
```

**5.修改build.gradle 配置文件，最终配置如下**

```bash
buildscript {
    ext.kotlin_version = '1.3.61'
    repositories {
        maven { url 'https://jitpack.io' }
        google()
        jcenter()
    }

    dependencies {
        classpath 'com.github.snail-Lin:logger:2.2.0'
    }
}
```

**6.更新项目依赖即可**

---


### 3.同步原作者的最新代码提交
简单概况就是两个步骤
**1.从原有远程仓库拉取最新提交，合并最新提交到本地仓库的分支**
**2.将合并后的最新代码推送到自己的远程仓库**

具体命令如下 （推荐使用SourceTree操作）：

```shell
#切换到需要更新的分支
git switch master
#从原作者仓库拉取最新提交
git pull origin
#将最新代码推送到自己的远程仓库
git push haidilao
#切换到修改代码的分支
git checkout my_dev
#合并原作者的最新代码
git merge master
#将合并后的代码推送到自己的远程仓库
git push haidilao
#更新项目依赖
npm i
```


