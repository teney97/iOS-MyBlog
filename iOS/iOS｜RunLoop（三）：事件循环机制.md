## RunLoop 系列文章
[深入浅出 RunLoop（一）：初识](https://juejin.im/post/6844904073922101261)<br>
[深入浅出 RunLoop（二）：数据结构](https://juejin.im/post/6844904073930473480)<br>
[深入浅出 RunLoop（三）：事件循环机制](https://juejin.im/post/6844904073938878477)<br>
[深入浅出 RunLoop（四）：RunLoop 与线程](https://juejin.im/post/6844904073959833613)<br>
[深入浅出 RunLoop（五）：RunLoop 与 NSTimer](https://juejin.im/post/6844904073972416519)<br>
[iOS - 聊聊 autorelease 和 @autoreleasepool：RunLoop 与 @autoreleasepool](https://juejin.im/post/6844904094503567368#heading-17)

![](https://user-gold-cdn.xitu.io/2020/2/27/17086e906ef0e25c?w=1920&h=1080&f=jpeg&s=125353)

## 前言
前面我们介绍了`RunLoop`的基本概念以及相关数据结构，这篇我们来讲解一下`RunLoop`到底是怎么工作的。


## 主线程的 RunLoop 的启动过程
首先我们来看一下主线程的`RunLoop`的启动过程。<br>
前面我们说过，我们的 iOS 程序能保持持续运行的原因就是在`main()`函数中调用了`UIApplicationMain`函数，这个函数内部会启动主线程的`RunLoop`。<br>
打断点，通过 LLDB 指令`bt`查看函数调用栈如下：
![](https://user-gold-cdn.xitu.io/2020/2/27/170866a7c74d9fcf?w=1240&h=853&f=png&s=593974)
可以看到，在`UIApplicationMain`函数中调用了 Core Foundation 框架下的`CFRunLoopRunSpecific`函数。

## CFRunLoopRunSpecific 函数实现：RunLoop 的入口
查看源码中该函数的实现，如下：
```objc
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    // 根据 modeName 找到本次运行的 mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    // 如果没找到 || mode 中没有注册任何事件，则就此停止，不进入循环
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
    Boolean did = false;
    if (currentMode) __CFRunLoopModeUnlock(currentMode);
    __CFRunLoopUnlock(rl);
    return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;

    // 通知 Observers：即将进入 RunLoop
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    // RunLoop 具体要做的事情
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    // 通知 Observers：即将退出 RunLoop
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

        __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopPopPerRunData(rl, previousPerRun);
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```
删掉不重要的细节：
```objc
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */

    // 通知 Observers：即将进入 RunLoop
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    // RunLoop 具体要做的事情
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    // 通知 Observers：即将退出 RunLoop
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

    return result;
}
```

从函数调用栈，以及`CFRunLoopRunSpecific`函数的实现中可以得知，`RunLoop`事件循环的实现机制体现在`__CFRunLoopRun`函数中。
![](https://user-gold-cdn.xitu.io/2020/2/27/170866a7ca3f8e56?w=762&h=1008&f=png&s=44081)


## __CFRunLoopRun 函数实现：事件循环的实现机制
由于该函数实现较复杂，以下为删掉细节的精简版本，想探究具体的可以查看 [Core Foundation 源码](https://opensource.apple.com/tarballs/CF/)。


```objc
/**
 *  __CFRunLoopRun
 *
 *  @param rl              运行的 RunLoop 对象
 *  @param rlm             运行的 mode
 *  @param seconds         loop 超时时间
 *  @param stopAfterHandle true: RunLoop 处理完事件就退出  false:一直运行直到超时或者被手动终止
 *  @param previousMode    上一次运行的 mode
 *
 *  @return 返回 4 种状态
 */
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) 
{
    int32_t retVal = 0;
    do {
        // 通知 Observers：即将处理 Timers
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        // 通知 Observers：即将处理 Sources
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        // 处理 Blocks
        __CFRunLoopDoBlocks(rl, rlm);
        // 处理 Sources0
        if (__CFRunLoopDoSources0(rl, rlm, stopAfterHandle)) {
            // 处理 Blocks
            __CFRunLoopDoBlocks(rl, rlm);
        }
        // 判断有无 Source1
        if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
            // 如果有 Source1，就跳转到 handle_msg
            goto handle_msg;
        }
        // 通知 Observers：即将进入休眠
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        __CFRunLoopSetSleeping(rl);
        // ⚠️休眠，等待消息来唤醒线程
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
        __CFRunLoopUnsetSleeping(rl);
        // 通知 Observers：刚从休眠中唤醒
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

handle_msg:
        if (被 Timer 唤醒) {
            // 处理 Timer
            __CFRunLoopDoTimers(rl, rlm, mach_absolute_time())
        } else if (被 GCD 唤醒) {
            // 处理 GCD 
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
        } else {  // 被 Source1 唤醒   
            // 处理 Source1
            __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;  
        }

        // 处理 Blocks
        __CFRunLoopDoBlocks(rl, rlm);
        
        /* 设置返回值 */
        // 进入 loop 时参数为处理完事件就返回
        if (sourceHandledThisLoop && stopAfterHandle) {  
            retVal = kCFRunLoopRunHandledSource;
        // 超出传入参数标记的超时时间
        } else if (timeout_context->termTSR < mach_absolute_time()) {  
            retVal = kCFRunLoopRunTimedOut;
        // 被外部调用者强制停止
        } else if (__CFRunLoopIsStopped(rl)) {  
            __CFRunLoopUnsetStopped(rl);
            retVal = kCFRunLoopRunStopped;
        // 自动停止
        } else if (rlm->_stopped) {  
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped;
        // mode 中没有任何的 Source0/Source1/Timer/Observer
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {  
            retVal = kCFRunLoopRunFinished;
        }
    
    } while (0 == retVal);

    return retVal;
}
```
从该函数实现中可以得知`RunLoop`主要就做以下几件事情：
* __CFRunLoopDoObservers：通知`Observers`接下来要做什么
* __CFRunLoopDoBlocks：处理`Blocks`
* __CFRunLoopDoSources0：处理`Sources0`
* __CFRunLoopDoSources1：处理`Sources1`
* __CFRunLoopDoTimers：处理`Timers`
* 处理 GCD 相关：`dispatch_async(dispatch_get_main_queue(), ^{ });`
* __CFRunLoopSetSleeping/__CFRunLoopUnsetSleeping：休眠等待/结束休眠
* __CFRunLoopServiceMachPort -> mach-msg()：转移当前线程的控制权

![事件循环机制](https://user-gold-cdn.xitu.io/2020/2/27/170866a7cb070efe?w=980&h=1290&f=png&s=188859)



## __CFRunLoopServiceMachPort 函数实现：RunLoop 休眠的实现原理
在`__CFRunLoopRun`函数中，会调用`__CFRunLoopServiceMachPort`函数，该函数中调用了`mach_msg()`函数来转移当前线程的控制权给内核态/用户态。
* 没有消息需要处理时，休眠线程以避免资源占用。调用`mach_msg()`从用户态切换到内核态，等待消息；
* 有消息需要处理时，立刻唤醒线程，调用`mach_msg()`回到用户态处理消息。

这就是`RunLoop`休眠的实现原理，也是`RunLoop`与简单的`do...while`循环区别：
* `RunLoop`：休眠的时候，当前线程不会做任何事，CPU 不会再分配资源；
* 简单的`do...while`循环：当前线程并没有休息，一直占用 CPU 资源。

```objc
static Boolean __CFRunLoopServiceMachPort(mach_port_name_t port, mach_msg_header_t **buffer, size_t buffer_size, mach_port_t *livePort, mach_msg_timeout_t timeout, voucher_mach_msg_state_t *voucherState, voucher_t *voucherCopy) {
    Boolean originalBuffer = true;
    kern_return_t ret = KERN_SUCCESS;
    for (;;) {        /* In that sleep of death what nightmares may come ... */
        mach_msg_header_t *msg = (mach_msg_header_t *)*buffer;
        msg->msgh_bits = 0;
        msg->msgh_local_port = port;
        msg->msgh_remote_port = MACH_PORT_NULL;
        msg->msgh_size = buffer_size;
        msg->msgh_id = 0;
        if (TIMEOUT_INFINITY == timeout) { CFRUNLOOP_SLEEP(); } else { CFRUNLOOP_POLL(); }

        // ⚠️⚠️⚠️
        ret = mach_msg(msg, MACH_RCV_MSG|(voucherState ? MACH_RCV_VOUCHER : 0)|MACH_RCV_LARGE|((TIMEOUT_INFINITY != timeout) ? MACH_RCV_TIMEOUT : 0)|MACH_RCV_TRAILER_TYPE(MACH_MSG_TRAILER_FORMAT_0)|MACH_RCV_TRAILER_ELEMENTS(MACH_RCV_TRAILER_AV), 0, msg->msgh_size, port, timeout, MACH_PORT_NULL);

        // Take care of all voucher-related work right after mach_msg.
        // If we don't release the previous voucher we're going to leak it.
        voucher_mach_msg_revert(*voucherState);
        
        // Someone will be responsible for calling voucher_mach_msg_revert. This call makes the received voucher the current one.
        *voucherState = voucher_mach_msg_adopt(msg);
        
        if (voucherCopy) {
            if (*voucherState != VOUCHER_MACH_MSG_STATE_UNCHANGED) {
                // Caller requested a copy of the voucher at this point. By doing this right next to mach_msg we make sure that no voucher has been set in between the return of mach_msg and the use of the voucher copy.
                // CFMachPortBoost uses the voucher to drop importance explicitly. However, we want to make sure we only drop importance for a new voucher (not unchanged), so we only set the TSD when the voucher is not state_unchanged.
                *voucherCopy = voucher_copy();
            } else {
                *voucherCopy = NULL;
            }
        }

        CFRUNLOOP_WAKEUP(ret);
        if (MACH_MSG_SUCCESS == ret) {
            *livePort = msg ? msg->msgh_local_port : MACH_PORT_NULL;
            return true;
        }
        if (MACH_RCV_TIMED_OUT == ret) {
            if (!originalBuffer) free(msg);
            *buffer = NULL;
            *livePort = MACH_PORT_NULL;
            return false;
        }
        if (MACH_RCV_TOO_LARGE != ret) break;
        buffer_size = round_msg(msg->msgh_size + MAX_TRAILER_SIZE);
        if (originalBuffer) *buffer = NULL;
        originalBuffer = false;
        *buffer = realloc(*buffer, buffer_size);
    }
    HALT;
    return false;
}
```








