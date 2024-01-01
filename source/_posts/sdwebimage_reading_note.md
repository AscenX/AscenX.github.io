---
title: 读SDWebImage笔记
date: 2016-03-10 22:07:06
tags:
- iOS
- Objective-C
categories:
- 学习笔记
---

### 介绍

github地址：

https://github.com/rs/SDWebImage

### 简介
一个异步图片下载及缓存的库

<!-- more -->

特性：

- 一个扩展UIImageView分类的库，支持加载网络图片并缓存图片
- 异步图片下载器
- 异步图片缓存和自动图片有效期管理
- 支持GIF动态图片
- 支持WebP
- 背景图片减压
- 保证同一个URL不会再次下载
- 保证无效的URL不会重新加载
- 保证主线程不会死锁
- 性能优越
- 使用GCD和ARC
- 支持ARM64位处理器
- 用法不说了 github上有，挺简单的

### 原理
只要有图片的url，就能下载到图片，使用SDWebImage的好处就是缓存机制，每次取图片先判断是否在内存中，再到缓存中查找，找到了直接加载，在缓存中找不到才重新下载，url也会记录，是否是失效的url，是则不会再尝试。下载到的图片会缓存，用于下次可以直接加载。图片下载，解码，转换都异步进行，不会阻塞主线程。

### 类的作用
- SDImageCache
设置缓存的类型，方式，路径等

- SDWebImageCompat
兼容类，定义了很多宏和一个转换图片的方法

- SDWebImageDecoder
解码器，让图片色彩转换（涉及到color space）

- SDWebImageDownloader
下载器，设置下载相关，要用到SDWebImageDownloaderOperation

- SDWebImageDownloaderOperation
下载器的操作

- SDWebImageManager
管理图片下载，取消操作，判断url是否已缓存等

- SDWebImageOperation
图片操作，后面很多类都要用到

- SDWebImagePrefetcher
预抓取器，预先下载urls中的图片

- UIButton+WebCache
按钮图片的缓存

- UIImage+GIF
缓存gif

- NSData+ImageContentType
判断图片的类型,png/jpeg/gif/webp

- UIImage+MultiFormat
缓存多种格式的图片，要用到NSData+ImageContentType的判断图片类型方法和UIImage+GIF的判断是否为gif图片方法，以及ImageIO里面的方法

- UIImageView+HighlightedWebCache
缓存高亮图片

- UIImageView+WebCache
主要用到这个，加载及缓存UIImageView的图片

- UIView+WebCacheOperation
缓存的操作，有缓存，取消操作，移除缓存

### 源码解析
先讲一些比较边缘的方法

#### 0.SDWebImageOperation
图片操作，只有头文件，定义了协议SDWebImageOperation，里面也只有取消方法
这个类后面很多类都要用到。

```
@protocol SDWebImageOperation <NSObject>
- (void)cancel;
@end
```

#### 1.NSData+ImageContentType
这个文件是NSData的分类，只有一个方法，传入图片数据，根据图片的头标识来确定图片的类型。头标识都不一样，只需获取文件头字节，对比十六进制信息，判断即可。

图片文件    头标识	十六进制头字节
jpeg/jpg	FFD8	0xFF
png	8950	0x89
gif	4749	0x47
tiff	4D4D / 4949	0x49/0x4D
Webp格式开头是0x52，但是还有可能是其他类型文件，所以要识别前缀为
52 49 46 46 对应 RIFF
后缀 57 45 42 50 对应 WEBP，符合这些条件的才是webp图片文件

```
+ (NSString *)sd_contentTypeForImageData:(NSData *)data {
    uint8_t c;
    [data getBytes:&c length:1];
    switch (c) {
        case 0xFF:
            return @"image/jpeg";
        case 0x89:
            return @"image/png";
        case 0x47:
            return @"image/gif";
        case 0x49:
        case 0x4D:
            return @"image/tiff";
        case 0x52:
            // R as RIFF for WEBP
            if ([data length] < 12) {
                return nil;
            }
 
            NSString *testString = [[NSString alloc] initWithData:[data subdataWithRange:NSMakeRange(0, 12)] encoding:NSASCIIStringEncoding];
            if ([testString hasPrefix:@"RIFF"] && [testString hasSuffix:@"WEBP"]) {
                return @"image/webp";
            }
 
            return nil;
    }
    return nil;
}
```

