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

关键类讲解

* `SDWebImageDownloader`:负责维持图片的下载队列；
* `SDWebImageDownloaderOperation`:负责真正的图片下载请求；
* `SDImageCache`:负责图片的缓存；
* `SDWebImageManager`:是总的管理类，维护了一个`SDWebImageDownloader`实例和一个`SDImageCache`实例，是下载与缓存的桥梁;
* `SDWebImageDecoder`:负责图片的解压缩；
* `SDWebImagePrefetcher`:负责图片的预取；
* `UIImageView+WebCache`:和其他的扩展都是与用户直接打交道的。  

其中，最重要的三个类就是SDWebImageDownloader、SDImageCache、SDWebImageManager。

![](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/iOS_tupian/C5973F5A-5E97-4905-9F38-008F65EC799C.png?raw=true)  

* `UIImageView+WebCache`和`UIButton+WebCache`直接为表层的 UIKit框架提供接口  
* `SDWebImageManger`负责处理和协调`SDWebImageDownloader`和`SDWebImageCache`, 并与 `UIKit`层进行交互。  
* `SDWebImageDownloaderOperation`真正执行下载请求；最底层的两个类为高层抽象提供支持。  

## 三、SDWebImage 使用  

1.使用IImageView+WebCache category来加载UITableView中cell的图片  

```
[imageView sd_setImageWithURL:[NSURL URLWithString:@"http://img1.cache.netease.com/catchpic/5/51/5132C377F99EEEE927697E62C26DDFB1.jpg"] placeholderImage:[UIImage imageNamed:@"placeholder.png"]];
```  

2.使用Blocks,采用这个方案可以在网络图片加载过程中得知图片的下载进度和图片加载成功与否  

```
[imageView sd_setImageWithURL:[NSURL URLWithString:@"http://img1.cache.netease.com/catchpic/5/51/5132C377F99EEEE927697E62C26DDFB1.jpg"] placeholderImage:[UIImage imageNamed:@"placeholder.png"] completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
    //... completion code here ... 
 }];
```  

3.使用SDWebImageManager,SDWebImageManager为UIImageView+WebCache category的实现提供接口。  

```
SDWebImageManager *manager = [SDWebImageManager sharedManager] ;
[manager downloadImageWithURL:imageURL options:0 progress:^(NSInteger   receivedSize, NSInteger expectedSize) { 
      // progression tracking code
 }  completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType,   BOOL finished, NSURL *imageURL) { 
   if (image) { 
    // do something with image
   }
 }];
```  

4.加载图片还有使用SDWebImageDownloader和SDImageCache方式
5.key的来源 

```
// 利用Image的URL生成一个缓存时需要的key.
// 这里有两种情况,第一种是如果检测到cacheKeyFilter不为空时,利用cacheKeyFilter来处理URL生成一个key.
// 如果为空,那么直接返回URL的string内容,当做key.
- (NSString *)cacheKeyForURL:(NSURL *)url {
    if (self.cacheKeyFilter) {
        return self.cacheKeyFilter(url);
    }
    else {
        return [url absoluteString];
    }
}
```

## 三、SDWebImage 流程  

![](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/iOS_tupian/429AC748-CDD6-4962-B3E6-47F85C8239B7.png?raw=true)  
  


## 三、SDWebImage 解析  

1. 入口 setImageWithURL:placeholderImage:options: 会先把 placeholderImage 显示，然后 SDWebImageManager 根据 URL 开始处理图片。

2. 进入 SDWebImageManager-downloadWithURL:delegate:options:userInfo:，交给 SDImageCache 从缓存查找图片是否已经下载 queryDiskCacheForKey:delegate:userInfo:.

3. 先从内存图片缓存查找是否有图片，如果内存中已经有图片缓存，SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo: 到 SDWebImageManager。

4. SDWebImageManagerDelegate 回调 webImageManager:didFinishWithImage: 到 UIImageView+WebCache 等前端展示图片。

5. 如果内存缓存中没有，生成 NSInvocationOperation 添加到队列开始从硬盘查找图片是否已经缓存。

6. 根据 URLKey 在硬盘缓存目录下尝试读取图片文件。这一步是在 NSOperation 进行的操作，所以回主线程进行结果回调 notifyDelegate:。

7. 如果上一操作从硬盘读取到了图片，将图片添加到内存缓存中（如果空闲内存过小，会先清空内存缓存）。SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo:。进而回调展示图片。

8. 如果从硬盘缓存目录读取不到图片，说明所有缓存都不存在该图片，需要下载图片，回调 imageCache:didNotFindImageForKey:userInfo:。

9. 共享或重新生成一个下载器 SDWebImageDownloader 开始下载图片。

10. 图片下载由 NSURLConnection 来做，实现相关 delegate 来判断图片下载中、下载完成和下载失败。

11. connection:didReceiveData: 中利用 ImageIO 做了按图片下载进度加载效果。

12. connectionDidFinishLoading: 数据下载完成后交给 SDWebImageDecoder 做图片解码处理。

13. 图片解码处理在一个 NSOperationQueue 完成，不会拖慢主线程 UI。如果有需要对下载的图片进行二次处理，最好也在这里完成，效率会好很多。

14. 在主线程 notifyDelegateOnMainThreadWithInfo: 宣告解码完成，imageDecoder:didFinishDecodingImage:userInfo: 回调给 SDWebImageDownloader。

15. imageDownloader:didFinishWithImage: 回调给 SDWebImageManager 告知图片下载完成。

16. 通知所有的 downloadDelegates 下载完成，回调给需要的地方展示图片。

17. 将图片保存到 SDImageCache 中，内存缓存和硬盘缓存同时保存。写文件到硬盘也在以单独 NSInvocationOperation 完成，避免拖慢主线程。

18. SDImageCache 在初始化的时候会注册一些消息通知，在内存警告或退到后台的时候清理内存图片缓存，应用结束的时候清理过期图片。

19. SDWebImage 也提供了 UIButton+WebCache 和MKAnnotationView+WebCache，方便使用。

20. SDWebImagePrefetcher 可以预先下载图片，方便后续使用。







