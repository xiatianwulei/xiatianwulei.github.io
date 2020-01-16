---
layout:     post
title:      SDWebImage 介绍
subtitle:   
date:       2014-01-01
author:     夏天无泪
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 学习整理
---  

# iOS SDWebImage  

> 在项目中使用SDWebImage来管理图片加载相关操作可以极大地提高开发效率，让我们更加专注于业务逻辑实现。版本为4.4.2版本。  

### 一、SDWebImage 功能  

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

### 二 、SDWebImage组织架构  

![](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/iOS_tupian/792D48A1-4E2A-4E9F-8BE5-8628222AEC70.png?raw=true)  

关键类讲解

* `SDWebImageDownloader`:负责维持图片的下载队列；
* `SDWebImageDownloaderOperation`:负责真正的图片下载请求；
* `SDImageCache`:负责图片的缓存；
* `SDWebImageManager`:是总的管理类，维护了一个`SDWebImageDownloader`实例和一个`SDImageCache`实例，是下载与缓存的桥梁;
* `SDWebImageDecoder`:负责图片的解压缩；
* `SDWebImagePrefetcher`:负责图片的预取；
* `UIImageView+WebCache`:和其他的扩展都是与用户直接打交道的。  

其中，最重要的三个类就是`SDWebImageDownloader`、`SDImageCache`、`SDWebImageManager`。

![](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/iOS_tupian/C5973F5A-5E97-4905-9F38-008F65EC799C.png?raw=true)  

* `UIImageView+WebCache`和`UIButton+WebCache`直接为表层的 UIKit框架提供接口  
* `SDWebImageManger`负责处理和协调`SDWebImageDownloader`和`SDWebImageCache`, 并与 `UIKit`层进行交互。  
* `SDWebImageDownloaderOperation`真正执行下载请求；最底层的两个类为高层抽象提供支持。  


### 三、SDWebImage 功能类分析  

**我们按照上图中从上到下执行的流程来研究各个类**  

####  1.UIImageView+WebCache   

这里只用`UIImageView+WebCache`来举个例子，其他的扩展类似。

**使用场景：已知图片的url地址，下载图片并设置到UIImageView上。**  

```
- (void)sd_setImageWithURL:(nullable NSURL *)url;
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder;
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options;
- (void)sd_setImageWithURL:(nullable NSURL *)url completed:(nullable SDExternalCompletionBlock)completedBlock;
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder completed:(nullable SDExternalCompletionBlock)completedBlock;
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options completed:(nullable SDExternalCompletionBlock)completedBlock;
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock completed:(nullable SDExternalCompletionBlock)completedBlock;
- (void)sd_setImageWithPreviousCachedImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock completed:(nullable SDExternalCompletionBlock)completedBlock;
```

这些接口最终都会调用  

```
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options
                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                 completed:(nullable SDExternalCompletionBlock)completedBlock;

```  

新版本还给UIView增加了分类，即`UIView+WebCache`，最终上述方法会走到下面的方法去具体操作，比如下载图片等。  

```  

- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                      operationKey:(nullable NSString *)operationKey
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                         completed:(nullable SDExternalCompletionBlock)completedBlock
                           context:(nullable NSDictionary<NSString *, id> *)context;
                           
```

**接下来对该方法进行解析**  

* 第一步：取消当前正在进行的异步下载，确保每个 UIImageView 对象中永远只存在一个 operation，当前只允许一个图片网络请求，该 operation 负责从缓存中获取 image 或者是重新下载 image。具体执行代码是：

```
NSString *validOperationKey = operationKey ?: NSStringFromClass([self class]);
// 取消先前下载的任务
[self sd_cancelImageLoadOperationWithKey:validOperationKey];
... // 下载图片操作
// 将生成的加载操作赋值给UIView的自定义属性
[self sd_setImageLoadOperation:operation forKey:validOperationKey];

```

上述方法定义在`UIView+WebCacheOperation`类中  

```
- (void)sd_setImageLoadOperation:(nullable id<SDWebImageOperation>)operation forKey:(nullable NSString *)key {
    if (key) {
        // 如果之前已经有过该图片的下载操作,则取消之前的图片下载操作
        [self sd_cancelImageLoadOperationWithKey:key];
        if (operation) {
            SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
            @synchronized (self) {
                [operationDictionary setObject:operation forKey:key];
            }
        }
    }
}

- (void)sd_cancelImageLoadOperationWithKey:(nullable NSString *)key {
    if (key) {
        // Cancel in progress downloader from queue
        SDOperationsDictionary *operationDictionary = [self sd_operationDictionary]; // 获取添加在UIView的自定义属性
        id<SDWebImageOperation> operation;
        
        @synchronized (self) {
            operation = [operationDictionary objectForKey:key];
        }
        if (operation) {
            // 实现了SDWebImageOperation的协议
            if ([operation conformsToProtocol:@protocol(SDWebImageOperation)]) {
                [operation cancel];
            }
            @synchronized (self) {
                [operationDictionary removeObjectForKey:key];
            }
        }
    }
}
```

