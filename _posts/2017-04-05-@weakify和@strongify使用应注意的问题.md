---
published: true
---
本文主要谈论关于这两个宏使用的三个问题：

- 如果多层block嵌套怎么办？
- 如果block返回时，捕获的对象已经dealloc，该怎么办？
- 使用weakify开发时，一定要使用Debug模式



## block嵌套

如果存在block嵌套的情况，以下哪种用法是对的呢？以下例子来自[github issue][github-issue]

{% highlight c %}
//用法1：使用一次weakify，block中多次使用strongify
- (void) test {
      @weakify(self)
      id block = ^{
        @strongify(self);
        [self doSomething];
        nestedBlock = ^{   
          @strongify(self);		// 嵌套的block不再使用weakify，然而再次使用strongify
          [self doSomethingElse];
        }();
      };
      block();
}
{% endhighlight %}

{% highlight c %}
//用法2：多次使用weakify，多次使用strongify
- (void) test {
      @weakify(self)
      id block = ^{
        @strongify(self);
        [self doSomething];
        @weakify(self)         // 嵌套的block再次使用weakify和strongify
        nestedBlock = ^{
          @strongify(self);
          [self doSomethingElse];
        }();
      };
      block();
}
{% endhighlight %}

{% highlight c %}
//用法3：一次使用weakify，一次使用strongify，嵌套的block忽略之
- (void) test {
      @weakify(self)
      id block = ^{
        @strongify(self);
        [self doSomething];
        nestedBlock = ^{  			//嵌套的block不再使用weakify和strongify
          [self doSomethingElse];
        }();
      };
      block();
}
{% endhighlight %}

毫无疑问，最外层的嵌套是需要weakify和strongify的，我们将其展开：

{% highlight c %}
- (void) test {
      __weak id weakSelf = self
      id block = ^{
        id self = weakSelf;
        [self doSomething];
        nestedBlock = ^{  	
          //嵌套的block
          [self doSomethingElse];
        }();
      };
      block();
}
{% endhighlight %}

可以看到嵌套的block里面的self，实际是strongSelf，实际上nestedBlock会持有当前对象，retainCount会加1，然而就像`jspahrsummers `所说的那样,在block执行后，nestedBlock才会创建，那么self的retainCount会加1，紧接着nextBlock由于没有被其他变量引用，只是存在于stack中，因而会被释放掉，self的retainCount也会随之减1。因而不存在任何问题。

那么如果nestedBlock会被引用呢？ 是long live的block ，例如以下这段代码：

{% highlight c %}
- (void) test {
      __weak id weakSelf = self
      id block = ^{
        id self = weakSelf;
        [self doSomething];
        self.nestedBlock = ^{  	
          //嵌套的block
          [self doSomethingElse];
        }();
      };
      block();
}
{% endhighlight %}

可以看出nestedBlock不会被立即释放，因而也有了循环引用，self强引用nestedBlock，nestedBlock也强引用self。

综上分析用法1，2，3都没问题，但是用法2中的多次weakify是多余的，用法3 在nestedblock的生命周期延长（或者说是被外面的实例变量引用）的特定情况下，会发生循环引用，用法2是不会存在任何问题的。
在[stackoverflow][stackoverflow] 也讨论了类似的的问题

## 捕获的对象已经dealloc

在某些情况下，block捕获的weak变量已经变为nil，此时block是没有必要再执行下去的，在项目中，我们可以增加宏定义：

{% highlight c %}
#define strongify_return_if_nil(VAR, ...) strongify(VAR); if (VAR == nil) { return __VA_ARGS__; }
{% endhighlight %}

该宏判断参数是否变为了nil，如果是nil的话，则返回，使用__VA_ARGS__是为了兼容有返回值的block，缺陷是此宏只能一次只能创建一个指向weak_VAR的变量了，大多数情况下还是不存在任何问题的。

## 使用weakify开发时，一定要使用Debug模式
weakify和strongify的定义用到了宏rac_keywordify，它的缺陷是在非Debug模式下并且恰好该block是有返回值的，try,catch会抑制编译器检查代码的block是否存在返回值的代码，因而存在风险，所以开发者自己需要确保使用Debug模式下进行开发
{% highlight c %}
#define rac_keywordify autoreleasepool {}
#else
#define rac_keywordify try {} @catch (...) {}
#endif
{% endhighlight %}

[github-issue]: https://github.com/jspahrsummers/libextobjc/issues/45
[stackoverflow]:http://stackoverflow.com/questions/28305356/ios-proper-use-of-weakifyself-and-strongifyself