#### 2.SDWebImageCompat
兼容类，这个类定义了很多宏还有一个伸缩图片的方法，宏就不说了
这个方法定义成C语言式的内联方法
核心代码如下，传入key和图片，如果key中出现@2x就设定scale为2.0,出现@3x就设定scale为3.0，然后伸缩图片

```
CGFloat scale = [UIScreen mainScreen].scale;
if (key.length >= 8) {
    NSRange range = [key rangeOfString:@"@2x."];
    if (range.location != NSNotFound) {
        scale = 2.0;
    }
 
    range = [key rangeOfString:@"@3x."];
    if (range.location != NSNotFound) {
        scale = 3.0;
    }
}
 
UIImage *scaledImage = [[UIImage alloc] initWithCGImage:image.CGImage scale:scale orientation:image.imageOrientation];
image = scaledImage;
```

#### 3.SDWebImageDecoder
这个是解码器类，只定义了一个解码方法，传入图片，返回的也是图片
CGImageRef是一个指针类型。typedef struct CGImage *CGImageRef;
获取传入图片的alpha信息，然后判断是否符合苹果定义的CGImageAlphaInfo，如果是就返回原图片

```
CGImageRef imageRef = image.CGImage;
 
CGImageAlphaInfo alpha = CGImageGetAlphaInfo(imageRef);
BOOL anyAlpha = (alpha == kCGImageAlphaFirst ||
                 alpha == kCGImageAlphaLast ||
                 alpha == kCGImageAlphaPremultipliedFirst ||
                 alpha == kCGImageAlphaPremultipliedLast);
 
if (anyAlpha) { 
    return image; 
}
```

然后获取图片的宽高和color space（指定颜色值如何解释），判断color space是否支持，不支持就转换为支持的模式（RGB)，再用图形上下文根据获得的信息画出来，释放掉创建的CG指针再返回图片

```
size_t width = CGImageGetWidth(imageRef);
size_t height = CGImageGetHeight(imageRef);
 
// current
CGColorSpaceModel imageColorSpaceModel = CGColorSpaceGetModel(CGImageGetColorSpace(imageRef));
CGColorSpaceRef colorspaceRef = CGImageGetColorSpace(imageRef);
 
bool unsupportedColorSpace = (imageColorSpaceModel == 0 || imageColorSpaceModel == -1 || imageColorSpaceModel == kCGColorSpaceModelCMYK || imageColorSpaceModel == kCGColorSpaceModelIndexed);
if (unsupportedColorSpace)
    colorspaceRef = CGColorSpaceCreateDeviceRGB();
 
CGContextRef context = CGBitmapContextCreate(NULL, width,
                                             height,
                                             CGImageGetBitsPerComponent(imageRef),
                                             0,
                                             colorspaceRef,
                                             kCGBitmapByteOrderDefault | kCGImageAlphaPremultipliedFirst);
CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
CGImageRef imageRefWithAlpha = CGBitmapContextCreateImage(context);
UIImage *imageWithAlpha = [UIImage imageWithCGImage:imageRefWithAlpha scale:image.scale orientation:image.imageOrientation];
if (unsupportedColorSpace)
    CGColorSpaceRelease(colorspaceRef);
CGContextRelease(context);
CGImageRelease(imageRefWithAlpha);
 
return imageWithAlpha;
```

这个算是核心部分

#### 4.UIView+WebCacheOperation
缓存操作的UIView的分类，支持三种操作，也是整个库中比较核心的操作。
但是首先我们来了解三种操作都要用到的存储数据的方法。
这两个方法用的是OC中runtime方法，原理是两个文件关联方法，和上层的存储方法差不多，传入value和key对应，取出也是根据key取出value
object传入self即可

1.设置关联方法

```
//传入object和key和value，policy
//policy即存储方式，和声明使用几种属性大致相同，有copy,retain,copy,retain_nonatomic,assign 五种）
 
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```

2.取出方法

//传入object和key返回value
id objc_getAssociatedObject(id object, const void *key)
这个方法是三种操作都要用到的，获得数据
这个方法是使用前面两个方法，根据缓存加载数据
有缓存则从缓存中取出数据，没有则缓存数据，返回格式是字典格式

