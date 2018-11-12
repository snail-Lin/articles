---
layout: post
title:  "使用nginx搭建视频直播服务器"
date:   2016-02-01
categories: Nginx
excerpt: "nginx是一个轻量级的Web服务器，使用nginx可以很快搭建出一个可以直接使用的直播服务器"
tag:
- nginx 
- rtmp
- 树莓派
comments: true
---

## 安装nginx和rtmp模块 
在Mac上安装完nginx成功推流后，便尝试在树莓派上安装nginx服务器，毕竟树莓派可以24小时不关机，先记录一下树莓派安装nginx的过程
### 安装依赖
首先使用ssh连接到树莓派，使用以下命令安装依赖包

    sudo apt-get install build-essential libpcre3 libpcre3-dev libssl-dev
    
### 下载服务器包和rtmp插件
    wget http://nginx.org/download/nginx-1.11.8.tar.gz
    wget https://github.com/arut/nginx-rtmp-module/archive/master.zip
    解压缩
    tar -zxvf nginx-1.11.8.tar.gz
    unzip master.zip
    cd nginx-1.11.8
    安装
    ./configure --with-http_ssl_module --add-module=../nginx-rtmp-module-master
    make
    sudo make install
### 修改nginx配置文件
进入目录

    cd /usr/local/nginx/conf/nginx.conf

使用vim或者nano编辑配置文件
    
    sudo nano /usr/local/nginx/conf/nginx.conf
    
在http节点后加上rtmp配置
    
    rtmp {
    server {
        listen 1935;
        chunk_size 4096;
        application live {
            live on;
            record off;
           }
        }
    }

开始启动nginx
    
    sudo /usr/local/nginx/sbin/nginx
    
停止的命令是
    
    sudo /usr/local/nginx/sbin/nginx -s stop
    
这时候，直播服务器已经搭建完成，想要进行直播测试，简单的可以使用github上开源的LiveVideo，在手机上安装好后，连接搭建好的服务器进行推流，输入地址
    
    rtmp://192.168.31.141:1935/rtmplive/home
    这里使用的是我当前电脑的ip
在需要播放直播的客户端，同样使用这个地址，便可以看到实时的画面，经过测试，延迟基本在10秒左右，延迟有点大，可能还是树莓派的处理器性能不足的问题

### 播放器
Mac上可以播放rtmp视频的软件有VLC等，手机上可以使用哔哩哔哩开源的ijkplaySDK进行播放。




