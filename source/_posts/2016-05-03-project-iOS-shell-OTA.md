---
layout: post
title:  搭建Web服务器以无线方式安装企业内部应用
date:   2016-05-03
categories: Nginx
excerpt: "iOS支持以无线方式安装自定义的企业内部应用，可以不通过iTunes或者App Store"
tag:
- nginx
- 无线更新
- OTA
comments: true
---

  **目前海底捞门店的iPad点餐应用使用MDM进行日常版本的维护更新，在MDM的后台网页上传ipa文件并指定需要更新的门店，MDM服务器就可以远程推送更新包到各个门店的iPad上，自动安装升级版本，不需要门店运维人工进行升级操作。虽然MDM可以远程维护、安装程序，但是受限于店内的网络带宽（MDM供应商的服务器没有布置在内网，传输更新包需要通过外网）和总部MDM服务器的性能，整体安装时间比较长，通常需要5分钟左右才能下载更新完毕,而且有一小部分未受监管的iPad无法自动更新。** 
   
  **所以在需要紧急更新的时候，需要一个更快，并且更稳定的更新方式。利用每个店内的点餐服务器，在应用启动时检查服务端上最新的应用版本号，使用局域网下载最新的ipa包进行更新。这种方式简称为`局域网OAT`。于是乎在自己的电脑上搭建一个测试服务:** 

进行无线安装的步骤大致分为这几步:

* 搭建一个Web服务器，在iOS7.1之后，必须使用HTTPS
* 生成一个XML plist文件，保存的是企业分发的归档应用的信息
* 部署下载网页到Web服务器上，供应用跳转下载页面更新

## 每个步骤的详细过程
### 1.搭建Web服务器
简单的web服务器可以使用Apache，Tomcat，Nginx等等，只要能够部署网页的服务器均可。我选择的是Nginx服务器，占用内存少，稳定性高，搭建也简单。首先要自己测试，就先在我自己的Mac上搭建安装。安装Nginx服务器的过程如下：
#### 1.安装Homebrew
基本上有Mac的都安装过，可以忽略这一步
```shell
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
#### 2.使用Homebrew安装nginx
等待进度完成nginx服务器就已经安装好了，很简单。可以使用`brew info nginx`查看nginx的安装目录
```shell
brew tap homebrew/nginx
brew install nginx-full --with-rtmp-module
```
#### 3.生成SSL自签名证书
iOS7.1之后，Apple要求HTTPS的安全链接才能进行无线安装，SSL证书需要向国际公认的证书认证机构(CA)申请,而且证书签名每年的费用是几十美元甚至于上万美元不等，自己测试的话肯定不想这么麻烦，那就只能使用自签名的SSL证书，就是自己扮演CA机构，自己给自己的服务器颁发证书，不过使用这种证书的网站会被浏览器提示是一个`不安全的链接`。在iOS系统上可以将SSL根证书作为描述文件导入，即可信任此自签名证书，所以可以在网页上生成一个SSL根证书的链接，系统会自动识别证书，点击右上角安装并确认即可。  
使用OpenSSL生成证书，执行以下命令后目录下将有5个文件，分别为`server.crt`,`server.csr`,`server.key`,`ca.crt`,`ca.key`
```shell
#生成服务器私钥
sudo openssl genrsa -out server.key 1024
#生成签署申请,Common Name使用自己的主机ip或域名
sudo openssl req -new -key server.key -out server.csr
#生成CA私钥
sudo openssl genrsa  -out ca.key 1024
#用CA的私钥产生CA的签署证书
sudo openssl req  -new -x509 -days 365 -key ca.key -out ca.crt
#创建demoCA目录，在demoCA目录下创建index.txt，serial文件，newcerts文件夹
mkdir demoCA
cd demoCA
touch index.txt
nano serial
#输入02后并保存文件
mkdir newcerts
#向自己的CA机构申请证书，签名过程需要CA的证书和私钥，生成一个带有CA签名的证书
sudo openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key
```
### 4.配置nginx服务器
使用`brew info nginx`命令找到nginx的配置文件  
![](/assets/img/ota/nginx-info.png)
如果没有更改的话，默认是在`/usr/local/etc/nginx/nginx.conf`，我们需要在配置文件中添加https的配置。将`server.crt`和`server.key`复制到nginx目录下
```shell
cp server.crt server.key /usr/local/etc/nginx/
#编辑nginx.conf文件
nano /usr/local/etc/nginx/nginx.conf
```
在配置文件的server下添加
```
server {
    ...
    ssl on;
    ssl_certificate     server.crt;
    ssl_certificate_key server.key;
}
```
**设定服务器的MIME类型，Nginx服务器的默认MIME保存在mime.types文件中，将以下两项添加到文件内容**

* *application/octet-stream ipa*

* *text/xml plist*

### 5.创建plist文件
创建一个XML plist文件，设备将根据文件内容在Web服务器上查找，下载和安装应用。使用Xcode或者sublime直接创建plist文件均可，以下是Apple提供的示例文件:
```
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- array of downloads. -->
  <key>items</key>
  <array>
   <dict>
    <!-- an array of assets to download -->
     <key>assets</key>
      <array>
       <!-- software-package: the ipa to install. -->
        <dict>
         <!-- required. the asset kind. -->
          <key>kind</key>
          <string>software-package</string>
          <!-- optional. md5 every n bytes. will restart a chunk if md5 fails. -->
          <key>md5-size</key>
          <integer>10485760</integer>
          <!-- optional. array of md5 hashes for each "md5-size" sized chunk. -->
          <key>md5s</key>
          <array>
            <string>41fa64bb7a7cae5a46bfb45821ac8bba</string>
            <string>51fa64bb7a7cae5a46bfb45821ac8bba</string>
          </array>
          <!-- required. the URL of the file to download. -->
          <key>url</key>
          <string>https://www.example.com/apps/foo.ipa</string>
        </dict>
        <!-- display-image: the icon to display during download.-->
        <dict>
         <key>kind</key>
         <string>display-image</string>
         <!-- optional. indicates if icon needs shine effect applied. -->
         <key>needs-shine</key>
         <true/>
         <key>url</key>
         <string>https://www.example.com/image.57x57.png</string>
        </dict>
        <!-- full-size-image: the large 512x512 icon used by iTunes. -->
        <dict>
         <key>kind</key>
         <string>full-size-image</string>
         <!-- optional. one md5 hash for the entire file. -->
         <key>md5</key>
         <string>61fa64bb7a7cae5a46bfb45821ac8bba</string>
         <key>needs-shine</key>
         <true/>
         <key>url</key><string>https://www.example.com/image.512x512.jpg</string>
        </dict>
      </array>
