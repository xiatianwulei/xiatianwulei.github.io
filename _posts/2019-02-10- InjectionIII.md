---
layout:     post
title:      InjectionIII 
subtitle:   
date:       2019-02-10
author:     夏天无泪
header-img: img/post-bg-ios9-web.jpg?raw=true
catalog: true
tags:
    - 知识扩展
    - 学习整理
---

# InjectionIII 使用 学习整理
**OS Switf 通过注入动态库的方式实现极速编译调试(InjectionIII、热重载、热编译)原理解析**

一、安装 与 使用

 1. 直接去App Store 搜
![-w1000](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15681674715020/15681676430581.jpg?raw=true)

1. 运行 InjectionIII 
![](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15681674715020/15681678306425.jpg?raw=true)

1. 在工程中添加代码

 在工程中
 ```
 - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions

 ```
 添加代码
    ```
    #if DEBUG
//or oc
[[NSBundle bundleWithPath:@"/Applications/InjectionIII.app/Contents/Resources/iOSInjection.bundle"] load];
//or switf
Bundle(path: "/Applications/InjectionIII.app/Contents/Resources/iOSInjection.bundle")?.load()
//for tvOS:
Bundle(path: "/Applications/InjectionIII.app/Contents/Resources/tvOSInjection.bundle")?.load()
//Or for macOS:
Bundle(path: "/Applications/InjectionIII.app/Contents/Resources/macOSInjection.bundle")?.load()
#endif
```

4. 运行工程
    1. 会弹出提示框选择工程文件路径，或者手动选择工程路径
    2. 控制台会出现如下
```
💉 Injection connected, watching /Users/diaohongrui/wandafilm-movie/**
```

eg:  当更改背景颜色 刷新 comend + s 保存 就会实现热更新

5. 运行原理
    InjectionIII 分为server 和 client部分，client部分在你的项目启动的时候会作为 bundle load 进去，server部分在Mac App那边，server 和 client 都会在后台发送和监听 Socket 消息，实现逻辑分别在 InjectionServer.mm 和 InjectionClient.mm 里的 runInBackground 方法里面。InjectionIII 会监听源代码文件的变化，如果文件被改动了，server 就会通过 Socket 通知 client 进行 rebuildClass 重新对该文件进行编译，打包成动态库，也就是 .dylib 文件。然后通过 dlopen 把动态库文件载入运行的 App 里，接下来 dlsym 会得到动态库的符号地址，然后就可以处理类的替换工作。当类的方法被替换后，我们就可以开始重新绘制界面了。整个过程无需重新编译和重载 App，使用动态库方式极速调试的目的就达成了。
    运行原理图
    ![](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/15681674715020/15681706419624.jpg?raw=true)
备注：此图作者戴铭

    学习原著 https://mp.weixin.qq.com/s/yWfpzzyhTwMf8eAtdO7AFg