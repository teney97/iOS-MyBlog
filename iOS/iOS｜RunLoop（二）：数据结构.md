## RunLoop 系列文章
[深入浅出 RunLoop（一）：初识](https://juejin.im/post/6844904073922101261)<br>
[深入浅出 RunLoop（二）：数据结构](https://juejin.im/post/6844904073930473480)<br>
[深入浅出 RunLoop（三）：事件循环机制](https://juejin.im/post/6844904073938878477)<br>
[深入浅出 RunLoop（四）：RunLoop 与线程](https://juejin.im/post/6844904073959833613)<br>
[深入浅出 RunLoop（五）：RunLoop 与 NSTimer](https://juejin.im/post/6844904073972416519)<br>
[iOS - 聊聊 autorelease 和 @autoreleasepool：RunLoop 与 @autoreleasepool](https://juejin.im/post/6844904094503567368#heading-17)

![](https://user-gold-cdn.xitu.io/2020/2/27/17086e906ef0e25c?w=1920&h=1080&f=jpeg&s=125353)

## CFRunLoopRef
`RunLoop`对象的底层就是一个`CFRunLoopRef`结构体，它里面存储着：
* _pthread：`RunLoop`与线程是一一对应关系
* _commonModes：存储着 NSString 对象的集合（Mode 的名称）
* _commonModeItems：存储着被标记为通用模式的`Source0`/`Source1`/`Timer`/`Observer`
* _currentMode：`RunLoop`当前的运行模式
* _modes：存储着`RunLoop`所有的 Mode（`CFRunLoopModeRef`）模式

```objc
// CFRunLoop.h
typedef struct __CFRunLoop * CFRunLoopRef;
// CFRunLoop.c
struct __CFRunLoop {
    pthread_t _pthread;  // 与线程一一对应
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    ...
};
```
## CFRunLoopModeRef
* `CFRunLoopModeRef`代表`RunLoop`的运行模式；
* 一个`RunLoop`包含若干个 Mode，每个 Mode 又包含若干个`Source0`/`Source1`/`Timer`/`Observer`；
* `RunLoop`启动时只能选择其中一个 Mode，作为 currentMode；
* 如果需要切换 Mode，只能退出当前 Loop，再重新选择一个 Mode 进入，切换模式不会导致程序退出；
* 不同 Mode 中的`Source0`/`Source1`/`Timer`/`Observer`能分隔开来，互不影响；
* 如果 Mode 里没有任何`Source0`/`Source1`/`Timer`/`Observer`，`RunLoop`会立马退出。

```objc
// CFRunLoop.h
typedef struct __CFRunLoopMode *CFRunLoopModeRef;
// CFRunLoop.c
struct __CFRunLoopMode {
    CFStringRef _name;             // mode 类型，如：NSDefaultRunLoopMode
    CFMutableSetRef _sources0;     // CFRunLoopSourceRef
    CFMutableSetRef _sources1;     // CFRunLoopSourceRef
    CFMutableArrayRef _observers;  // CFRunLoopObserverRef
    CFMutableArrayRef _timers;     // CFRunLoopTimerRef
    ...
};
```

### RunLoop 的常见模式

ModeName|描述
:--:|:--:
NSDefaultRunLoopMode / KCFRunLoopDefaultMode|默认模式
UITrackingRunLoopMode|界面追踪模式，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响；
NSRunLoopCommonModes / KCFRunLoopCommonModes|通用模式（默认包含 KCFRunLoopDefaultMode 和 UITrackingRunLoopMode）<br><br>该模式不是实际存在的一种模式，它只是一个特殊的标记，是同步`Source0`/`Source1`/`Timer`/`Observer`到多个 Mode 中的技术方案。被标记为通用模式的`Source0`/`Source1`/`Timer`/`Observer`都会存放到 _commonModeItems 集合中，会同步这些`Source0`/`Source1`/`Timer`/`Observer`到多个 Mode 中。
>**备注：**
>* `NSDefaultRunLoopMode`和`NSRunLoopCommonModes`属于`Foundation`框架；
>* `KCFRunLoopDefaultMode`和`KCFRunLoopCommonModes`属于`Core Foundation`框架；
>* 前者是对后者的封装，作用相同。


### CFRunLoopModeRef 这样设计有什么好处？Runloop为什么会有多个 Mode？
* Mode 做到了屏蔽的效果，当`RunLoop`运行在 Mode1 下面的时候，是处理不了 Mode2 的事件的；
* 比如`NSDefaultRunLoopMode`默认模式和`UITrackingRunLoopMode`滚动模式，滚动屏幕的时候就会切换到滚动模式，就不用去处理默认模式下的事件了，保证了 UITableView 等的滚动顺畅。


## CFRunLoopSourceRef
* 在`RunLoop`中有两个很重要的概念，一个是上面提到的`模式`，还有一个就是`事件源`。`事件源`分为`输入源（Input Sources）`和`定时器源（Timer Sources）`两种；
* `输入源（Input Sources）`又分为`Source0`和`Source1`两种，以下`__CFRunLoopSource`中的共用体`union`中的`version0`和`version1`就分别对应`Source0`和`Source1`。
```objc
// CFRunLoop.h
typedef struct __CFRunLoopSource * CFRunLoopSourceRef;
// CFRunLoop.m
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;                       /* immutable */
    CFMutableBagRef _runLoops;
    union {
        CFRunLoopSourceContext version0;  /* immutable, except invalidation */
        CFRunLoopSourceContext1 version1; /* immutable, except invalidation */
    } _context;
};
```
**Source0 和 Source1 的区别：**

Input Sources|区别
:--|:--
Source0|需要手动唤醒线程：添加`Source0`到`RunLoop`并不会主动唤醒线程，需要手动唤醒）<br>① 触摸事件处理<br>② `performSelector:onThread:`
Source1|具备唤醒线程的能力<br>① 基于 Port 的线程间通信<br>② 系统事件捕捉：系统事件捕捉是由`Source1`来处理，然后再交给`Source0`处理




## CFRunLoopTimerRef
* `CFRunloopTimer`和`NSTimer`是 toll-free bridged 的，可以相互转换；
* `performSelector:withObject:afterDelay:`方法会创建`timer`并添加到`RunLoop`中。

```objc
// CFRunLoop.h
typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;
// CFRunLoop.c
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;           // 添加该 timer 的 RunLoop
    CFMutableSetRef _rlModes;        // 所有包含该 timer 的 modeName
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;        /* immutable 理想时间间隔 */    
    CFTimeInterval _tolerance;       /* mutable 时间偏差 */  
    uint64_t _fireTSR;                 /* TSR units */
    CFIndex _order;                  /* immutable */
    CFRunLoopTimerCallBack _callout; /* immutable 回调入口 */
    CFRunLoopTimerContext _context;  /* immutable, except invalidation */
};
```

## CFRunLoopObserverRef
**作用**
* `CFRunLoopObserverRef`用来监听`RunLoop`的 6 种活动状态
```objc
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),          // 即将进入 RunLoop
    kCFRunLoopBeforeTimers = (1UL << 1),   // 即将处理 Timers
    kCFRunLoopBeforeSources = (1UL << 2),  // 即将处理 Sources
    kCFRunLoopBeforeWaiting = (1UL << 5),  // 即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),   // 刚从休眠中唤醒
    kCFRunLoopExit = (1UL << 7),           // 即将退出 RunLoop
    kCFRunLoopAllActivities = 0x0FFFFFFFU  // 表示以上所有状态
};
```
* UI 刷新（BeforeWaiting）
* Autorelease pool（BeforeWaiting）

**定义**
```objc
// CFRunLoop.h
typedef struct __CFRunLoopObserver * CFRunLoopObserverRef;
// CFRunLoop.c
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;              // 添加该 observer 的 RunLoop
    CFIndex _rlCount;
    CFOptionFlags _activities;            /* immutable 监听的活动状态 */
    CFIndex _order;                     /* immutable */
    CFRunLoopObserverCallBack _callout;    /* immutable 回调入口 */
    CFRunLoopObserverContext _context;    /* immutable, except invalidation */
};
```
`CFRunLoopObserverRef`中的`_activities`用来保存`RunLoop`的活动状态。当`RunLoop`的状态发生改变时，通过回调`_callout`通知所有监听这个状态的`Observer`。
