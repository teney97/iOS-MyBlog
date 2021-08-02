## RunLoop 系列文章
[深入浅出 RunLoop（一）：初识](https://juejin.im/post/6844904073922101261)<br>
[深入浅出 RunLoop（二）：数据结构](https://juejin.im/post/6844904073930473480)<br>
[深入浅出 RunLoop（三）：事件循环机制](https://juejin.im/post/6844904073938878477)<br>
[深入浅出 RunLoop（四）：RunLoop 与线程](https://juejin.im/post/6844904073959833613)<br>
[深入浅出 RunLoop（五）：RunLoop 与 NSTimer](https://juejin.im/post/6844904073972416519)<br>
[iOS - 聊聊 autorelease 和 @autoreleasepool：RunLoop 与 @autoreleasepool](https://juejin.im/post/6844904094503567368#heading-17)




![](https://user-gold-cdn.xitu.io/2020/2/27/17086e906ef0e25c?w=1920&h=1080&f=jpeg&s=125353)


## 大纲


![](https://user-gold-cdn.xitu.io/2020/4/19/1718ef9c4b95c9dd?w=5305&h=3633&f=png&s=1920778)


## RunLoop 简介

* 运行循环，在程序运行过程中循环做一些事情（如接收消息、处理消息、休眠等待等）；
* `RunLoop`是通过内部维护的事件循环来对事件/消息进行管理的一个对象；
* `RunLoop`不是一个简单的`do...while`循环，它涉及到用户态和内核态之间的切换。

### 事件循环
事件循环就是对事件/消息进行管理，事件循环可以达到：
* 没有消息需要处理时，休眠线程以避免资源占用。从用户态切换到内核态，等待消息；
* 有消息需要处理时，立刻唤醒线程，回到用户态处理消息；
* 通过调用`mach_msg()`函数来转移当前线程的控制权给内核态/用户态。


## RunLoop 的基本作用
* 保持程序的持续运行：<br>
如果没有`RunLoop`，`main()`函数一执行完，程序就会立刻退出。<br>
而我们的 iOS 程序能保持持续运行的原因就是在`main()`函数中调用了`UIApplicationMain`函数，这个函数内部会启动主线程的`RunLoop`；
* 处理 App 中的的各种事件（比如触摸事件、定时器事件等）；
* 节省 CPU 资源，提高程序性能：该做事时做事，该休息时休息。

## RunLoop 的应用范畴
* 定时器（Timer）、PerformSelector
* GCD：dispatch_async(dispatch_get_main_queue(), ^{ });
* 事件响应、手势识别、界面刷新
* 网络请求
* [AutoreleasePool](https://juejin.im/post/6844904094503567368#heading-17)



## RunLoop 对象
* iOS 中有 2 套 API 来访问和使用`RunLoop`：
    * ① Foundation：`NSRunLoop`（是`CFRunLoopRef`的封装，提供了面向对象的 API）
    * ② Core Foundation：`CFRunLoopRef`
* `NSRunLoop`和`CFRunLoopRef`都代表着`RunLoop`对象
* `NSRunLoop`不开源，而`CFRunLoopRef`是开源的：[Core Foundation 源码](https://opensource.apple.com/tarballs/CF/)
* 获取`RunLoop`对象的方式：
```objc
    // Foundation
    [NSRunLoop mainRunLoop];     // 获取主线程的 RunLoop 对象
    [NSRunLoop currentRunLoop];  // 获取当前线程的 RunLoop 对象
    // Core Foundation
    CFRunLoopGetMain();     // 获取主线程的 RunLoop 对象
    CFRunLoopGetCurrent();  // 获取当前线程的 RunLoop 对象
```



## RunLoop 在实际开发中的应用
* 使用端口或自定义输入源与其他线程进行通信
* 在子线程上使用定时器
* 解决`NSTimer`在滑动时停止工作的问题
* 控制线程的生命周期，实现一个常驻线程
* 在 Cocoa 应用程序中使用任何`performSelector...`方法
* 监控应用卡顿
* 性能优化
* ......


## 相关链接
[Core Foundation 源码](https://opensource.apple.com/tarballs/CF/)<br>
[苹果官方文档 RunLoop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)
