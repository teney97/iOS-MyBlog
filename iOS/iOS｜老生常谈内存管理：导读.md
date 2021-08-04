## 导读

这段时间通过以下资料学习了 Objective-C 的内存管理：

* 书籍：《Objective-C 高级编程：iOS 与 OS X 多线程和内存管理》
* Apple 官方文档：[Advanced Memory Management Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)
* Apple 官方文档：[Transitioning to ARC Release Notes](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011226-CH1-SW11)
* Apple 维护的 Runtime 开源库：[https://opensource.apple.com/tarballs/objc4/](https://opensource.apple.com/tarballs/objc4/)


并总结了以下文章：

* [iOS - 老生常谈内存管理（一）：引用计数](https://juejin.im/post/6844904129676967950)
* [iOS - 老生常谈内存管理（二）：从 MRC 说起](https://juejin.im/post/6844904129676984334)
* [iOS - 老生常谈内存管理（三）：ARC 面世](https://juejin.im/post/6844904130431942670)
* [iOS - 老生常谈内存管理（四）：内存管理方法源码分析](https://juejin.im/post/6844904131719593998)
* [iOS - 老生常谈内存管理（五）：Tagged Pointer](https://juejin.im/post/6844904132940136462)
* [iOS - 老生常谈内存管理（六）：聊聊 autorelease 和 @autoreleasepool](https://juejin.im/post/6844904094503567368)
* iOS - 老生常谈内存管理（七）：努力编辑中💪 💪 💪 

文章大纲：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97606aab433f476bb3dec694fe4ee92c~tplv-k3u1fbpfcp-zoom-1.image)


以下列举了部分有关内存管理的问题。如果你对以下问题存在疑惑，或者只有模糊的答案，那么本系列文章可以给予你帮助。


* iOS 的内存管理方案有哪些？
* 讲讲 iOS 的内存管理机制
* 引用计数机制是怎么工作的？
* 引用计数存储在哪里？以前存储在哪？现在呢？
* 能聊聊 isa 吗？什么是 nonpointer ？
* SideTable 你有了解过吗，它是用来干嘛的？
* 引用计数具体是怎么管理的，你能说说内存管理方法的实现吗？
* 聊聊 MRC 下的内存管理规则吧？
* MRC 下什么时候需要给对象发送 release 消息？什么时候使用 autorelease？
* 为什么不要在初始化方法和 dealloc 中使用访问器方法？
* 为什么初始化方法中需要 self = [super init]？
* 你能讲一下 super 的原理吗？
* ARC 的内存管理规则？
* ARC 下没有 retain / release 等方法了吗？
* Toll-Free Bridged 了解过吗？详细描述一下。
* 所有权修饰符有哪些？
* weak 变量在对象被销毁后是如何置为 nil 的，Runtime 是怎样实现它的？
* Runtime 为 weak 变量赋值的过程？
* 既然 __weak 更安全，那么为什么已经有了 __weak 还要保留 __unsafe_unretained ？
* 循环引用是怎么产生的？MRC 下是如何避免循环引用问题的？
* ARC 下哪些情况会产生循环引用？如何解决？
* 释放 NSAutoreleasePool 对象，使用 [pool release] 与 [pool drain] 的区别？
* @autoreleasepool 你了解多少？
* @autoreleasepool 的实现原理？
* 什么时候需要自己创建 @autoreleasepool？
* ARC 环境下，方法里的局部对象什么时候释放？
* ARC 环境下，autorelease 对象在什么时候释放？
* ARC 环境下，需不需要手动添加 @autoreleasepool？
* Tagged Pointer 是什么？
* 如何判断 Tagged Pointer ？


## 阅读注意
为避免语义混淆，所有文章中的 “释放” 一词均指`release`，“销毁” 一词均指`dealloc`。

如果您在阅读中发现任何错误，欢迎指出。

总结不易，点个关注吧！


