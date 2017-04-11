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

该函数物


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