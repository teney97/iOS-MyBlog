## Runtime 系列文章
[深入浅出 Runtime（一）：初识](https://juejin.im/post/6844904071480999949)<br>
[深入浅出 Runtime（二）：数据结构](https://juejin.im/post/6844904072215003143)<br>
[深入浅出 Runtime（三）：消息机制](https://juejin.im/post/6844904072235974663)<br>
[深入浅出 Runtime（四）：super 的本质](https://juejin.im/post/6844904072252751880)<br>
[深入浅出 Runtime（五）：相关面试题](https://juejin.im/post/6844904072428912653)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c2f66d37c74452f8c43e6a243bb06e5~tplv-k3u1fbpfcp-zoom-1.image)


## 1. objc_msgSend 方法调用流程
在`OC`中调用一个方法时，编译器会根据情况调用以下函数中的一个进行消息传递：`objc_msgSend`、`objc_msgSend_stret`、`objc_msgSendSuper`、`objc_msgSendSuper_stret`。当方法调用者为`super`时会调用`objc_msgSendSuper`，当数据结构作为返回值时会调用`objc_msgSend_stret`或`objc_msgSendSuper_stret`。其他方法调用的情况都是转换为`objc_msgSend()`函数的调用。
```objc
void objc_msgSend(id _Nullable self, SEL _Nonnull op, ...)
```

给`receiver`（方法调用者/消息接收者）发送一条消息（`SEL`方法名）
* 参数 1 : `receiver`
* 参数 2 : `SEL`
* 参数 3、4、5... : `SEL`方法的参数

`objc_msgSend()`的执行流程可以分为 3 大阶段：
* 消息发送
* 动态方法解析
* 消息转发


&nbsp;
## 2. 消息发送

### “消息发送”流程
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80fc59fd622943a3a0f3e969a659ea05~tplv-k3u1fbpfcp-zoom-1.image)
![消息发送流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3232aa11fcfe4e6784a4958e2f499176~tplv-k3u1fbpfcp-zoom-1.image)



### 源码分析
在前面的文章说过，Runtime 是一个用`C、汇编`编写的运行时库。
在底层汇编里面如果需要调用 C 函数的话，苹果会为其加一个下划线_，
所以查看`objc_msgSend`函数的实现，需要搜索`_objc_msgSend`（objc-msg-arm64.s（objc4））。
```objc
// objc-msg-arm64.s（objc4）
    /*
       _objc_msgSend 函数实现
    */
    // ⚠️汇编程序入口格式为：ENTRY + 函数名
    ENTRY _objc_msgSend

    // ⚠️如果 receiver 为 nil 或者 tagged pointer，执行 LNilOrTagged，否则继续往下执行
    cmp    x0, #0            // nil check and tagged pointer check
    b.le    LNilOrTagged   

    // ⚠️通过 isa 找到 class/meta-class
    ldr    x13, [x0]        // x13 = isa
    and    x16, x13, #ISA_MASK    // x16 = class    
LGetIsaDone:
    // ⚠️进入 cache 缓存查找，传的参数为 NORMAL
    // CacheLookup 宏，用于在缓存中查找 SEL 对应方法实现
    CacheLookup NORMAL        // calls imp or objc_msgSend_uncached 

LNilOrTagged:
    // ⚠️如果 receiver 为 nil，执行 LReturnZero，结束 objc_msgSend
    b.eq    LReturnZero        // nil check 
    // ⚠️如果 receiver 为 tagged pointer，则执行其它
    ......
    b    LGetIsaDone

LReturnZero:
    ret  // 返回

    // ⚠️汇编中，函数的结束格式为：ENTRY + 函数名
    END_ENTRY _objc_msgSend



.macro CacheLookup
    // ⚠️根据 SEL 去哈希表 buckets 中查找方法
    // x1 = SEL, x16 = isa
    ldp    x10, x11, [x16, #CACHE]    // x10 = buckets, x11 = occupied|mask
    and    w12, w1, w11        // x12 = _cmd & mask
    add    x12, x10, x12, LSL #4    // x12 = buckets + ((_cmd & mask)<<4)

    ldp    x9, x17, [x12]        // {x9, x17} = *bucket
    // ⚠️缓存命中，进行 CacheHit 操作
1:    cmp    x9, x1            // if (bucket->sel != _cmd)
    b.ne    2f            //     scan more
    CacheHit $0            // call or return imp
    // ⚠️缓存中没有找到，进行 CheckMiss 操作
2:    // not hit: x12 = not-hit bucket
    CheckMiss $0            // miss if bucket->sel == 0
    cmp    x12, x10        // wrap if bucket == buckets
    b.eq    3f
    ldp    x9, x17, [x12, #-16]!    // {x9, x17} = *--bucket
    b    1b            // loop
3:    // wrap: x12 = first bucket, w11 = mask
    add    x12, x12, w11, UXTW #4    // x12 = buckets+(mask<<4)
    // Clone scanning loop to miss instead of hang when cache is corrupt.
    // The slow path may detect any corruption and halt later.
    ldp    x9, x17, [x12]        // {x9, x17} = *bucket
1:    cmp    x9, x1            // if (bucket->sel != _cmd)
    b.ne    2f            //     scan more
    CacheHit $0            // call or return imp    
2:    // not hit: x12 = not-hit bucket
    CheckMiss $0            // miss if bucket->sel == 0
    cmp    x12, x10        // wrap if bucket == buckets
    b.eq    3f
    ldp    x9, x17, [x12, #-16]!    // {x9, x17} = *--bucket
    b    1b            // loop
3:    // double wrap
    JumpMiss $0    
.endmacro


// CacheLookup NORMAL|GETIMP|LOOKUP
#define NORMAL 0
#define GETIMP 1
#define LOOKUP 2
.macro CacheHit
.if $0 == NORMAL  // ⚠️CacheLookup 传的参数是 NORMAL
    MESSENGER_END_FAST
    br    x17            // call imp  // ⚠️执行函数
.elseif $0 == GETIMP
    mov    x0, x17            // return imp
    ret
.elseif $0 == LOOKUP
    ret                // return imp via x17
.else
.abort oops
.endif
.endmacro


.macro CheckMiss
    // miss if bucket->sel == 0
.if $0 == GETIMP
    cbz    x9, LGetImpMiss
.elseif $0 == NORMAL  // ⚠️CacheLookup 传的参数是 NORMAL
    cbz    x9, __objc_msgSend_uncached  // ⚠️执行 __objc_msgSend_uncached
.elseif $0 == LOOKUP
    cbz    x9, __objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro


.macro JumpMiss
.if $0 == GETIMP
    b    LGetImpMiss
.elseif $0 == NORMAL
    b    __objc_msgSend_uncached
.elseif $0 == LOOKUP
    b    __objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro



    // ⚠️__objc_msgSend_uncached
    // ⚠️缓存中没有找到方法的实现，接下来去 MethodTableLookup 类的方法列表中查找
    STATIC_ENTRY __objc_msgSend_uncached
    MethodTableLookup NORMAL
    END_ENTRY __objc_msgSend_uncached

.macro MethodTableLookup
    blx    __class_lookupMethodAndLoadCache3  // ⚠️执行C函数 _class_lookupMethodAndLoadCache3
.endmacro
```
相反，通过汇编中函数名找对应 C 函数实现时，需要去掉一个下划线_。
```objc
// objc-runtime-new.mm（objc4）
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    // ⚠️注意传参，由于之前已经通过汇编去缓存中查找方法，所以这里不会再次到缓存中查找
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}

IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO;  // triedResolver 标记用于 动态方法解析

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup
    if (cache) {  // cache = NO，跳过 
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    // runtimeLock is held during isRealized and isInitialized checking
    // to prevent races against concurrent realization.

    // runtimeLock is held during method search to make
    // method-lookup + cache-fill atomic with respect to method addition.
    // Otherwise, a category could be added but ignored indefinitely because
    // the cache was re-filled with the old value after the cache flush on
    // behalf of the category.

    runtimeLock.read();

    if (!cls->isRealized()) {  // ⚠️如果 receiverClass(消息接受者类)  还未实现，就进行 realize 操作
        // Drop the read-lock and acquire the write-lock.
        // realizeClass() checks isRealized() again to prevent
        // a race while the lock is down.
        runtimeLock.unlockRead();
        runtimeLock.write();

        realizeClass(cls);

        runtimeLock.unlockWrite();
        runtimeLock.read();
    }

    // ⚠️如果 receiverClass 需要初始化且还未初始化，就进行初始化操作
    // 这里插入一个 +initialize 方法的知识点
    // 调用 _class_initialize(cls)，该函数中会递归遍历父类，判断父类是否存在且还未初始化 _class_initialize(cls->superclass)
    // 调用 callInitialize(cls) ，给 cls 发送一条 initialize 消息((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize)
    // 所以 +initialize 方法会在类第一次接收到消息时调用
    // 调用方式：objc_msgSend()
    // 调用顺序：先调用父类的 +initialize，再调用子类的 +initialize (先初始化父类，再初始化子类，每个类只会初始化1次）
    if (initialize  &&  !cls->isInitialized()) {
        runtimeLock.unlockRead();
        _class_initialize (_class_getNonMetaClass(cls, inst));
        runtimeLock.read();
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }


// ⚠️⚠️⚠️核心 
 retry:    
    runtimeLock.assertReading();

    // ⚠️去 receiverClass 的 cache 中查找方法，如果找到 imp 就直接调用
    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // ⚠️去 receiverClass 的 class_rw_t 中的方法列表查找方法，如果找到 imp 就调用并将该方法缓存到 receiverClass 的 cache 中
    {
        Method meth = getMethodNoSuper_nolock(cls, sel);  // ⚠️去目标类的方法列表中查找方法实现
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);  // ⚠️缓存方法
            imp = meth->imp;
            goto done;
        }
    }

    // ⚠️逐级查找父类的缓存和方法列表，如果找到 imp 就调用并将该方法缓存到 receiverClass 的 cache 中
    {
        unsigned attempts = unreasonableClassCount();
        for (Class curClass = cls->superclass;
             curClass != nil;
             curClass = curClass->superclass)
        {
            // Halt if there is a cycle in the superclass chain.
            if (--attempts == 0) {
                _objc_fatal("Memory corruption in class list.");
            }
            
            // Superclass cache.
            imp = cache_getImp(curClass, sel);
            if (imp) {
                if (imp != (IMP)_objc_msgForward_impcache) {
                    // Found the method in a superclass. Cache it in this class.
                    log_and_fill_cache(cls, imp, sel, inst, curClass);
                    goto done;
                }
                else {
                    // Found a forward:: entry in a superclass.
                    // Stop searching, but don't cache yet; call method 
                    // resolver for this class first.
                    break;
                }
            }
           
            // Superclass method list.
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                imp = meth->imp;
                goto done;
            }
        }
    }


    // ⚠️进入“动态方法解析”阶段
    // No implementation found. Try method resolver once.
    if (resolver  &&  !triedResolver) {
        runtimeLock.unlockRead();
        _class_resolveMethod(cls, sel, inst);
        runtimeLock.read();
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }


    // ⚠️进入“消息转发”阶段
    // No implementation found, and method resolver didn't help. 
    // Use forwarding.
    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);


 done:
    runtimeLock.unlockRead();

    return imp;
}
```
我们来看一下`getMethodNoSuper_nolock(cls, sel)`是怎么从类中查找方法实现的
* 如果方法列表是经过排序的，则进行二分查找;
* 如果方法列表没有进行排序，则进行线性遍历查找。
```
static method_t *
getMethodNoSuper_nolock(Class cls, SEL sel)
{
    runtimeLock.assertLocked();
    assert(cls->isRealized());
    // fixme nil cls? 
    // fixme nil sel?
    for (auto mlists = cls->data()->methods.beginLists(), 
              end = cls->data()->methods.endLists(); 
         mlists != end;
         ++mlists)
    {
    // ⚠️核心函数 search_method_list()
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;
    }
    return nil;
}


static method_t *search_method_list(const method_list_t *mlist, SEL sel)
{
    int methodListIsFixedUp = mlist->isFixedUp();
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);
    
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) {
        // ⚠️如果方法列表是经过排序的，则进行二分查找
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        // ⚠️如果方法列表没有进行排序，则进行线性遍历查找
        // Linear search of unsorted method list
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }
    ......
    return nil;
}


static method_t *findMethodInSortedMethodList(SEL key, const method_list_t *list)
{
    assert(list);

    const method_t * const first = &list->first;
    const method_t *base = first;
    const method_t *probe;
    uintptr_t keyValue = (uintptr_t)key;
    uint32_t count;
    // ⚠️count >>= 1 二分查找
    for (count = list->count; count != 0; count >>= 1) {
        probe = base + (count >> 1);
        
        uintptr_t probeValue = (uintptr_t)probe->name;
        
        if (keyValue == probeValue) {
            // `probe` is a match.
            // Rewind looking for the *first* occurrence of this value.
            // This is required for correct category overrides.
            while (probe > first && keyValue == (uintptr_t)probe[-1].name) {
                probe--;
            }
            return (method_t *)probe;
        }
        
        if (keyValue > probeValue) {
            base = probe + 1;
            count--;
        }
    }
    
    return nil;
}
```
我们来看一下`log_and_fill_cache(cls, meth->imp, sel, inst, cls)`是怎么缓存方法的
```objc
/***********************************************************************
* log_and_fill_cache
* Log this method call. If the logger permits it, fill the method cache.
* cls is the method whose cache should be filled. 
* implementer is the class that owns the implementation in question.
**********************************************************************/
static void
log_and_fill_cache(Class cls, IMP imp, SEL sel, id receiver, Class implementer)
{
#if SUPPORT_MESSAGE_LOGGING
    if (objcMsgLogEnabled) {
        bool cacheIt = logMessageSend(implementer->isMetaClass(), 
                                      cls->nameForLogging(),
                                      implementer->nameForLogging(), 
                                      sel);
        if (!cacheIt) return;
    }
#endif
    cache_fill (cls, sel, imp, receiver);
}

#if TARGET_OS_WIN32  ||  TARGET_OS_EMBEDDED
#   define SUPPORT_MESSAGE_LOGGING 0
#else
#   define SUPPORT_MESSAGE_LOGGING 1
#endif
```
`cache_fill()`函数实现已经在上一篇文章中写到。

关于缓存查找流程和更多`cache_t`的知识，可以查看：[深入浅出 Runtime（二）：数据结构](https://juejin.im/post/6844904072215003143)



## 2. 动态方法解析
### “动态方法解析”流程


![动态方法解析流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f862586c2f0449d0b04f4da3a5c38de6~tplv-k3u1fbpfcp-zoom-1.image)


如果“消息发送”阶段未找到方法的实现，进行一次“动态方法解析”；

“动态方法解析”后，会再次进入“消息发送”流程，从“去 receiverClass 的 cache 中查找方法”这一步开始执行。

我们可以根据方法类型（实例方法 or 类方法）重写以下方法

```objectivec
+(BOOL)resolveInstanceMethod:(SEL)sel;
+(BOOL)resolveClassMethod:(SEL)sel;
```

在方法中调用以下函数来动态添加方法的实现
```objectivec
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
```

示例代码如下，我们分别调用了 HTPerson 的`eat`实例方法和类方法，而 HTPerson.m 文件中并没有这两个方法的对应实现，我们为这两个方法动态添加了实现，输出结果如下。

```objc
// main.m
#import <Foundation/Foundation.h>
#import "HTPerson.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [[HTPerson new] eat];
        [HTPerson eat];   
    }
    return 0;
}
@end

// HTPerson.h
#import <Foundation/Foundation.h>
@interface HTPerson : NSObject
- (void)eat;  // 没有对应实现
- (void)sleep;
+ (void)eat;  // 没有对应实现
+ (void)sleep;
@end

// HTPerson.m
#import "HTPerson.h"
#import <objc/runtime.h>
@implementation HTPerson
- (void)sleep
{
    NSLog(@"%s",__func__);
}
+ (void)sleep
{
    NSLog(@"%s",__func__);
}

+ (BOOL)resolveInstanceMethod:(SEL)sel
{
    if (sel == @selector(eat)) {
        
        // 获取其它方法， Method 就是指向 method_t 结构体的指针
        Method method = class_getInstanceMethod(self, @selector(sleep));
        /*
         ** 参数1:给哪个类添加
         ** 参数2:给哪个方法添加
         ** 参数3:方法的实现地址
         ** 参数4:方法的编码类型
         */
        class_addMethod(self,  // 实例方法存放在类对象中，所以这里要传入类对象
                        sel,
                        method_getImplementation(method),
                        method_getTypeEncoding(method)
                        );
        // 返回 YES 代表有动态添加方法实现
        // 从源码来看，该返回值只是用来打印解析结果相关信息，并不影响动态方法解析的结果
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

+ (BOOL)resolveClassMethod:(SEL)sel
{
    if (sel == @selector(eat)) {
        
        Method method = class_getClassMethod(object_getClass(self), @selector(sleep));

        class_addMethod(object_getClass(self),  // 类方法存放在元类对象中，所以这里要传入元类对象
                        sel,
                        method_getImplementation(method),
                        method_getTypeEncoding(method)
                        );
        return YES;
    }
    return [super resolveClassMethod:sel];
}

@end

```
> -[HTPerson sleep]<br>
> +[HTPerson sleep]


### 源码分析
```objc
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO;
    ......   

 retry:    
    ......

    // ⚠️如果“消息发送”阶段未找到方法的实现，进行一次“动态方法解析”
    if (resolver  &&  !triedResolver) {
        runtimeLock.unlockRead();
        _class_resolveMethod(cls, sel, inst);  // ⚠️核心函数
        runtimeLock.read();
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;  // ⚠️标记triedResolver为YES
        goto retry;  // ⚠️再次进入消息发送，从“去 receiverClass 的 cache 中查找方法”这一步开始
    }

    // ⚠️进入“消息转发”阶段
    ......
}

// objc-class.mm（objc4）
void _class_resolveMethod(Class cls, SEL sel, id inst)
{
    // ⚠️判断是 class 对象还是 meta-class 对象
    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]
        // ⚠️核心函数
        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        // ⚠️核心函数
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}

/***********************************************************************
* _class_resolveInstanceMethod
* Call +resolveInstanceMethod, looking for a method to be added to class cls.
* cls may be a metaclass or a non-meta class.
* Does not check if the method already exists.
**********************************************************************/
static void _class_resolveInstanceMethod(Class cls, SEL sel, id inst)
{
    // ⚠️查看 receiverClass 的 meta-class 对象的方法列表里面是否有 SEL_resolveInstanceMethod 函数 imp
    // ⚠️也就是看我们是否实现了 +(BOOL)resolveInstanceMethod:(SEL)sel 方法
    // ⚠️这里一定会找到该方法实现，因为 NSObject 中有实现
    if (! lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // ⚠️如果没找到，说明程序异常，直接返回
        // Resolver not implemented.
        return;
    }

    // ⚠️如果找到了，通过 objc_msgSend 给对象发送一条 SEL_resolveInstanceMethod 消息
    // ⚠️即调用一下 +(BOOL)resolveInstanceMethod:(SEL)sel 方法
    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

    // ⚠️下面是解析结果的一些打印信息
    ......
}

/***********************************************************************
* _class_resolveClassMethod
* Call +resolveClassMethod, looking for a method to be added to class cls.
* cls should be a metaclass.
* Does not check if the method already exists.
**********************************************************************/
static void _class_resolveClassMethod(Class cls, SEL sel, id inst)
{
    assert(cls->isMetaClass());
    // ⚠️查看 receiverClass 的 meta-class 对象的方法列表里面是否有 SEL_resolveClassMethod 函数 imp
    // ⚠️也就是看我们是否实现了 +(BOOL)resolveClassMethod:(SEL)sel 方法
    // ⚠️这里一定会找到该方法实现，因为 NSObject 中有实现
    if (! lookUpImpOrNil(cls, SEL_resolveClassMethod, inst, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // ⚠️如果没找到，说明程序异常，直接返回
        // Resolver not implemented.
        return;
    }
    // ⚠️如果找到了，通过 objc_msgSend 给对象发送一条 SEL_resolveClassMethod 消息
    // ⚠️即调用一下 +(BOOL)resolveClassMethod:(SEL)sel 方法
    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(_class_getNonMetaClass(cls, inst),   // 该函数返回值是类对象，而非元类对象
                        SEL_resolveClassMethod, sel);

    // ⚠️下面是解析结果的一些打印信息
    ......
}
```


## 3. 消息转发
### “消息转发”流程

![消息转发流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7edba290b72d4d2eab0091628a6fb2bf~tplv-k3u1fbpfcp-zoom-1.image)



如果“消息发送”阶段未找到方法的实现，且通过“动态方法解析”没有解决，就进入“消息转发”阶段；

“消息转发”阶段分两步进行：Fast forwarding 和 Normal forwarding，顾名思义，第一步速度要比第二步快；
* Fast forwarding：将消息转发给一个其它 OC 对象（找一个备用接收者），我们可以重写以下方法，返回一个`!= receiver`的对象，来完成这一步骤；
    `+/- (id)forwardingTargetForSelector:(SEL)sel`
* Normal forwarding：实现一个完整的消息转发过程，
如果上一步没能解决未知消息，可以重写以下两个方法启动完整的消息转发。

**① 第一个方法**：我们需要在该方法中返回一个适合该未知消息的方法签名（方法签名就是对返回值类型、参数类型的描述，可以使用 Type Encodings 编码，关于 Type Encodings 可以阅读我的上一篇 blog [深入浅出 Runtime（二）：数据结构](https://juejin.im/post/6844904072215003143)）。
```objectivec
+/- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
```
Runtime 会根据这个方法签名，创建一个`NSInvocation`对象（`NSInvocation`封装了未知消息的全部内容，包括：方法调用者 target、方法名 selector、方法参数 argument 等），然后调用第二个方法并将该`NSInvocation`对象作为参数传入。


**② 第二个方法**：我们可以在该方法中：将未知消息转发给其它对象；改变未知消息的内容(如方法名、方法参数)再转发给其它对象；甚至可以定义任何逻辑。
```objectivec
+/- (void)forwardInvocation:(NSInvocation *)invocation
```
如果第一个方法中没有返回方法签名，或者我们没有重写第二个方法，系统就会认为我们彻底不想处理这个消息了，这时候就会调用`+/- (void)doesNotRecognizeSelector:(SEL)sel`方法并抛出经典的 crash:`unrecognized selector sent to instance/class`，结束 objc_msgSend 的全部流程。

下面我们来看一下这几个代码的默认实现：
```objc
// NSObject.mm
+ (id)forwardingTargetForSelector:(SEL)sel {
    return nil;
}
+ (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    _objc_fatal("+[NSObject methodSignatureForSelector:] "
                "not available without CoreFoundation");
}
+ (void)forwardInvocation:(NSInvocation *)invocation {
    [self doesNotRecognizeSelector:(invocation ? [invocation selector] : 0)];
}
+ (void)doesNotRecognizeSelector:(SEL)sel {
    _objc_fatal("+[%s %s]: unrecognized selector sent to instance %p", 
                class_getName(self), sel_getName(sel), self);
}
```

**Fast forwarding 示例代码如下：**

我们调用了 HTPerson 的`eat`实例方法，而 HTPerson.m 文件中并没有该方法的对应实现，HTDog.m 中有同名方法的实现，我们将消息转发给 HTDog 的实例对象，输出结果如下。
```objc
// main.m
#import <Foundation/Foundation.h>
#import "HTPerson.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [[HTPerson new] eat];        
    }
    return 0;
}
@end

// HTPerson.h
#import <Foundation/Foundation.h>
@interface HTPerson : NSObject
- (void)eat;  // 没有对应实现
@end

// HTPerson.m
#import "HTPerson.h"
#import "HTDog.h"
@implementation HTPerson
- (id)forwardingTargetForSelector:(SEL)aSelector
{
    if (aSelector == @selector(eat)) {
        return [HTDog new];  // 将 eat 消息转发给 HTDog 的实例对象
//        return [HTDog class];  // 还可以将 eat 消息转发给 HTDog 的类对象
    }
    return [super forwardingTargetForSelector:aSelector];
}

@end

// HTDog.m
#import "HTDog.h"
@implementation HTDog
- (void)eat
{
    NSLog(@"%s",__func__);
}
+ (void)eat
{
    NSLog(@"%s",__func__);
}
@end
```
>-[HTDog eat]

**Normal forwarding 示例代码及输出结果如下：**

```objc
// HTPerson.m
#import "HTPerson.h"
#import "HTDog.h"

@implementation HTPerson

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    if (aSelector == @selector(eat)) {
        
        return [[HTDog new] methodSignatureForSelector:aSelector];
        //return [NSMethodSignature signatureWithObjCTypes:"v@:i"];
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    // 将未知消息转发给其它对象
    [anInvocation invokeWithTarget:[HTDog new]];
    
    // 改变未知消息的内容(如方法名、方法参数)再转发给其它对象
    /*
    anInvocation.selector = @selector(sleep);
    anInvocation.target = [HTDog new];
    int age;
    [anInvocation getArgument:&age atIndex:2];  // 参数顺序：target、selector、other arguments
    [anInvocation setArgument:&age atIndex:2];  // 参数的个数由上个方法返回的方法签名决定，要注意数组越界问题
    [anInvocation invoke];
    
    int ret;
    [anInvocation getReturnValue:&age];  // 获取返回值
     */
    
    // 定义任何逻辑，如：只打印一句话
    /*
     NSLog(@"好好学习");
     */
}

@end
```
> -[HTDog eat]


### 源码分析
```objc
// objc-runtime-new.mm（objc4）
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    ......

    // ⚠️如果“消息发送”阶段未找到方法的实现，且通过“动态方法解析”没有解决
    // ⚠️进入“消息转发”阶段
    imp = (IMP)_objc_msgForward_impcache;  // 进入汇编
    cache_fill(cls, sel, imp, inst);       // 缓存方法

    ......
}
```
```objc
// objc-msg-arm64.s（objc4）
    STATIC_ENTRY __objc_msgForward_impcache
    b    __objc_msgForward
    END_ENTRY __objc_msgForward_impcache
    
    ENTRY __objc_msgForward
    adrp    x17, __objc_forward_handler@PAGE   // ⚠️执行C函数 _objc_forward_handler
    ldr    x17, [x17, __objc_forward_handler@PAGEOFF]
    br    x17
    END_ENTRY __objc_msgForward
```
```objc
// objc-runtime.mm（objc4）
// Default forward handler halts the process.
__attribute__((noreturn)) void 
objc_defaultForwardHandler(id self, SEL sel)
{
    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "
                "(no message forward handler is installed)", 
                class_isMetaClass(object_getClass(self)) ? '+' : '-', 
                object_getClassName(self), sel_getName(sel), self);
}
void *_objc_forward_handler = (void*)objc_defaultForwardHandler;
```
可以看到`_objc_forward_handler`是一个函数指针，指向`objc_defaultForwardHandler()`，该函数只是打印信息。由于苹果没有对此开源，我们无法再深入探索关于“消息转发”的详细执行逻辑。

我们知道，如果调用一个没有实现的方法，并且没有进行“动态方法解析”和“消息转发”处理，会报经典的 crash：`unrecognized selector sent to instance/class`。我们查看 crash 打印信息的函数调用栈，如下，可以看的系统调用了一个叫`___forwarding___`的函数。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e96189c102443e9bdfb33ea35ec3b69~tplv-k3u1fbpfcp-zoom-1.image)

该函数是 CoreFoundation 框架中的，苹果对此函数尚未开源，我们可以打断点进入该函数的汇编实现。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bbf3c3452913406dba4a3df1d9a793cf~tplv-k3u1fbpfcp-zoom-1.image)


以下是从网上找到的`___forewarding___`的 C 语言伪代码实现。
```objc
// 伪代码
int __forwarding__(void *frameStackPointer, int isStret) {
    id receiver = *(id *)frameStackPointer;
    SEL sel = *(SEL *)(frameStackPointer + 8);
    const char *selName = sel_getName(sel);
    Class receiverClass = object_getClass(receiver);

    // ⚠️⚠️⚠️调用 forwardingTargetForSelector:
    if (class_respondsToSelector(receiverClass, @selector(forwardingTargetForSelector:))) {
        id forwardingTarget = [receiver forwardingTargetForSelector:sel];
        // ⚠️判断该方法是否返回了一个对象且该对象 != receiver
        if (forwardingTarget && forwardingTarget != receiver) {
            if (isStret == 1) {
                int ret;
                objc_msgSend_stret(&ret,forwardingTarget, sel, ...);
                return ret;
            }
            //⚠️objc_msgSend(返回值, sel, ...);
            return objc_msgSend(forwardingTarget, sel, ...);
        }
    }

    // 僵尸对象
    const char *className = class_getName(receiverClass);
    const char *zombiePrefix = "_NSZombie_";
    size_t prefixLen = strlen(zombiePrefix); // 0xa
    if (strncmp(className, zombiePrefix, prefixLen) == 0) {
        CFLog(kCFLogLevelError,
              @"*** -[%s %s]: message sent to deallocated instance %p",
              className + prefixLen,
              selName,
              receiver);
        <breakpoint-interrupt>
    }

    // ⚠️⚠️⚠️调用 methodSignatureForSelector 获取方法签名后再调用 forwardInvocation
    if (class_respondsToSelector(receiverClass, @selector(methodSignatureForSelector:))) {
        // ⚠️调用 methodSignatureForSelector 获取方法签名
        NSMethodSignature *methodSignature = [receiver methodSignatureForSelector:sel];
        // ⚠️判断返回值是否为 nil
        if (methodSignature) {
            BOOL signatureIsStret = [methodSignature _frameDescriptor]->returnArgInfo.flags.isStruct;
            if (signatureIsStret != isStret) {
                CFLog(kCFLogLevelWarning ,
                      @"*** NSForwarding: warning: method signature and compiler disagree on struct-return-edness of '%s'.  Signature thinks it does%s return a struct, and compiler thinks it does%s.",
                      selName,
                      signatureIsStret ? "" : not,
                      isStret ? "" : not);
            }
            if (class_respondsToSelector(receiverClass, @selector(forwardInvocation:))) {

                // ⚠️根据方法签名创建一个 NSInvocation 对象
                NSInvocation *invocation = [NSInvocation _invocationWithMethodSignature:methodSignature frame:frameStackPointer];

                // ⚠️调用 forwardInvocation
                [receiver forwardInvocation:invocation];

                void *returnValue = NULL;
                [invocation getReturnValue:&value];
                return returnValue;
            } else {
                CFLog(kCFLogLevelWarning ,
                      @"*** NSForwarding: warning: object %p of class '%s' does not implement forwardInvocation: -- dropping message",
                      receiver,
                      className);
                return 0;
            }
        }
    }

    SEL *registeredSel = sel_getUid(selName);

    // selector 是否已经在 Runtime 注册过
    if (sel != registeredSel) {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: selector (%p) for message '%s' does not match selector known to Objective C runtime (%p)-- abort",
              sel,
              selName,
              registeredSel);
    } // ⚠️⚠️⚠️调用 doesNotRecognizeSelector 
    else if (class_respondsToSelector(receiverClass,@selector(doesNotRecognizeSelector:))) {

        [receiver doesNotRecognizeSelector:sel];
    }
    else {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: object %p of class '%s' does not implement doesNotRecognizeSelector: -- abort",
              receiver,
              className);
    }

    // The point of no return.
    kill(getpid(), 9);
}
```

&nbsp;
## 总结

至此，`objc_msgSend`方法调用流程就已经讲解结束了。
下面来做一个小总结。

### objc_msgSend 执行流程图
![objc_msgSend流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3bbae4476224db5ac1678db4ec1e2eb~tplv-k3u1fbpfcp-zoom-1.image)