实际上，所有的操作都是由一个实际上，所有的操作都是由一个`operationDictionary`字典维护的,执行新的操作之前，`cancel`所有的`operation`。

* 第二步:占位图策略  
   作为图片下载完成之前的替代图片。dispatch_main_async_safe是一个宏，保证在主线程安全执行.  
   
   ```
  if (!(options & SDWebImageDelayPlaceholder)) {
    dispatch_main_async_safe(^{
        // 设置占位图
        [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock];
    });
}
```

* 第三步：判断url是否合法  
如果url合法，则进行图片下载操作，否则直接block回调失败  

```
if (url) {
    // 下载图片操作
} else {
    dispatch_main_async_safe(^{
#if SD_UIKIT
        [self sd_removeActivityIndicator];
#endif
        if (completedBlock) {
            NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
            completedBlock(nil, error, SDImageCacheTypeNone, url);
        }
    });
}

```  

* 第四步 下载图片操作
下载图片的操作是由`SDWebImageManager`完成的，它是一个单例  

```
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock;

```  

下载完成之后刷新UIImageView的图片。  

```
// 根据枚举类型，判断是否需要设置图片
shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                          (!image && !(options & SDWebImageDelayPlaceholder)));
SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
    if (!sself) { return; }
    if (!shouldNotSetImage) {
        [sself sd_setNeedsLayout];  // 设置图片
    }
    if (completedBlock && shouldCallCompletedBlock) {
        completedBlock(image, error, cacheType, url);
    }
};

if (shouldNotSetImage) {    // 不要自动设置图片,则调用block传入image对象
    dispatch_main_async_safe(callCompletedBlockClojure);
    return;
}

// 设置图片操作
dispatch_main_async_safe(^{
#if SD_UIKIT || SD_MAC
    [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
#else
    [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock];
#endif
    callCompletedBlockClojure();
});
```  

最后，把返回的`id operation`添加到`operationDictionary`中，方便后续的cancel。

```
// 将生成的加载操作赋值给UIView的自定义属性
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
```

####  2. SDWebImageManager  

**在`UIImageView+WebCache`背后，用于处理异步下载和图片缓存的类，当然你也可以直接使用 `SDWebImageManager` 的方法 来直接下载图片  **

```
- (nullable id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                              options:(SDWebImageOptions)options
                                             progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                            completed:(nullable SDInternalCompletionBlock)completedBlock;
```

`SDWebImageManager.h`首先定义了一些枚举类型的SDWebImageOptions。  
然后，声明了四个block：  

```
//操作完成的回调，被上层的扩展调用。
typedef void(^SDWebImageCompletionBlock)(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL);

//被SDWebImageManager调用。如果使用了SDWebImageProgressiveDownload标记，这个block可能会被重复调用，直到图片完全下载结束，finished=true,再最后调用一次这个block。
typedef void(^SDWebImageCompletionWithFinishedBlock)(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL);

//SDWebImageManager每次把URL转换为cache key的时候调用，可以删除一些image URL中的动态部分。
typedef NSString *(^SDWebImageCacheKeyFilterBlock)(NSURL *url);

typedef NSData * _Nullable(^SDWebImageCacheSerializerBlock)(UIImage * _Nonnull image, NSData * _Nullable data, NSURL * _Nullable imageURL);
```

定义了`SDWebImageManagerDelegate`协议：  

```
@protocol SDWebImageManagerDelegate 
 
@optional
// 控制在cache中没有找到image时 是否应该去下载。
- (BOOL)imageManager:(SDWebImageManager *)imageManager shouldDownloadImageForURL:(NSURL *)imageURL;

// 在下载之后，缓存之前转换图片。在全局队列中操作，不阻塞主线程
- (UIImage *)imageManager:(SDWebImageManager *)imageManager transformDownloadedImage:(UIImage *)image withURL:(NSURL *)imageURL;

@end
```

`SDWebImageManager`是单例使用的，分别维护了一个`SDImageCache`实例和一个`SDWebImageDownloader`实例。 对象方法分别是：  

