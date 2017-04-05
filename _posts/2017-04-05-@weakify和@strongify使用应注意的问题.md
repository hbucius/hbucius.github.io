---
published: false
---
本文主要谈论关于这两个宏使用的三个问题：

如果多层block嵌套怎么办？
如果block返回时，捕获的对象已经dealloc，该怎么办？
使用weakify开发时，一定要使用Debug模式


## block嵌套