```
- (NSMutableDictionary *)operationDictionary {
    NSMutableDictionary *operations = objc_getAssociatedObject(self, &loadOperationKey);
    if (operations) {
        return operations;
    }
    operations = [NSMutableDictionary dictionary];
    objc_setAssociatedObject(self, &loadOperationKey, operations, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    return operations;
}
```

接下来是三种操作

一.加载图片根据是否有缓存
从获得数据方法获得数据，传入key，先调用第二个方法停止操作，再根据key缓存数据

```
- (void)sd_setImageLoadOperation:(id)operation forKey:(NSString *)key {
    [self sd_cancelImageLoadOperationWithKey:key];
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    [operationDictionary setObject:operation forKey:key];
}
```
二.取消加载图片如果有缓存
先获得方法一的返回字典数据，传入key在返回的字典中查找是否已经存在，如果存在则取消所有操作
conformsToProtocol方法如果符合这个协议（协议中声明了取消方法），调用协议中的取消方法

```
- (void)sd_cancelImageLoadOperationWithKey:(NSString *)key {
    // Cancel in progress downloader from queue
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    id operations = [operationDictionary objectForKey:key];
    if (operations) {
        if ([operations isKindOfClass:[NSArray class]]) {
            for (id <SDWebImageOperation> operation in operations) {
                if (operation) {
                    [operation cancel];
                }
            }
        } else if ([operations conformsToProtocol:@protocol(SDWebImageOperation)]){
            [(id<SDWebImageOperation>) operations cancel];
        }
        [operationDictionary removeObjectForKey:key];
    }
}
```

三.移除缓存
获得方法一的数据，传入key如果key对应的数据在数据中则移除

```
- (void)sd_removeImageLoadOperationWithKey:(NSString *)key {
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    [operationDictionary removeObjectForKey:key];
}
```

#### 5.SDWebImageDownloader
下载器类，需要用到SDWebImageDownloaderOperation类，下载器操作，后面会说到

定义了一些属性

```
//下载队列的最大下载数
@property (assign, nonatomic) NSInteger maxConcurrentDownloads;
//当前下载数
@property (readonly, nonatomic) NSUInteger currentDownloadCount;
//下载超时的时间
@property (assign, nonatomic) NSTimeInterval downloadTimeout;
//是否解压图片，默认是
@property (assign, nonatomic) BOOL shouldDecompressImages;
//下载器顺序，枚举类型，有两种，先进先出，还是后进先出
@property (assign, nonatomic) SDWebImageDownloaderExecutionOrder executionOrder;
 
#####还有一些用户属性
//url证书
 
@property (strong, nonatomic) NSURLCredential *urlCredential;
//用户名
@property (strong, nonatomic) NSString *username;
//密码
@property (strong, nonatomic) NSString *password;
//头像过滤器，block指针类型，接受url和字典headers
@property (nonatomic, copy) SDWebImageDownloaderHeadersFilterBlock headersFilter;
```

init方法
初始化了一些属性和写好http请求头

```
- (id)init {
    if ((self = [super init])) {
        _operationClass = [SDWebImageDownloaderOperation class];
        _shouldDecompressImages = YES;
        _executionOrder = SDWebImageDownloaderFIFOExecutionOrder;
        _downloadQueue = [NSOperationQueue new];
        _downloadQueue.maxConcurrentOperationCount = 6;
        _URLCallbacks = [NSMutableDictionary new];
#ifdef SD_WEBP
        _HTTPHeaders = [@{@"Accept": @"image/webp,image/*;q=0.8"} mutableCopy];
#else
        _HTTPHeaders = [@{@"Accept": @"image/*;q=0.8"} mutableCopy];
#endif
        _barrierQueue = dispatch_queue_create("com.hackemist.SDWebImageDownloaderBarrierQueue", DISPATCH_QUEUE_CONCURRENT);
        _downloadTimeout = 15.0;
    }
    return self;
}
```

核心方法
传入url，下载器选项（接下来会说），进度block，完成回调block

```
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageDownloaderOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageDownloaderCompletedBlock)completedBlock;
```

这个方法非常复杂，定义了http请求，定义了SDWebImageDownloaderOperation实例，即下载器操作，初始化过程非常复杂，用到了http请求，用到了前面定义的那些属性，最后返回这个操作，这个过程建议去看源码