```
//初始化SDWebImageManager单例，在init方法中已经初始化了cache单例和downloader单例。
- (instancetype)initWithCache:(SDImageCache *)cache downloader:(SDWebImageDownloader *)downloader;
//下载图片
- (id )downloadImageWithURL:(NSURL *)url
                    options:(SDWebImageOptions)options
                   progress:(SDWebImageDownloaderProgressBlock)progressBlock
                  completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;
//缓存给定URL的图片
- (void)saveImageToCache:(UIImage *)image forURL:(NSURL *)url;
//取消当前所有的操作
- (void)cancelAll;
//监测当前是否有进行中的操作
- (BOOL)isRunning;
//监测图片是否在缓存中， 先在memory cache里面找  再到disk cache里面找
- (BOOL)cachedImageExistsForURL:(NSURL *)url;
//监测图片是否缓存在disk里
- (BOOL)diskImageExistsForURL:(NSURL *)url;
//监测图片是否在缓存中,监测结束后调用completionBlock
- (void)cachedImageExistsForURL:(NSURL *)url
                     completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;
//监测图片是否缓存在disk里,监测结束后调用completionBlock
- (void)diskImageExistsForURL:(NSURL *)url
                   completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;
//返回给定URL的cache key
- (NSString *)cacheKeyForURL:(NSURL *)url;
```

**我们主要研究**  

```
- (nullable id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                              options:(SDWebImageOptions)options
                                             progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                            completed:(nullable SDInternalCompletionBlock)completedBlock;
```

##### 1. 首先，监测url 的合法性：  

```
if ([url isKindOfClass:NSString.class]) {
    url = [NSURL URLWithString:(NSString *)url];
}
// Prevents app crashing on argument type error like sending NSNull instead of NSURL
if (![url isKindOfClass:NSURL.class]) {
    url = nil;
}
```  

第一个判断条件是防止很多用户直接传递NSString作为NSURL导致的错误，第二个判断条件防止crash。  

##### 2.集合failedURLs保存之前失败的urls，如果url为空或者url之前失败过且不采用重试策略，直接调用completedBlock返回错误。  

```
BOOL isFailedUrl = NO;
if (url) {  // 判断url是否是失败过的url
    LOCK(self.failedURLsLock);
    isFailedUrl = [self.failedURLs containsObject:url];
    UNLOCK(self.failedURLsLock);
}
// 如果url为空或者url下载失败并且设置了不再重试
if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
    [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil] url:url];
    return operation;
}
```  

##### 3.保存操作  

```
LOCK(self.runningOperationsLock);
[self.runningOperations addObject:operation];
UNLOCK(self.runningOperationsLock);
```   

runningOperations是一个可变数组，保存所有的operation，主要用来监测是否有operation在执行，即判断running 状态。  

##### 4.查找缓存  

SDWebImageManager会首先在memory以及disk的cache中查找是否下载过相同的照片，即调用imageCache的下面方法  

```
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options done:(nullable SDCacheQueryCompletedBlock)doneBlock;
```  

如果操作取消，则直接返回

```
__strong __typeof(weakOperation) strongOperation = weakOperation;
if (!strongOperation || strongOperation.isCancelled) {  // operation取消,那么将下载任务从下载队列中直接移除
    [self safelyRemoveOperationFromRunning:strongOperation];
    return;
}
```  

如果没有在缓存中找到图片，或者不管是否找到图片，只要`operation`有`SDWebImageRefreshCached`标记，那么若`SDWebImageManagerDelegate`的`shouldDownloadImageForURL`方法返回true，即允许下载时，都使用 imageDownloader 的下载方法

```
- (id )downloadImageWithURL:(NSURL *)url options:(SDWebImageDownloaderOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageDownloaderCompletedBlock)completedBlock
```  

如果下载有错误，直接调用`completedBlock`返回错误，并且视情况将url添加到failedURLs里面；  

```
dispatch_main_sync_safe(^{
    if (strongOperation && !strongOperation.isCancelled) {
        completedBlock(nil, error, SDImageCacheTypeNone, finished, url);
    }
});
 
if (error.code != NSURLErrorNotConnectedToInternet
 && error.code != NSURLErrorCancelled
 && error.code != NSURLErrorTimedOut
 && error.code != NSURLErrorInternationalRoamingOff
 && error.code != NSURLErrorDataNotAllowed
 && error.code != NSURLErrorCannotFindHost
 && error.code != NSURLErrorCannotConnectToHost) {
      @synchronized (self.failedURLs) {
          [self.failedURLs addObject:url];
      }
}
```  

如果下载成功，若支持失败重试，将url从failURLs里删除  

```
if ((options & SDWebImageRetryFailed)) {
    @synchronized (self.failedURLs) {
         [self.failedURLs removeObject:url];
    }
}
```  

