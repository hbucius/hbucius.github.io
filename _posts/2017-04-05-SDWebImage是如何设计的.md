---
published: true
---
SDWebImage是iOS常用的图片库，[代码行数3k][cloc], 把玩难度属于简单，下图是SDWebImage比较重要的类关系图，画图工具是亿图。

## 主要类的作用

从高层级到低层级来说，最上层是UIImageView和UIButton等UIView类的category，负责封装将SDWebImageManager的方法，使用户容易使用。SDWebImageManager是一个关键类，完成了图片的下载和缓存，其中下载依赖于SDWebImageDownloader，缓存依赖于SDImageCache，也包含了一些Options, 例如失败后重新下载，是否只缓存到memory等，类之间的依赖关系比较清晰

[![](https://raw.githubusercontent.com/hbucius/hbucius.github.io/master/_posts/SDWebImage.png "SDWebImage类关系图")][SDWebImage]	

下面简要的对每个类关键实现进行介绍

## SDWebImageManager

关键函数：
{% highlight c %}

- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;

{% endhighlight %}

该函数大家应该都比较熟悉，具体的实现比较容易看晕，包含了太多的if else 判断。 主要包含以下方面内容: 首先判断该URL是否包含在failedURLs中，如果包含并且失败不再继续的开关打开，则就直接调用completionBlock了; 读取memory cache的缓存，没用命中则异步读取disk 缓存； 如果没用命中disk缓存，则使用SDWebImageDownloader下载图片; 下载图片使用SDImageCache缓存图片，并调用completionBlock。

为了尽可能不重复下载失效图片，使用了failedURLs机制记录过去的失败记录，并且有开关将其关闭；为了尽可能可能cancel掉各种下载，disk 缓存查询等操作，manager使用了runningOperation保存operation时，用户可以取消各项操作

## SDWebImageDownloader
## SDWebImageDownloaderOperation
## SDImageCache
此类主要提供图片的内存缓存以及文件缓存。主要方法如下:

{% highlight c %}

- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk;

{% endhighlight %}

存储图片到memory，如果允许存储到disk，也存一份。同时存在冗余参数image 和 imageData的原因是： 如果只传image,那么存到文件时，需要重新计算data，降低了效率，如果参数只传data，那么使用者也可能已经transform image了，已经该data已经是原始数据了，需要从image重新计算得到data，因而也就有了recalculate 参数。


{% highlight c %}

- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key;

{% endhighlight %}

同步从memory cache获取图片

{% highlight c %}

- (UIImage *)imageFromDiskCacheForKey:(NSString *)key;

{% endhighlight %}

同步从disk cache获取图片，如果memory cache有的话，会优先返回memory cache的

{% highlight c %}

- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock;

{% endhighlight %}

异步从disk cache 获取图片，返回的是Operation, operation唯一的一个作用，就是提供了取消功能

{% highlight c %}
  NSOperation *operation = [NSOperation new];
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) {
            return;
        }

        @autoreleasepool {
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage && self.shouldCacheImagesInMemory) {
                NSUInteger cost = SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });
{% endhighlight %}


{% highlight c %}

- (NSString *)cachePathForKey:(NSString *)key inPath:(NSString *)path {

{% endhighlight %}

从url映射到存储到文件后文件名，用的是md5函数

## AutoPurgeCache
此类是NSCache的子类，唯一的一个作用就是:在收到'memory warning'的通知时，释放内存


此外SDWebImage还提供了两个有用的宏函数

{% highlight c %}

#define dispatch_main_sync_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_sync(dispatch_get_main_queue(), block);\
    }

#define dispatch_main_async_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }

{% endhighlight %}

本文至此结束

[cloc]:https://github.com/AlDanial/cloc