#### 6.SDWebImageDownloaderOperation
下载器的操作
直接看前面下载器需要用到的初始化方法
需要初始化了各种属性，主要是几个block，进度block，完成回调block，取消回调block

```
- (id)initWithRequest:(NSURLRequest *)request
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock {
    if ((self = [super init])) {
        _request = request;
        _shouldDecompressImages = YES;
        _shouldUseCredentialStorage = YES;
        _options = options;
        _progressBlock = [progressBlock copy];
        _completedBlock = [completedBlock copy];
        _cancelBlock = [cancelBlock copy];
        _executing = NO;
        _finished = NO;
        _expectedSize = 0;
        responseFromCached = YES;
    }
    return self;
}
```

#### 7.SDWebImageManager
图片管理器，负责图片的下载，转换，缓存等
这里先说明SDWebImageOptions
1 << X 这种是位运算符，1左移多少位，后面要用到，说明一下

```
typedef NS_OPTIONS(NSUInteger, SDWebImageOptions) {
SDWebImageRetryFailed = 1 << 0，//无效url会加入黑名单，这个标志是禁用黑名单
SDWebImageLowPriority = 1 << 1, //低优先级，会后下载
SDWebImageCacheMemoryOnly = 1 << 2, //禁用磁盘缓存
SDWebImageProgressiveDownload = 1 << 3, //显示下载进度，下载完才显示
SDWebImageRefreshCached = 1 << 4, //重新从远程缓存
SDWebImageContinueInBackground = 1 << 5, //在后台继续下载图片
SDWebImageHandleCookies = 1 << 6, //把cookie存储到NSHTTPCookieStorey
SDWebImageAllowInvalidSSLCertificates = 1 << 7, //允许非信任ssl证书
SDWebImageHighPriority = 1 << 8, //高优先级，插队下载队列
SDWebImageDelayPlaceholder = 1 << 9, //显示的是替代图片(初始化图片)
SDWebImageTransformAnimatedImage = 1 << 10, //转换图片大小
SDWebImageAvoidAutoSetImage = 1 << 11 //避免自动设置图片（想手动的时候设置)
};
```

这里包含了各种选择

核心方法
传入url，上面的options，进度block，完成回调block

```
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
options:(SDWebImageOptions)options
progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;
```
实例化过程请去看源码

说下其他方法
一个传入key判断图片是否存在存储空间的方法
使用的是NSFileManager的方法

```
- (BOOL)diskImageExistsWithKey:(NSString *)key {
    BOOL exists = NO;
    exists = [[NSFileManager defaultManager] fileExistsAtPath:[self defaultCachePathForKey:key]];
 
    if (!exists) {
        exists = [[NSFileManager defaultManager] fileExistsAtPath:[[self defaultCachePathForKey:key] stringByDeletingPathExtension]];
    }
    return exists;
}
```
还有从存储空间或者缓存取出图片的方法
self.memCache是AutoPurgeCache（单纯继承自NSCache）的实例
从存储空间取图片要先判断内存中是否存在，然后才从存储空间中查找

```
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key {
    return [self.memCache objectForKey:key];
}
 
- (UIImage *)imageFromDiskCacheForKey:(NSString *)key {
 
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        return image;
    }
    UIImage *diskImage = [self diskImageForKey:key];
    if (diskImage && self.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(diskImage);
        [self.memCache setObject:diskImage forKey:key cost:cost];
    }
    return diskImage;
}
 
- (UIImage *)diskImageForKey:(NSString *)key {
    NSData *data = [self diskImageDataBySearchingAllPathsForKey:key];
    if (data) {
        UIImage *image = [UIImage sd_imageWithData:data];
        image = [self scaledImageForKey:key image:image];
        if (self.shouldDecompressImages) {
            image = [UIImage decodedImageWithImage:image];
        }
        return image;
    }
    else {
        return nil;
    }
}
```

还有几个清除方法

```
- (void)clearDisk;
- (void)clearMemory;
 
- (void)cleanDisk;
- (void)cleanDiskWithCompletionBlock;
```

#### 8.SDWebImagePrefetcher
预抓取器，用来预抓取图片
核心方法