<key>metadata</key>
      <dict>
       <!-- required -->
       <key>bundle-identifier</key>
       <string>com.example.fooapp</string>
       <!-- optional (software only) -->
       <key>bundle-version</key>
       <string>1.0</string>
       <!-- required. the download kind. -->
       <key>kind</key>
       <string>software</string>
       <!-- optional. displayed during download; typically company name -->
       <key>subtitle</key>
       <string>Apple</string>
       <!-- required. the title to display during the download. -->
       <key>title</key>
       <string>Example Corporate App</string>
      </dict>
    </dict>
  </array>
</dict>
</plist>
```
必填项一共有:

* URL：应用 (.ipa) 文件的完全限定 HTTPS URL
* display-image：57 x 57 像素的 PNG 图像，在下载和安装过程中显示。指定图像的完全限定 URL
* full-size-image：512 x 512 像素的 PNG 图像，表示 iTunes 中相应的应用
* bundle-identifier：应用的包标识符，与 Xcode 项目中指定的完全一样
* bundle-version：应用的包版本，在 Xcode 项目中指定
* title：下载和安装过程中显示的应用的名称

### 6.创建HTML文件
我们需要一个可以安装根证书和下载App的网页，简单的测试网页上只需要两个链接即可。App的下载链接的格式为`itms-services://?action=download-manifest&url=(plist文件地址)`，设备会在载入plist文件后，解析文件中的信息校验后将下载.ipad文件，虽然URL的协议部分是`itms-services`，但iTunes Store并不参加下载过程。
```html
<!DOCTYPE html>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<html>
<head>
<title>海底捞点餐App更新</title>AAA
</head>
<body>

<a href="/ca.crt" download="ssl">
<p>ssl证书下载</p>
</a>

<a href="itms-services://?action=download-manifest&url=https://本机的ip地址/manifest.plist">
点击下载App更新
</a>

</body>
</html>
```

### 7.部署网页并启动Nginx服务器
将`HTML文件`、`ca.crt`、`ipa文件`、`plist文件`，放到同一个目录下，更改nginx的配置文件,这里我将网站根目录文件夹设置为本地根目录下的nginx文件夹。
 ```
 server {
    ...
    location / {
            root   /nginx;
            index  index.html index.htm;
    }
 }   
 ```

启动nginx服务器,报错的话检查一下是否把crt和key文件复制到了nginx目录下。
``` 
sudo nginx 
```

### 8.iPad安装根证书
如果以上的步骤都成功完成的话，网页就可以通过地址`https://你的ip地址`进行访问了。使用iPad访问地址，第一次将提示这是一个不安全的链接，点击继续即可，先安装ssl证书，系统将会自动识别文件类型安装，点击安装App:  
![](/assets/img/ota/haidilao-download.png)

如果显示截图中的提示，就表示已经可以通过自己搭建的网页安装App了。

## 集成
应用内部必须有一个检测最新版本的组件，门店Pad服务器上创建一个用于上传ipa文件的后台网页，并自动解包ipa并检测其中的plist文件提取版本号，点餐应用将检测服务端的版本号跳转页面进行强制更新。其实MDM推送的实现方式也是基于此方法，但是由于MDM服务是其它供应商提供的，且服务器部署在外网，所以只能`曲线救国`了。  
具体流程可以看流程图:  
![](/assets/img/ota/flow.png)

### 安装出现的一些问题解决方案

* 如果出现`无法连接`  
先检查plist文件中的下载链接是否正常  
虽然可以通过内网Web服务器下载ipa文件，但是设备也必须能够访问`https://ppq.apple.com`网站，这是Apple用于检测校验描述文件内容的网站，设备将会自动联系。

* 安装失败  
确认一下ipa包是否是In-House类型，否则设备无法安装

**2017.3.31更新:**  
**iOS 10.3系统通过此方法进行更新，HTTPS将无法使用自签名的SSL证书，只能使用正规CA签名的证书，否则将提示无法连接**


