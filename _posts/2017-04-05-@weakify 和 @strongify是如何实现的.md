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

可以发现二者的区别很少，都是首先使用了rac_keywordify，然后用了metamacro_foreach函数，一一来看。

### rac_keywordify

{% highlight c %}

// Details about the choice of backing keyword:
//
// The use of @try/@catch/@finally can cause the compiler to suppress
// return-type warnings.
// The use of @autoreleasepool {} is not optimized away by the compiler,
// resulting in superfluous creation of autorelease pools.
//
// Since neither option is perfect, and with no other alternatives, the
// compromise is to use @autorelease in DEBUG builds to maintain compiler
// analysis, and to use @try/@catch otherwise to avoid insertion of unnecessary
// autorelease pools.
#if DEBUG
#define rac_keywordify autoreleasepool {}
#else
#define rac_keywordify try {} @catch (...) {}
#endif

{% endhighlight %}

可以看到此宏并没有具体的含义，只是为了让使用者在weakify加上@，变为@weakify，这样它就更像一个关键字了。

### metamacro_foreach

该宏在库libextobjc中

{% highlight c %}

/**
 * For each consecutive variadic argument (up to twenty), MACRO is passed the
 * zero-based index of the current argument, CONTEXT, and then the argument
 * itself. The results of adjoining invocations of MACRO are then separated by
 * SEP.
 *
 * Inspired by P99: http://p99.gforge.inria.fr
 */
#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...) \
        metamacro_concat(metamacro_foreach_cxt, metamacro_argcount(__VA_ARGS__))(MACRO, SEP, CONTEXT, __VA_ARGS__)

{% endhighlight %}

该宏有三个确定参数，N个不定参数，MACRO是一个宏函数，类似于函数指针，SEP，CONTEXT的作用后面可以看到

metamacro_concat的作用的是连接，metamacro_argcount的作用是取得宏的个数，例如如果宏有2个不定参数，则该宏可以编译为：

{% highlight c %}

metamacro_foreach_cxt2(MACRO, SEP, CONTEXT, __VA_ARGS__)

{% endhighlight %}

后面可以看出，此宏的唯一作用是进行metamacro_foreach_cxt的展开。

### metamacro_foreach2

{% highlight c %}

#define metamacro_foreach_cxt2(MACRO, SEP, CONTEXT, _0, _1) \
    metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) \
    SEP \
    MACRO(1, CONTEXT, _1)
   
#define metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) MACRO(0, CONTEXT, _0)

{% endhighlight %}

metamacro_foreach_cxt1的作用是在只有一个确定的参数的情况下，将参数传递个MACRO宏，metamacro_foreach_cxt2的定义类似于迭代，即首先执行metamacro_foreach_cxt1，再将第二个参数传递给宏执行，这样相当于第一个参数(_0),第二个参数(_1)，都相继传递给了MACRO 执行。

此外还有metamacro_foreach3至metamacro_foreach20，都是此种定义


### rac_weakify_

该宏是weakify的核心

{% highlight c %}

#define rac_weakify_(INDEX, CONTEXT, VAR) \
    CONTEXT __typeof__(VAR) metamacro_concat(VAR, _weak_) = (VAR);

{% endhighlight %}

该宏函数有三个参数，第一个是index索引，第二个CONTEXT是规定__weak引用的，var是具体的需要weakify的变量，展开就是

__weak __typeof_(VAR) VAR__weak_ = VAR

### rac_strongify_

该宏是strongify的核心

{% highlight c %}

#define rac_strongify_(INDEX, VAR) \
    __strong __typeof__(VAR) VAR = metamacro_concat(VAR, _weak_);

{% endhighlight %}

展开为 __strong __typeof_(VAR) VAR = VAR_weak_

该宏增加一个强引用到weakify生成的变量VAR__weak_

到这里基本就结束了，有兴趣的同学，可以继续往下看看metamacro_concat 和 metamacro_argcount的实现

### metamacro_concat

{% highlight c %}

#define metamacro_concat(A, B) \
        metamacro_concat_(A, B)
     
#define metamacro_concat_(A, B) A ## B

{% endhighlight %}

##即宏连接符，将A，B两个宏拼接在一起。

###  metamacro_argcount


{% highlight c %}

/**
 * Returns the number of arguments (up to twenty) provided to the macro. At
 * least one argument must be provided.
 *
 * Inspired by P99: http://p99.gforge.inria.fr
 */
#define metamacro_argcount(...) \
        metamacro_at(20, __VA_ARGS__, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
        
        
/**
 * Returns the Nth variadic argument (starting from zero). At least
 * N + 1 variadic arguments must be given. N must be between zero and twenty,
 * inclusive.
 */
#define metamacro_at(N, ...) \
        metamacro_concat(metamacro_at, N)(__VA_ARGS__)
        
        
// metamacro_at expansions
#define metamacro_at0(...) metamacro_head(__VA_ARGS__)
#define metamacro_at1(_0, ...) metamacro_head(__VA_ARGS__)
#define metamacro_at2(_0, _1, ...) metamacro_head(__VA_ARGS__)

{% endhighlight %}

metamacro_at是得到可变参数的第N个，metamacro_argcount的实现非常巧妙，metamacro_at的实现也是通过展开式的方式，得到可变参数的第一个来实现的

到此本文结束