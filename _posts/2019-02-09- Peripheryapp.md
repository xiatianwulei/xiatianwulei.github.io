---
layout:     post
title:      Peripheryapp 进阶
subtitle:   
date:       2019-02-09
author:     夏天无泪
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
   - 学习扩展
---

# Peripheryapp的使用 

*源地址：*
 https://github.com/peripheryapp/periphery#installation
作用：使用Periphery删除未使用的声明
## 使用和配置

1. 在你的Podfile文件里添加源 执行 pod install
```
pod 'Periphery'
```

2. 配置 xcode
链接： https://www.jianshu.com/p/bca8fc1ff3f0
安装XCode或者Command Line Tools for Xcode。Xcode可以从AppStore里下载安装，Command Line Tools for Xcode需要在终端中输入以下代码运行安装：
```
xcode-select --install  注意：这要翻墙 要不会失败
```
3. 安装Homebrew。将以下命令粘贴至终端
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

 移除Homebrew。将以下命令粘贴至终端（如果想要移除）
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"
```
1. 安装 Periphery releases
    1. brew tap peripheryapp/periphery
    2. brew cask install periphery
2. 使用 
    1. cd 到你的项目文件里 执行

    ```
    periphery scan
    ```
```
periphery help scan
```


学习作者 https://www.onswiftwings.com/unused-code-cleanup/