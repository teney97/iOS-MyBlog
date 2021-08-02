## RunLoop 系列文章
[深入浅出 RunLoop（一）：初识](https://juejin.im/post/6844904073922101261)<br>
[深入浅出 RunLoop（二）：数据结构](https://juejin.im/post/6844904073930473480)<br>
[深入浅出 RunLoop（三）：事件循环机制](https://juejin.im/post/6844904073938878477)<br>
[深入浅出 RunLoop（四）：RunLoop 与线程](https://juejin.im/post/6844904073959833613)<br>
[深入浅出 RunLoop（五）：RunLoop 与 NSTimer](https://juejin.im/post/6844904073972416519)<br>
[iOS - 聊聊 autorelease 和 @autoreleasepool：RunLoop 与 @autoreleasepool](https://juejin.im/post/6844904094503567368#heading-17)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c3ac07107afb43c89977fba451cd5c13~tplv-k3u1fbpfcp-zoom-1.image)


## RunLoop 与线程的关系
苹果官方文档中，`RunLoop`的相关介绍写在线程编程指南中，可见`RunLoop`和线程的关系不一般。[Threading Programming Guide（苹果官方文档）](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)
* `RunLoop`对象和线程是一一对应关系；
* `RunLoop`保存在一个全局的`Dictionary`里，线程作为`key`，`RunLoop`作为`value`；
* 如果没有`RunLoop`，线程执行完任务就会退出；如果没有`RunLoop`，主线程执行完`main()`函数就会退出，程序就不能处于运行状态；
* `RunLoop`创建时机：线程刚创建时并没有`RunLoop`对象，`RunLoop`会在第一次获取它时创建；
* `RunLoop`销毁时机：`RunLoop`会在线程结束时销毁；
* 主线程的`RunLoop`已经自动获取（创建），子线程默认没有开启`RunLoop`；
* 主线程的`RunLoop`对象是在`UIApplicationMain`中通过`[NSRunLoop currentRunLoop]`获取，一旦发现它不存在，就会创建`RunLoop`对象。

## 未启动 RunLoop 的子线程
创建一个`NSThread`的子类`HTThread`并重写了`dealloc`方法来观察线程的状态。执行以下代码，发现子线程执行完一次`test`任务就退出销毁了，没有再执行`test`任务，原因就是没有启动该线程的`RunLoop`。
```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    HTThread *thread = [[HTThread alloc] initWithTarget:self selector:@selector(test) object:nil];
    [thread start];
    [self performSelector:@selector(test) onThread:thread withObject:nil waitUntilDone:NO];
}

- (void)test {
    NSLog(@"test on %@", [NSThread currentThread]);
}
// test on <HTThread: 0x600003cb52c0>{number = 7, name = (null)}
// HTThread dealloc
```


## 开启子线程的 RunLoop 的过程
### 获取 RunLoop 对象
可以通过以下方式来获取`RunLoop`对象：
```objc
    // Foundation
    [NSRunLoop mainRunLoop];     // 获取主线程的 RunLoop 对象
    [NSRunLoop currentRunLoop];  // 获取当前线程的 RunLoop 对象
    // Core Foundation
    CFRunLoopGetMain();     // 获取主线程的 RunLoop 对象
    CFRunLoopGetCurrent();  // 获取当前线程的 RunLoop 对象
```
我们来看一下`CFRunLoopGetCurrent()`函数是怎么获取`RunLoop`对象的：
```objc
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());  // 调用 _CFRunLoopGet0 函数并传入当前线程
}
```
```objc
// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {
    t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);
    CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
    CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
    CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
    if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
        CFRelease(dict);
    }
    CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }
    // ️当前线程作为 Key，从 __CFRunLoops 字典中获取 RunLoop 
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    if (!loop) {  // ️如果字典中不存在
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);  // 创建当前线程的 RunLoop
        __CFLock(&loopsLock);
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop) {
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);  // 保存到字典中
            loop = newLoop;
        }
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
        CFRelease(newLoop);
    }
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```
### 启动子线程的 RunLoop
可以通过以下方式来启动子线程的`RunLoop`：
```objc
    // Foundation
    [[NSRunLoop currentRunLoop] run];
    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    // Core Foundation
    CFRunLoopRun();
    CFRunLoopRunInMode(kCFRunLoopDefaultMode, 1.0e10, false);  // 第3个参数：设置为 true，代表执行完 Source/Port 后就会退出当前 loop
```
我们来看一下`CFRunLoopRun()`/`CFRunLoopRunInMode()`函数是怎么启动`RunLoop`的：

```objc
void CFRunLoopRun(void) {
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

SInt32 CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {   
    CHECK_FOR_FORK();
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
```
可以看到它通过调用`CFRunLoopRunSpecific()`函数来启动`RunLoop`，而该函数实现已在[《深入浅出 RunLoop（三）：事件循环机制》](https://juejin.im/post/6844904073938878477)文章中讲解到。


## 实现一个常驻线程
* 好处：经常用到子线程的时候，不用一直创建销毁，提高性能；
* 条件：该任务需是串行的，而非并发；
* 步骤：
    * ① 获取/创建当前线程的`RunLoop`；
    * ② 向该`RunLoop`中添加一个`Source`/`Port`等来维持`RunLoop`的事件循环（如果 Mode 里没有任何`Source0`/`Source1`/`Timer`/`Observer`，`RunLoop`会立马退出）；
    * ③ 启动该`RunLoop`。
* 示例代码及测试输出如下：

```objc
// ViewController.m
#import "ViewController.h"
#import "HTThread.h"

@interface ViewController ()
@property (nonatomic, strong) HTThread *thread;
@property (nonatomic, assign, getter=isStoped) BOOL stopped;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    __weak typeof(self) weakSelf = self;
    
    self.stopped = NO;
    self.thread = [[HTThread alloc] initWithBlock:^{
        NSLog(@"begin-----%@", [NSThread currentThread]);
        
        // ① 获取/创建当前线程的 RunLoop 
        // ② 向该 RunLoop 中添加一个 Source/Port 等来维持 RunLoop 的事件循环
        [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
    
        while (weakSelf && !weakSelf.isStoped) {
            // ③ 启动该 RunLoop
            /* 
              [[NSRunLoop currentRunLoop] run]
              如果调用 RunLoop 的 run 方法，则会开启一个永不销毁的线程
              因为 run 方法会通过反复调用 runMode:beforeDate: 方法，以运行在 NSDefaultRunLoopMode 模式下
              换句话说，该方法有效地开启了一个无限的循环，处理来自 RunLoop 的输入源 Sources 和 Timers 的数据
            */ 
            [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
        NSLog(@"end-----%@", [NSThread currentThread]);    
    }];
    [self.thread start];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    if (!self.thread) return;
    [self performSelector:@selector(test) onThread:self.thread withObject:nil waitUntilDone:NO];
}

// 子线程需要执行的任务
- (void)test
{
    NSLog(@"%s-----%@", __func__, [NSThread currentThread]);
}

// 停止子线程的 RunLoop
- (void)stopThread
{
    // 设置标记为 YES
    self.stopped = YES;   
    // 停止 RunLoop
    CFRunLoopStop(CFRunLoopGetCurrent());    
    NSLog(@"%s-----%@", __func__, [NSThread currentThread]);
    // 清空线程
    self.thread = nil;
}

- (void)dealloc
{
    NSLog(@"%s", __func__);
    
    if (!self.thread) return;
    // 在子线程调用（waitUntilDone设置为YES，代表子线程的代码执行完毕后，当前方法才会继续往下执行）
    [self performSelector:@selector(stopThread) onThread:self.thread withObject:nil waitUntilDone:YES];
}

@end


// HTThread.h
#import <Foundation/Foundation.h>
@interface HTThread : NSThread
@end

// HTThread.m
#import "HTThread.h"
@implementation HTThread
- (void)dealloc
{
    NSLog(@"%s", __func__);
}
@end
```
点击 view，接着退出当前 ViewController。输出如下：
>begin-----<HTThread: 0x600002b71240>{number = 6, name = (null)}<br>
>-[ViewController test]-----<HTThread: 0x600002b71240>{number = 6, name = (null)}<br>
>-[ViewController dealloc]<br>
>-[ViewController stopThread]-----<HTThread: 0x600002b71240>{number = 6, name = (null)}<br>
>end-----<HTThread: 0x600002b71240>{number = 6, name = (null)}<br>
>-[HTThread dealloc]

## 相关链接
[Threading Programming Guide（苹果官方文档）](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)

