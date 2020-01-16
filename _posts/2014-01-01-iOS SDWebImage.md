---
layout:     post
title:      SDWebImage 介绍
subtitle:   
date:       2014-01-01
author:     夏天无泪
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
---  

# iOS SDWebImage  

> 在项目中使用SDWebImage来管理图片加载相关操作可以极大地提高开发效率，让我们更加专注于业务逻辑实现。版本为4.4.2版本。  

## 一、SDWebImage 功能  

>SDWebImage是个支持异步下载与缓存的UIImageView扩展  

项目主要提供了一下功能：  

1.提供了一个UIImageView的category用来加载网络图片并且对网络图片的缓存进行管理  
2.采用异步方式来下载网络图片  
3.采用异步方式，使用内存＋磁盘来缓存网络图片，拥有自动的缓存过期处理机制。  
4.支持GIF动画  
5.支持WebP格式  
6.同一个URL的网络图片不会被重复下载  
7.失效，虚假的URL不会被无限重试  
8.耗时操作都在子线程，确保不会阻塞主线程  
9.使用GCD和ARC  
10.支持Arm64  
11.支持后台图片解压缩处理  
12.项目支持的图片格式包括 PNG,JPEG,GIF,Webp等  

## 二 、SDWebImage组织架构  

![](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/iOS_tupian/792D48A1-4E2A-4E9F-8BE5-8628222AEC70.png?raw=true)


