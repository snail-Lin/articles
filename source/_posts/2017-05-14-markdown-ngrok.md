---
layout: post
title:  "搭建ngrok服务实现树莓派的内网穿透"
date:   2017-05-14
categories: RaspberryPi
excerpt: "ngrok是由go语言编写的一个开源的服务端，可以为内网机器创建TCP隧道，通过ngrok服务访问内网机器"
tag:
- ngrok
- Raspberry Pi
comments: true
---

### 前言
之前在树莓派上搭建了一个ari2服务用于Bt下载，并且搭建了一个Web服务器用于访问aria2的Web界面进行远程控制下载，但是在公司的时候无法访问树莓派的air2页面，正好最近租了一台VPS，就想到了搭建一个ngrok服务，将树莓派连接到VPS上的`ngrok`服务就可以实现内网穿透的功能。  
以前一直用的是`natapp`的内网穿透服务进行远程的`ssh`连接,natapp的服务也是基于ngrok的源码进行改造的，而且免费的隧道每隔一段时间都会更换一次远程端口，使用起来颇不方便。国内类似于这种服务还有`花生壳`等等，但基本上免费的都有流量限制或者端口限制。  
如果你有一台VPS，就可以自己搭建ngrok服务进行内网穿透，可以让家里的大多数设备在外网也能够访问。这几天查了很多关于ngrok的资料，但是大多数写的不是很详细，写这篇文章给有此需要的人参考。

### 准备工作
#### 安装依赖软件包
因为我的系统是Ubuntu，所以使用`apt-get`命令，CentOS则使用`yum`,主要是`golang`包，因为ngrok使用go语言编写的。
```shell
sudo apt-get install build-essential golang mercurial git
```
### 下载ngrok源码进行编译
#### github源码
现在ngrok最新的开源版本是`1.7`，`2.x`版本则是用于商业运营的，没有开源。
```
git clone https://github.com/inconshreveable/ngrok.git ngrok
cd ngrok
```
#### 替换SSL证书
因为要自己搭建服务，所以要使用自己的域名对应的SSL证书。可以使用`openssl`生成自签名的证书。即自己当`CA`签发证书。  
切换到源码目录，客户端的证书存放在`ngrok/assets/client/tls/`目录下，将生成的`CA`根证书放到此目录。服务端的证书路径为`ngrok/assets/server/tls/`,存放的是服务端的私钥和证书，证书将会被打包到编译生成的服务端程序和客户端程序。可以使用下面这一段`shell`脚本进行替换，记得使用自己的域名。  
只有使用自己生成的`CA`根证书打包出来的客户端程序才能连接对应的服务端，简单的说就是别人编译出的客户端无法连接自己服务器搭建的服务。
```shell
DOMAIN="yihanv.com"

openssl genrsa -out base.key 2048
openssl req -new -x509 -nodes -key base.key -days 10000 -subj "/CN=$NGROK_DOMAIN" -out base.pem
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=$NGROK_DOMAIN" -out server.csr
openssl x509 -req -in server.csr -CA base.pem -CAkey base.key -CAcreateserial -days 10000 -out server.crt

cp base.pem assets/client/tls/ngrokroot.crt

cp server.csr assets/server/tls/snakeoil.crt

cp server.key assets/server/tls/snakeoil.key
```
#### 编译程序
首先编译生成服务端，执行完毕后将生成`ngrokd`文件,路径为`ngrok/bin/ngrokd`，这个便是服务端程序。
```shell
sudo make release-server
```
因为树莓派的CPU是arm架构，所以生成客户端时要更改`GOARCH`参数,编译命令为:
```shell
sudo GOOS=linux GOARCH=arm make release-client
```
编译完成后会在`ngrok/bin`下生成一个`linux_arm`目录，目录下的`ngrok`便是客户端程序。  
因为我使用的是MacBook，所以再编译一个客户端用于MacBook，使用以下命令参数:
```shell
sudo GOOS=darwin GOARCH=amd64 make release-client
```
生成的客户端文件目录为`ngrok/bin/darwin_amd64/ngrok`。  
简单点的可以使用`make release-all`命令生成所有的服务端程序和客户端程序，编译生成的程序将会存放在以架构命名的文件夹下。
### 泛解析二级域名
ngrok的`TCP隧道`会使用`ngrok`服务的域名加上随机端口号，而`http`和`https`的映射隧道则会使用自定义的子域名。
#### 设置泛解析
我的域名的DNS托管在`DNSPod`上，进入管理界面，添加一个`*`的`主机记录`，记录类型为`A记录`,如图所示。记录值为自己VPS的ip地址
![](/assets/img/ngrok/dns.png)
### 开启ngrok服务
进入ngrok目录,执行以下命令ngrok服务便会在后台运行。
```shell
nohup sudo ./bin/ngrokd -tlsKey=server.key -tlsCrt=server.crt -domain="ng.yihanv.com" -httpAddr=":5600" -httpsAddr=":5601" &
```
服务端程序的每个参数的含义是:
* domain：服务的域名，可以是顶级域名或者二级域名
* httpAddr：远程http端口
* log：日志的输出方式，默认为none，`stdout`为直接在屏幕上输出
* log-level：日志输出等级，`DEBUG, INFO, WARNING, ERROR`，默认为`DEBUG`
* tlsCrt：服务端证书路径
* tlsKey：服务端私钥路径
* tunnelAddr：通信端口，默认为4443

### 客户端配置
复制编译好的对应的客户端到本地，创建一个`ngrok.cfg`配置文件，使用`yml`语言添加需要映射的端口和方式。
```
server_addr: yihanv.com:4443    //这里的域名要对应SSL证书域名
trust_host_root_certs: false
tunnels:
  web:
    proto:
      http: 80
  ssh:
    proto:
      tcp: 22
```
这里的配置文件的含义是将本地的`80`端口映射到`web.ng.yihanv.com`域名下，直接打开`http://web.ng.yihanv.com:5600`即可访问到本地的Web网页。  
将本地的`22`端口映射到服务器的`ng.yihanv.com`域名下，将会随机生成一个远程端口号,通过`ssh user@ng.yihanv.com -p "port"`即可进行远程`SSH`连接。
#### 运行客户端
将程序和配置文件放到同一个文件夹下，使用`chmod 755 ngrok`为程序添加执行权限。运行：
```shell
# 运行某些
./ngrok -config=ngrok.cfg start ssh
./ngrok -config=ngrok.cfg start ssh web
# 运行全部
./ngrok -config=ngrok.cfg start-all
```
若屏幕上显示`Tunnel Status online`则连接成功,如图所示：
![](/assets/img/ngrok/ngrok.png)
你可以可以打开本地的ngrok管理界面，查看经过此隧道的`http请求`等。默认地址为`127.0.0.1:4040`。截图如下：
![](/assets/img/ngrok/ngrokWeb.png)
这时候就可以使用指定的域名访问本地网页或者进行远程SSH了。  
如果有需要，可以将运行命令直接添加到开机启动中。



