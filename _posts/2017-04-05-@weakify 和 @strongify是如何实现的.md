---
published: true
---

在`Objective C`使用block时，为了防止`循环引用`常常使用ReactiveCocoa自带的@weakify和@strongify，那么二者是如何完成防止循环引用的功能的呢？ 

在RACEXTScope.h中，可以找到weakify和strongify的定义：

{% highlight c %}
#define weakify(...) \
    rac_keywordify \
    metamacro_foreach_cxt(rac_weakify_,, __weak, __VA_ARGS__)
  
#define strongify(...) \
    rac_keywordify \
    _Pragma("clang diagnostic push") \
    _Pragma("clang diagnostic ignored \"-Wshadow\"") \
    metamacro_foreach(rac_strongify_,, __VA_ARGS__) \
    _Pragma("clang diagnostic pop")
{% endhighlight %}