```
//预抓取图片
- (void)prefetchURLs:(NSArray *)urls progress:(SDWebImagePrefetcherProgressBlock)progressBlock completed:(SDWebImagePrefetcherCompletionBlock)completionBlock;
//取消预抓取图片
- (void)cancelPrefetching;
```
先来看预抓取图片
传入url，进度block，完成回调block
首先取消抓取，然后重新开始

```
- (void)prefetchURLs:(NSArray *)urls progress:(SDWebImagePrefetcherProgressBlock)progressBlock completed:(SDWebImagePrefetcherCompletionBlock)completionBlock {
    [self cancelPrefetching]; // Prevent duplicate prefetch request
    self.startedTime = CFAbsoluteTimeGetCurrent();
    self.prefetchURLs = urls;
    self.completionBlock = completionBlock;
    self.progressBlock = progressBlock;
 
    if (urls.count == 0) {
        if (completionBlock) {
            completionBlock(0,0);
        }
    } else {
        NSUInteger listCount = self.prefetchURLs.count;
        for (NSUInteger i = 0; i < self.maxConcurrentDownloads && self.requestedCount < listCount; i++) {
            [self startPrefetchingAtIndex:i];
        }
    }
}
```
最后调用startPrefetchingAtIndex:方法，再调用self.manager的核心方法，即开始下载图片

```
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;
```

最后是各种分类，即直接再初始化的控件设置图片，支持UIButton，UIImage,UIImageView，大同小异，我直接说UIImageView+WebCache

#### 9.UIImageView+WebCache
很多加载方法最终都会以缺省参数方式或者直接调用这个方法，传入一个URL，一个用来初始化的image，一个options（枚举，下面详细说明），一个progressBlock（返回图片接受进度等），一个completedBlock（完成回调block）

```
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock;
```
首先根据url缓存图片，这里用到的是OC的runtime中的关联方法（见4）

```
objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```

然后判断options（见7）是其他选择则直接给图片赋值placehoder图片，这里判断使用的是 & 与 位运算符，SDWebImageDelayPlacehoder是 1 << 9，1左移9位与options相与

``` objc
if (!(options & SDWebImageDelayPlaceholder)) {
    dispatch_main_async_safe(^{
        self.image = placeholder;
    });
}
```
如果url存在，则定义图片操作，使用图片管理器的单例来调用核心方法（下载图片方法）

```
id <SDWebImageOperation> operation = [SDWebImageManager.sharedManager downloadImageWithURL:url options:options progress:progressBlock completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
    //过程省略
 
}
```

#### 10.UIImage+GIF
gif的实现使用了ImageIO中的CGImageSourceRef
用获得的gif数据得到CGImageSourceRef，然后算出时间，在这个时间内把图片一帧一帧的放进一个数组，最后再把这个数组和时间转成图片，就成了gif

```

+ (UIImage *)sd_animatedGIFWithData:(NSData *)data {
    if (!data) {
        return nil;
    }
 
    CGImageSourceRef source = CGImageSourceCreateWithData((__bridge CFDataRef)data, NULL);
 
    size_t count = CGImageSourceGetCount(source);
 
    UIImage *animatedImage;
 
    if (count <= 1) {
        animatedImage = [[UIImage alloc] initWithData:data];
    }
    else {
        NSMutableArray *images = [NSMutableArray array];
 
        NSTimeInterval duration = 0.0f;
 
        for (size_t i = 0; i < count; i++) {
            CGImageRef image = CGImageSourceCreateImageAtIndex(source, i, NULL);
 
            duration += [self sd_frameDurationAtIndex:i source:source];
 
            [images addObject:[UIImage imageWithCGImage:image scale:[UIScreen mainScreen].scale orientation:UIImageOrientationUp]];
 
            CGImageRelease(image);
        }
 
        if (!duration) {
            duration = (1.0f / 10.0f) * count;
        }
 
        animatedImage = [UIImage animatedImageWithImages:images duration:duration];
    }
 
    CFRelease(source);
 
    return animatedImage;
}
```

### 总结
看完SDWebImage的源码后感觉学到了很多东西，特别是缓存那一块写的特别好。ImageIO和objc/runtime很值得学习一下。也感觉到设计出这样一个库需要很强大的知识面，非常严谨的思想，真的不容易。

