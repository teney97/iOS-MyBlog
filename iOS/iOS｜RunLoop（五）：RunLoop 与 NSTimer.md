## RunLoop 系列文章
[深入浅出 RunLoop（一）：初识](https://juejin.im/post/6844904073922101261)<br>
[深入浅出 RunLoop（二）：数据结构](https://juejin.im/post/6844904073930473480)<br>
[深入浅出 RunLoop（三）：事件循环机制](https://juejin.im/post/6844904073938878477)<br>
[深入浅出 RunLoop（四）：RunLoop 与线程](https://juejin.im/post/6844904073959833613)<br>
[深入浅出 RunLoop（五）：RunLoop 与 NSTimer](https://juejin.im/post/6844904073972416519)<br>
[iOS - 聊聊 autorelease 和 @autoreleasepool：RunLoop 与 @autoreleasepool](https://juejin.im/post/6844904094503567368#heading-17)

![](https://user-gold-cdn.xitu.io/2020/2/27/17086e906ef0e25c?w=1920&h=1080&f=jpeg&s=125353)



## RunLoop 与 NSTimer
* 由前面的文章我们知道，`NSTimer`是由`RunLoop`来管理的，`NSTimer`其实就是`CFRunLoopTimerRef`，他们之间是 toll-free bridged 的，可以相互转换；
* 如果我们在子线程上使用`NSTimer`，就必须开启子线程的`RunLoop`，否则定时器无法生效。


## 解决 tableview 滑动时 NSTimer 失效的问题
* 问题：由前面的文章我们知道，`RunLoop`同一时间只能运行在一种模式下，当我们滑动`tableview`/`scrollview`的时候`RunLoop`会切换到`UITrackingRunLoopMode`界面追踪模式下。如果我们的`NSTimer`是添加到`RunLoop`的`KCFRunLoopDefaultMode`/`NSDefaultRunLoopMode`默认模式下的话，此时是会失效的。
* 解决：我们可以将`NSTimer`添加到`RunLoop`的`KCFRunLoopCommonModes`/`NSRunLoopCommonModes`通用模式下，来保证无论在默认模式还是界面追踪模式下`NSTimer`都可以执行。
* `NSTimer`的创建方式

如果我们是通过以下方法创建的`NSTimer`，是自动添加到`RunLoop`的默认模式下的
```objc
    [NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
        NSLog(@"123");
    }];
```
我们可以通过以下方法创建`NSTimer`，来自定义添加到`RunLoop`的某种模式下
```objc
    NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
        NSLog(@"123");
    }];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```
>**注意：** 如果是通过`timerxxx`开头方法创建的`NSTimer`是不会自动添加到`RunLoop`中的，所以一定要记得手动添加，否则`NSTimer`不生效。

## CFRunLoopAddTimer 函数实现
`CFRunLoopAddTimer()`函数中会判断传入的`modeName`模式名称是不是
`kCFRunLoopCommonModes`通用模式，是的话就会将`timer`添加到`RunLoop`的 _commonModeItems 集合中，并同步该`timer`到 _commonModes 里的所有模式中，这样无论在默认模式还是界面追踪模式下`NSTimer`都可以执行。
```objc
void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName) {    
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rlt) || (NULL != rlt->_runLoop && rlt->_runLoop != rl)) return;
    __CFRunLoopLock(rl);
    if (modeName == kCFRunLoopCommonModes) {       // 判断 modeName 是不是 kCFRunLoopCommonModes
        CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        if (NULL == rl->_commonModeItems) {    // 懒加载，判断 _commonModeItems 是否为空，是的话创建
            rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
        }
        CFSetAddValue(rl->_commonModeItems, rlt);  // 将 timer 添加到 _commonModeItems 中
        if (NULL != set) {
            CFTypeRef context[2] = {rl, rlt};  // 将 timer 和 RunLoop 封装到 context 中
            /* add new item to all common-modes */
            // 遍历 commonModes，将 timer 添加到 commonModes 的所有模式下
            CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
            CFRelease(set);
        }
    ......
    }
}


static void __CFRunLoopAddItemToCommonModes(const void *value, void *ctx) {
    CFStringRef modeName = (CFStringRef)value;
    CFRunLoopRef rl = (CFRunLoopRef)(((CFTypeRef *)ctx)[0]);
    CFTypeRef item = (CFTypeRef)(((CFTypeRef *)ctx)[1]);
    if (CFGetTypeID(item) == CFRunLoopSourceGetTypeID()) {
    CFRunLoopAddSource(rl, (CFRunLoopSourceRef)item, modeName);
    } else if (CFGetTypeID(item) == CFRunLoopObserverGetTypeID()) {
    CFRunLoopAddObserver(rl, (CFRunLoopObserverRef)item, modeName);
    } else if (CFGetTypeID(item) == CFRunLoopTimerGetTypeID()) {
    CFRunLoopAddTimer(rl, (CFRunLoopTimerRef)item, modeName);
    }
}
```

## NSTimer 和 CADisplayLink 存在的问题
* 不准时：`NSTime`和`CADisplayLink`底层都是基于`RunLoop`的`CFRunLoopTimerRef`的实现的，也就是说它们都依赖于`RunLoop`。如果`RunLoop`的任务过于繁重，会导致它们不准时。

比如`NSTimer`每1.0秒就会执行一次任务，`Runloop`每进行一次循环，就会看一下`NSTimer`的时间是否达到1.0秒，是的话就执行任务。但是由于`Runloop`每一次循环的任务不一样，所花费的时间就不固定。假设第一次循环所花时间为 0.2s，第二次 0.3s，第三次 0.3s，则再过 0.2s 就会执行`NSTimer`的任务，这时候可能`Runloop`的任务过于繁重，第四次花了0.5s，那加起来时间就是 1.3s，导致`NSTimer`不准时。

解决方法：使用 GCD 的定时器。GCD 的定时器是直接跟系统内核挂钩的，而且它不依赖于`RunLoop`，所以它非常的准时。示例如下：

```objc
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    
    //创建定时器
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    //设置时间（start:几s后开始执行； interval:时间间隔）
    uint64_t start = 2.0;    //2s后开始执行
    uint64_t interval = 1.0; //每隔1s执行
    dispatch_source_set_timer(timer, dispatch_time(DISPATCH_TIME_NOW, start * NSEC_PER_SEC), interval * NSEC_PER_SEC, 0);
    //设置回调
    dispatch_source_set_event_handler(timer, ^{
       NSLog(@"%@",[NSThread currentThread]);
    });
    //启动定时器
    dispatch_resume(timer);
    NSLog(@"%@",[NSThread currentThread]);
    
    self.timer = timer;
/*
2020-02-01 21:34:23.036474+0800 多线程[7309:1327653] <NSThread: 0x600001a5cfc0>{number = 1, name = main}
2020-02-01 21:34:25.036832+0800 多线程[7309:1327705] <NSThread: 0x600001acb600>{number = 7, name = (null)}
2020-02-01 21:34:26.036977+0800 多线程[7309:1327705] <NSThread: 0x600001acb600>{number = 7, name = (null)}
2020-02-01 21:34:27.036609+0800 多线程[7309:1327707] <NSThread: 0x600001a1e5c0>{number = 4, name = (null)}
 */
```

* 循环引用