如果delegate实现了，`imageManager:transformDownloadedImage:withURL:`方法，图片在缓存之前，需要做转换（在全局队列中调用，不阻塞主线程）。转化成功切下载全部结束，图片存入缓存，调用completedBlock回调，第一个参数是转换后的image。

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
    UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];
 
    if (transformedImage && finished) {
        BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
        //将图片缓存起来
        [self.imageCache storeImage:transformedImage recalculateFromImage:imageWasTransformed imageData:(imageWasTransformed ? nil : data) forKey:key toDisk:cacheOnDisk];
    }
    dispatch_main_sync_safe(^{
        if (strongOperation && !strongOperation.isCancelled) {
            completedBlock(transformedImage, nil, SDImageCacheTypeNone, finished, url);
        }
    });
});
```  

否则，直接存入缓存，调用`completedBlock`回调，第一个参数是下载的原始image。  

```
if (downloadedImage && finished) {
    [self.imageCache storeImage:downloadedImage recalculateFromImage:NO imageData:data forKey:key toDisk:cacheOnDisk];
}
 
dispatch_main_sync_safe(^{
    if (strongOperation && !strongOperation.isCancelled) {
        completedBlock(downloadedImage, nil, SDImageCacheTypeNone, finished, url);
    }
});
```  

存入缓存都是调用imageCache的下面方法  

```
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk
```  

如果没有在缓存找到图片，且不允许下载，直接调用completedBlock，第一个参数为nil。  

```
dispatch_main_sync_safe(^{
    __strong __typeof(weakOperation) strongOperation = weakOperation;
    if (strongOperation && !weakOperation.isCancelled) {//为啥这里用weakOperation TODO
        completedBlock(nil, nil, SDImageCacheTypeNone, YES, url);
    }
});
```  

最后都要将这个`operation从runningOperations`里删除。

```
@synchronized (self.runningOperations) {
    [self.runningOperations removeObject:operation];
 }
```

####  3. SDWebImageCombinedOperation   

```
@interface SDWebImageCombinedOperation : NSObject 
 
@property (assign, nonatomic, getter = isCancelled) BOOL cancelled;
@property (copy, nonatomic) SDWebImageNoParamsBlock cancelBlock;
@property (strong, nonatomic) NSOperation *cacheOperation;
 
@end
```  

是一个遵循SDWebImageOperation协议的NSObject子类。  

```
@protocol SDWebImageOperation 
 
- (void)cancel;
 
@end
```

在里面封装一个NSOperation，这么做的目的应该是为了使代码更简洁。因为下载操作需要查询缓存的operation和实际下载的operation，这个类的cancel方法可以同时cancel两个operation，同时还可以维护一个状态cancelled。

### 四、SDWebImage 使用  

1.使用`IImageView+WebCache` category来加载UITableView中cell的图片  

```
[imageView sd_setImageWithURL:[NSURL URLWithString:@"http://img1.cache.netease.com/catchpic/5/51/5132C377F99EEEE927697E62C26DDFB1.jpg"] placeholderImage:[UIImage imageNamed:@"placeholder.png"]];
```  

2.使用Blocks,采用这个方案可以在网络图片加载过程中得知图片的下载进度和图片加载成功与否  

```
[imageView sd_setImageWithURL:[NSURL URLWithString:@"http://img1.cache.netease.com/catchpic/5/51/5132C377F99EEEE927697E62C26DDFB1.jpg"] placeholderImage:[UIImage imageNamed:@"placeholder.png"] completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
    //... completion code here ... 
 }];
```  

3.使用`SDWebImageManager`,`SDWebImageManager`为`UIImageView+WebCache` category的实现提供接口。  

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

4.加载图片还有使用`SDWebImageDownloader`和`SDImageCache`方式  

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

### 五、SDWebImage 流程  

![](https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/iOS_tupian/429AC748-CDD6-4962-B3E6-47F85C8239B7.png?raw=true)  
  


### 六、SDWebImage 解析  

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

**补充：**    

当你调用setImageWithURL:方法的时候，他会自动去给你干这么多事，当你需要在某一具体时刻做事情的时候，你可以覆盖这些方法。比如在下载某个图片的过程中要响应一个事件，就覆盖这个方法：  

```
//覆盖方法，指哪打哪，这个方法是下载imagePath2的时候响应
    SDWebImageManager *manager = [SDWebImageManager sharedManager];
     
    [manager downloadImageWithURL:imagePath2 options:SDWebImageRetryFailed progress:^(NSInteger receivedSize, NSInteger expectedSize) {
         
        NSLog(@"显示当前进度");
         
    } completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
         
        NSLog(@"下载完成");
    }];
```







