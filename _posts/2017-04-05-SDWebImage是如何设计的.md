---
published: true
---
SDWebImage是iOS常用的图片库,[代码行数3k][cloc], 把玩难度属于简单，下图是SDWebImage比较重要的类关系图，画图工具是亿图。

## 主要类的作用

从高层级到低层级来说，最上层是UIImageView和UIButton等UIView类的category，负责封装将SDWebImageManager的方法，使用户容易使用。SDWebImageManager是一个关键类，完成了图片的下载和缓存，其中下载依赖于SDWebImageDownloader，缓存依赖于SDImageCache，也包含了一些Options, 例如失败后重新下载，是否只缓存到memory等，类之间的依赖关系比较清晰

(https://raw.githubusercontent.com/hbucius/hbucius.github.io/master/_posts/SDWebImage.png "SDWebImage类关系图")

下面简要的对每个类关键实现进行介绍

## SDWebImageManager

SDWebImageManager是最重要的一个类，它使用SDWebImageDownloader完成图片的下载,使用SDImageCache完成图片的缓存，给使用者提供了直接的接口

关键函数：
{% highlight c %}

- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;

{% endhighlight %}

该函数大家应该都比较熟悉，具体的实现比较容易看晕，包含了太多的if else 判断。 主要包含以下方面内容: 首先判断该URL是否包含在failedURLs中，如果包含并且失败不再继续的开关打开，则就直接调用completionBlock了; 读取memory cache的缓存，没用命中则异步读取disk 缓存； 如果没用命中disk缓存，则使用SDWebImageDownloader下载图片; 下载图片后使用SDImageCache缓存图片，并调用completionBlock。

为了尽可能不重复下载失效图片，使用了failedURLs机制记录过去的失败记录，并且有开关将其关闭；为了尽可能可能cancel掉各种下载，disk 缓存查询等操作，manager使用了runningOperation保存operation时，用户可以取消各项操作

## SDWebImageDownloader

SDWebImageDownloader的作用很纯粹，就是下载图片，并且执行progress和 completion操作。

关键函数：

{% highlight c %}
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageDownloaderOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageDownloaderCompletedBlock)completedBlock;
 {% endhighlight %}

该类首先要解决相同图片的下载问题，解决思路是将url作为key，completionBlock和progressBlock作为value保存在
URLCallbacks('字典类型')中，每个url只有在第一次加入到URLCallbacks时，才进行下载操作, 下载过程中调用progressBlock数组，完成后调用completionBlock数组

该类不是下载图片的最底层类，它依赖于SDWebImageDownloaderOperation去完成下载操作, 该类设置request的各种开关，创建downloaderOperation将progressBlock 和 completionBlock传给Operation，加入operationQueue 去完成下载功能

## SDWebImageDownloaderOperation

该类是下载图片的最底层类，最终由该类完成图片的下载操作，它是NSOperation的子类。 唯一的函数就是下面的初始化函数接收request，block等。 

{% highlight c %}

- (id)initWithRequest:(NSURLRequest *)request
            inSession:(NSURLSession *)session
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock;                                      
            
 {% endhighlight %}
 
该类的入口函数式start, 在该函数中启动request，并设置自己为request的delegate，在回调函数中，调用各种block等，就是该类主要的功能。 需要注意的是对304 not modified的操作，主要在dataDelegate的此函数中进行

{% highlight c %}
 (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                 didReceiveResponse:(NSURLResponse *)response
                                  completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler;

 {% endhighlight %}
 
有response，但是还没有收到data，如果此时检查response 的status code 是304，就可以取消剩余的操作，直接调用completionBlockr即可

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