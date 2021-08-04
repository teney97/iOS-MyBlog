
### 走进苹果源码分析内存管理方法的实现

前面我们只是讲解了内存管理方法的使用以及使用注意，那么这些方法的内部实现到底是怎样的？引用计数具体又是怎样管理的呢？接下来我们走进`Runtime`最新源码`objc4-779.1`（写该文章时的最新），分析`alloc`、`retainCount`、`retain`、`release`、`dealloc`等方法的实现。

源码下载地址：[https://opensource.apple.com/tarballs/objc4/](https://opensource.apple.com/tarballs/objc4/)

### alloc
`alloc`方法的函数调用栈为：
```objc
// NSObject.mm
① objc_alloc
② callAlloc
// objc-runtime-new.mm
③ _objc_rootAllocWithZone
④ _class_createInstanceFromZone
⑤ calloc、
// objc-object.h
  initInstanceIsa->initIsa
```

#### ① objc_alloc
```objc
// Calls [cls alloc].
id
objc_alloc(Class cls)
{
    return callAlloc(cls, true/*checkNil*/, false/*allocWithZone*/);
}
```
#### ② callAlloc
```objc
// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
// 调用 [cls alloc] or [cls allocWithZone:nil] 会来到这个函数，使用适当的快捷方式优化
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
// 如果是 __OBJC2__ 代码（判断当前语言是否是 Objective-C 2.0）
#if __OBJC2__ 
    // 如果 (checkNil && !cls)，直接返回 nil
    if (slowpath(checkNil && !cls)) return nil;  
    // 如果 cls 没有实现自定义 allocWithZone 方法，调用 _objc_rootAllocWithZone
    if (fastpath(!cls->ISA()->hasCustomAWZ())) { 
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif
       
    // No shortcuts available.    
    // 没有可用的快捷方式
    // 如果 allocWithZone 为 true，给 cls 发送 allocWithZone:nil 消息
    if (allocWithZone) { 
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    // 否则发送 alloc 消息
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc)); 
}
```
> **备注：** slowpath & fastpath<br>
>这两个宏的定义如下：
>```objc
>#define fastpath(x) (__builtin_expect(bool(x), 1))
>#define slowpath(x) (__builtin_expect(bool(x), 0))
>```
>它们都使用了`__builtin_expect()`：
>```c
>long __builtin_expect(long exp, long c);
>```
>`__builtin_expect()`是 GCC (version >= 2.96）提供给程序员使用的，由于大部分程序员在分支预测方面做得很糟糕，所以 GCC 提供这个内建函数来帮助程序员处理分支预测，目的是将 “分支转移” 的信息提供给编译器，这样编译器可以对代码进行优化，以减少指令跳转带来的性能下降。它的意思是：`exp == c`的概率很大。<br>`fastpath(x)`表示`x`为`1`的概率很大，`slowpath(x)`表示`x`为`0`的概率很大。它和`if`一起使用，`if (fastpath(x))`表示执行`if`语句的可能性大，`if (slowpath(x))`表示执行`if`语句的可能性小。

`callAlloc`函数中`主要`执行以下步骤：

1. 判断类有没有实现自定义`allocWithZone`方法，如果没有，就调用`_objc_rootAllocWithZone`函数（这属于快捷方式）。
2. 如果不能使用快捷方式（即第 1 步条件不成立），根据`allocWithZone`的值给`cls`类发送消息。由于`allocWithZone`传的`false`，则给`cls`发送`alloc`消息。

我们先来看一下第二种情况，就是给`cls`发送`alloc`消息。
```objc
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```
```objc
// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
```
小朋友，你是否有很多问号？它怎么又调用了`callAlloc`？但不同的是，这次传参不一样：
* `checkNil`为`false`，`checkNil`作用是是否需要判空，由于第一次调用该函数时已经进行判空操作了，所以这次传`false`。
* `allocWithZone`为`true`，所以接下来会给对象发送`allocWithZone:nil`消息。
```objc
// Replaced by ObjectAlloc
+ (id)allocWithZone:(struct _NSZone *)zone {
    return _objc_rootAllocWithZone(self, (malloc_zone_t *)zone);
}
```
可以看到，第一种（快捷方式）和第二种（非快捷方式）调用的都是`_objc_rootAllocWithZone`函数，且传参都是`cls`和`nil`。
> **备注：** 在 ARC 下 NSZone 已被忽略。<br>
在`《iOS - 老生常谈内存管理（三）：ARC 面世 —— ARC 实施新规则》`章节中已经提到，对于现在的运行时系统（编译器宏 __ OBJC2 __ 被设定的环境），不管是`MRC`还是`ARC`下，区域（`NSZone`）都已单纯地被忽略。所以现在`allocWithZone`和`alloc`方法已经没有区别。



#### ③ _objc_rootAllocWithZone
```objc
// objc-runtime-new.mm
NEVER_INLINE
id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone __unused)
{
    // allocWithZone under __OBJC2__ ignores the zone parameter
    // allocWithZone 在 __OBJC2__ 下忽略 zone 参数
    return _class_createInstanceFromZone(cls, 0, nil,
                                         OBJECT_CONSTRUCT_CALL_BADALLOC);
}
```
该函数中调用了`_class_createInstanceFromZone`函数，可以发现，参数`zone`已被忽略，直接传`nil`。


#### ④ _class_createInstanceFromZone
```objc
/***********************************************************************
* class_createInstance
* fixme
* Locking: none
*
* Note: this function has been carefully written so that the fastpath
* takes no branch.
**********************************************************************/
static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor(); // 获取 cls 是否有构造函数
    bool hasCxxDtor = cls->hasCxxDtor();                 // 获取 cls 是否有析构函数
    bool fast = cls->canAllocNonpointer();               // 获取 cls 是否可以分配 nonpointer，如果是的话代表开启了内存优化 
    size_t size;

    // 获取需要申请的空间大小
    size = cls->instanceSize(extraBytes);  
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    // zone == nil，调用 calloc 来申请内存空间
    if (zone) { 
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {    
        obj = (id)calloc(1, size);
    }
    // 如果内存空间申请失败，调用 callBadAllocHandler
    if (slowpath(!obj)) { 
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    // 初始化 isa。如果是 nonpointer，就调用 initInstanceIsa
    if (!zone && fast) { 
        obj->initInstanceIsa(cls, hasCxxDtor); 
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    // 如果 cls 没有构造函数，直接返回对象
    if (fastpath(!hasCxxCtor)) {
        return obj;
    }
    // 进行构造函数的处理，再返回
    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```
在`_class_createInstanceFromZone`函数中，通过调用 C 函数`calloc`来申请内存空间，并初始化对象的`isa`。


接着我们来看一下初始化对象`isa`(`nonpointer`)的过程。

#### ⑤ initInstanceIsa
```objc
// objc-object.h
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    ASSERT(!cls->instancesRequireRawIsa());
    ASSERT(hasCxxDtor == cls->hasCxxDtor());

    initIsa(cls, true, hasCxxDtor);
}
```

#### initIsa
```objc
// objc-config.h
// Define SUPPORT_INDEXED_ISA=1 on platforms that store the class in the isa 
// field as an index into a class table.
// Note, keep this in sync with any .s files which also define it.
// Be sure to edit objc-abi.h as well.
#if __ARM_ARCH_7K__ >= 2  ||  (__arm64__ && !__LP64__)
#   define SUPPORT_INDEXED_ISA 1
#else
#   define SUPPORT_INDEXED_ISA 0
#endif

// objc-object.h
inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    ASSERT(!isTaggedPointer()); 
    
    if (!nonpointer) {
        isa = isa_t((uintptr_t)cls);
    } else {
        ASSERT(!DisableNonpointerIsa);
        ASSERT(!cls->instancesRequireRawIsa());

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA  // 对于 64 位系统，该值为 0
        ASSERT(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;  
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;  // 将 isa 的 bits 赋值为 ISA_MAGIC_VALUE
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa;
    }
}
```
在`initIsa`方法中将`isa`的`bits`赋值为`ISA_MAGIC_VALUE`。源码注释写的是`ISA_MAGIC_VALUE`初始化了`isa`的`magic`和`nonpointer`字段，下面我们加以验证。
```objc
#if SUPPORT_PACKED_ISA

    // extra_rc must be the MSB-most field (so it matches carry/overflow flags)
    // nonpointer must be the LSB (fixme or get rid of it)
    // shiftcls must occupy the same bits that a real class pointer would
    // bits + RC_ONE is equivalent to extra_rc + 1
    // RC_HALF is the high bit of extra_rc (i.e. half of its range)

    // future expansion:
    // uintptr_t fast_rr : 1;     // no r/r overrides
    // uintptr_t lock : 2;        // lock for atomic property, @synch
    // uintptr_t extraBytes : 1;  // allocated with extra bytes

# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL  // here
#   define ISA_BITFIELD                                                      \
      uintptr_t nonpointer        : 1;                                       \
      uintptr_t has_assoc         : 1;                                       \
      uintptr_t has_cxx_dtor      : 1;                                       \
      uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
      uintptr_t magic             : 6;                                       \
      uintptr_t weakly_referenced : 1;                                       \
      uintptr_t deallocating      : 1;                                       \
      uintptr_t has_sidetable_rc  : 1;                                       \
      uintptr_t extra_rc          : 19
#   define RC_ONE   (1ULL<<45)
#   define RC_HALF  (1ULL<<18)

# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
#   define ISA_BITFIELD                                                        \
      uintptr_t nonpointer        : 1;                                         \
      uintptr_t has_assoc         : 1;                                         \
      uintptr_t has_cxx_dtor      : 1;                                         \
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ \
      uintptr_t magic             : 6;                                         \
      uintptr_t weakly_referenced : 1;                                         \
      uintptr_t deallocating      : 1;                                         \
      uintptr_t has_sidetable_rc  : 1;                                         \
      uintptr_t extra_rc          : 8
#   define RC_ONE   (1ULL<<56)
#   define RC_HALF  (1ULL<<7)

# else
#   error unknown architecture for packed isa
# endif

// SUPPORT_PACKED_ISA
#endif
```
在`__arm64__`下，`ISA_MAGIC_VALUE`的值为`0x000001a000000001ULL`。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3070c9b894044e88bcbc91be2de5819a~tplv-k3u1fbpfcp-zoom-1.image)

对应到`ISA_BITFIELD`中，`ISA_MAGIC_VALUE`确实是用于初始化`isa`的`magic`和`nonpointer`字段。

在初始化`isa`的时候，并没有对`extra_rc`进行操作。也就是说`alloc`方法实际上并没有设置对象的引用计数值为 1。
> **Why?** alloc 居然没有让引用计数值为 1？<br>不急，我们先留着疑问分析其它内存管理方法。

>**小结：** `alloc`方法经过一系列的函数调用栈，最终通过调用 C 函数`calloc`来申请内存空间，并初始化对象的`isa`，但并没有设置对象的引用计数值为 1。

### init
```objc
// NSObject.mm
// Calls [[cls alloc] init].
id
objc_alloc_init(Class cls)
{
    return [callAlloc(cls, true/*checkNil*/, false/*allocWithZone*/) init];
}

- (id)init {
    return _objc_rootInit(self);
}

id
_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```
基类的`init`方法啥都没干，只是将`alloc`创建的对象返回。我们可以重写`init`方法来对`alloc`创建的实例做一些初始化操作。


### new
```objc
// Calls [cls new]
id
objc_opt_new(Class cls)
{
#if __OBJC2__
    if (fastpath(cls && !cls->ISA()->hasCustomCore())) {
        return [callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/) init];
    }
#endif
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(new));
}

+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```
`new`方法很简单，只是嵌套了`alloc`和`init`。

### copy & mutableCopy
```objc
- (id)copy {
    return [(id)self copyWithZone:nil];
}

- (id)mutableCopy {
    return [(id)self mutableCopyWithZone:nil];
}
```
`copy`和`mutableCopy`也很简单，只是调用了`copyWithZone`和`mutableCopyWithZone`方法。

### retainCount

我们都知道，`retainCount`方法是取出对象的引用计数值。那么，它是从哪里取值，怎么取值的呢？相信你们已经想到了，`isa`和`Sidetable`，下面我们进入源码看看它的取值过程。

`retainCount`方法的函数调用栈为：
```objc
// NSObject.mm
① retainCount
② _objc_rootRetainCount
// objc-object.h
③ objc_object::rootRetainCount
// NSObject.mm
④ objc_object::sidetable_getExtraRC_nolock
objc_object::sidetable_retainCount
```


#### ① retainCount
```objc
- (NSUInteger)retainCount {
    return _objc_rootRetainCount(self);
}
```
#### ② _objc_rootRetainCount
```objc
uintptr_t
_objc_rootRetainCount(id obj)
{
    ASSERT(obj);

    return obj->rootRetainCount();
}
```
#### ③ objc_object::rootRetainCount
```objc
inline uintptr_t 
objc_object::rootRetainCount()
{
    // 如果是 tagged pointer，直接返回 this
    if (isTaggedPointer()) return (uintptr_t)this; 

    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits); // 获取 isa 
    ClearExclusive(&isa.bits);
    // 如果 isa 是 nonpointer
    if (bits.nonpointer) { 
        uintptr_t rc = 1 + bits.extra_rc; // 引用计数 = 1 + isa 中 extra_rc 的值
        // 如果还额外使用 sidetable 存储引用计数
        if (bits.has_sidetable_rc) { 
            rc += sidetable_getExtraRC_nolock(); // 加上 sidetable 中引用计数的值
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    // 如果 isa 不是 nonpointer，返回 sidetable_retainCount() 的值
    return sidetable_retainCount(); 
}
```
#### ④ objc_object::sidetable_getExtraRC_nolock 
```objc
size_t 
objc_object::sidetable_getExtraRC_nolock()
{
    ASSERT(isa.nonpointer);
    SideTable& table = SideTables()[this];               // 获得 SideTable
    RefcountMap::iterator it = table.refcnts.find(this); // 获得 refcnts
    if (it == table.refcnts.end()) return 0;       // 如果没找到，返回 0
    else return it->second >> SIDE_TABLE_RC_SHIFT; // 如果找到了，通过 SIDE_TABLE_RC_SHIFT 位掩码获取对应的引用计数
}

#define SIDE_TABLE_RC_SHIFT 2
```
如果`isa`是`nonpointer`，则对象的引用计数就存储在它的`isa_t`的`extra_rc`中以及`SideTable`的`RefCountMap`中。由于`extra_rc`存储的对象本身之外的引用计数值，所以需要加上对象本身的引用计数 1；再加上`SideTable`中存储的引用计数值，通过`sidetable_getExtraRC_nolock()`函数获取。



`sidetable_getExtraRC_nolock()`函数中进行了两次哈希查找：
* ① 第一次根据当前对象的内存地址，经过哈希查找从`SideTables()`中取出它所在的`SideTable`；
* ② 第二次根据当前对象的内存地址，经过哈希查找从`SideTable`中的`refcnts`中取出它的引用计数表。

#### objc_object::sidetable_retainCount
```objc
uintptr_t
objc_object::sidetable_retainCount()
{
    SideTable& table = SideTables()[this];

    size_t refcnt_result = 1; // 设置对象本身的引用计数为1
    
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        // this is valid for SIDE_TABLE_RC_PINNED too
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT; // 引用计数 = 1 + SideTable 中存储的引用计数
    }
    table.unlock();
    return refcnt_result;
}
```
如果`isa`不是`nonpointer`，它直接存储着`Class`、`Meta-Class`对象的内存地址，没办法存储引用计数，所以引用计数都存储在`SideTable`中，这时候就通过`sidetable_retainCount()`获得引用计数。

>**小结：**`retainCount`方法：
>* 在`arm64`之前，`isa`不是`nonpointer`。对象的引用计数全都存储在`SideTable`中，`retainCount `方法返回的是对象本身的引用计数值 1，加上`SideTable`中存储的值；
>* 从`arm64`开始，`isa`是`nonpointer`。对象的引用计数先存储到它的`isa`中的`extra_rc`中，如果 19 位的`extra_rc`不够存储，那么溢出的部分再存储到`SideTable`中，`retainCount `方法返回的是对象本身的引用计数值 1，加上`isa`中的`extra_rc`存储的值，加上`SideTable`中存储的值。
>* 所以，其实我们通过`retainCount`方法打印`alloc`创建的对象的引用计数为 1，这是`retainCount`方法的功劳，`alloc`方法并没有设置对象的引用计数。

>**Why：** 那也不对啊，`alloc`方法没有设置对象的引用计数为 1，而且它内部也没有调用`retainCount`方法啊。那我们通过`alloc`创建出来的对象的引用计数岂不是就是 0，那不是会直接`dealloc`吗？<br><br>`dealloc`方法是在`release`方法内部调用的。只有你直接调用了`dealloc`，或者调用了`release`且在`release`方法中判断对象的引用计数为 0 的时候，才会调用`dealloc`。详情请参阅`release`源码分析。


### retain
在`《iOS - 老生常谈内存管理（二）：从 MRC 说起》`文章中已经讲解过，持有对象有两种方式，一是通过 `alloc`/`new`/`copy`/`mutableCopy`等方法创建对象，二是通过`retain`方法。`retain`方法会将对象的引用计数 +1。

`retain`方法的函数调用栈为：
```objc
// NSObject.mm
① objc_retain
// objc-object.h 
② objc_object::retain
// NSObject.mm
③ retain
④ _objc_rootRetain
// objc-object.h
⑤ objc_object::rootRetain
// NSObject.mm
⑥ objc_object::sidetable_retain
   addc // objc-os.h
   objc_object::rootRetain_overflow
   objc_object::sidetable_addExtraRC_nolock
```

#### ① objc_retain
```objc
#if __OBJC2__
__attribute__((aligned(16), flatten, noinline))
id 
objc_retain(id obj)
{
    if (!obj) return obj;
    if (obj->isTaggedPointer()) return obj;
    return obj->retain();
}
#else
id objc_retain(id obj) { return [obj retain]; }
#endif
```
如果是`__OBJC2__`，则调用`objc_object::retain`函数；否则调用`retain`方法。

#### ② objc_object::retain
```objc
// Equivalent to calling [this retain], with shortcuts if there is no override
inline id 
objc_object::retain()
{
    ASSERT(!isTaggedPointer());

    if (fastpath(!ISA()->hasCustomRR())) {
        return rootRetain();
    }

    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, @selector(retain));
}
```
如果方法没有被重写，直接调用`objc_object::rootRetain`，这是快捷方式；否则调用`retain`方法。

#### ③ retain
```objc
// Replaced by ObjectAlloc
- (id)retain {
    return _objc_rootRetain(self);
}
```
#### ④ _objc_rootRetainCount
```objc
NEVER_INLINE id
_objc_rootRetain(id obj)
{
    ASSERT(obj);

    return obj->rootRetain();
}
```

#### ⑤ objc_object::rootRetain
```objc
ALWAYS_INLINE id 
objc_object::rootRetain()
{
    return rootRetain(false, false);
}

ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    // 如果是 tagged pointer，直接返回 this
    if (isTaggedPointer()) return (id)this; 

    bool sideTableLocked = false;
    bool transcribeToSideTable = false; // 是否需要将引用计数存储在 sideTable 中

    isa_t oldisa;
    isa_t newisa;

    do {
        transcribeToSideTable = false;
        // 获取 isa
        oldisa = LoadExclusive(&isa.bits);  
        newisa = oldisa; 
        // 如果 isa 不是 nonpointer
        if (slowpath(!newisa.nonpointer)) { 
            ClearExclusive(&isa.bits);
            if (rawISA()->isMetaClass()) return (id)this;
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            // tryRetain == false，调用 sidetable_retain
            if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
            else return sidetable_retain(); 
        }
        // don't check newisa.fast_rr; we already called any RR overrides
        if (slowpath(tryRetain && newisa.deallocating)) {
            ClearExclusive(&isa.bits);
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            return nil;
        }
        uintptr_t carry; // 用于判断 isa 的 extra_rc 是否溢出，这里指上溢，即存满
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++

        // 如果 extra_rc 上溢
        if (slowpath(carry)) { 
            // newisa.extra_rc++ overflowed
            // 如果 handleOverflow == false，调用 rootRetain_overflow
            if (!handleOverflow) { 
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain); 
            }
            // Leave half of the retain counts inline and 
            // prepare to copy the other half to the side table.
            // 保留一半的引用计数在 extra_rc 中
            // 准备把另一半引用计数存储到 Sidetable 中
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;   // 设置 transcribeToSideTable 为 true
            newisa.extra_rc = RC_HALF;      // 设置 extra_rc 的值为 RC_HALF   # define RC_HALF  (1ULL<<18)
            newisa.has_sidetable_rc = true; // 设置 has_sidetable_rc 为 true
        }
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits))); // 保存更新后的 isa.bits

    // 如果需要将溢出的引用计数存储到 sidetable 中
    if (slowpath(transcribeToSideTable)) { 
        // Copy the other half of the retain counts to the side table.
        // 将 RC_HALF 个引用计数存储到 Sidetable 中
        sidetable_addExtraRC_nolock(RC_HALF); 
    }

    if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
    return (id)this;
}
```
#### ⑥ objc_object::sidetable_retain
我们先来看几个偏移量：
```objc
// The order of these bits is important.
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  // MSB-ward of weak bit
#define SIDE_TABLE_RC_ONE            (1UL<<2)  // MSB-ward of deallocating bit
#define SIDE_TABLE_RC_PINNED         (1UL<<(WORD_BITS-1))

#define SIDE_TABLE_RC_SHIFT 2
#define SIDE_TABLE_FLAG_MASK (SIDE_TABLE_RC_ONE-1)
```
* SIDE_TABLE_WEAKLY_REFERENCED：标记对象是否有弱引用
* SIDE_TABLE_DEALLOCATING：标记对象是否正在 dealloc
* SIDE_TABLE_RC_ONE：对象引用计数存储的开始位，引用计数存储在第 2～63 位
* SIDE_TABLE_RC_PINNED：引用计数的溢出标志位（最后一位）

以下是对象的引用计数表：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17c961c076d54eda82f20851da5f6c75~tplv-k3u1fbpfcp-zoom-1.image)


```objc
id
objc_object::sidetable_retain()
{
#if SUPPORT_NONPOINTER_ISA
    ASSERT(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];          // 获取 SideTable
    
    table.lock();
    size_t& refcntStorage = table.refcnts[this];    // 获取 refcnt
    if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) { // 如果获取到了，且未溢出
        refcntStorage += SIDE_TABLE_RC_ONE;         // 将引用计数加 1
    }
    table.unlock();

    return (id)this;
}
```
如果`isa`不是`nonpointer`，就会调用`sidetable_retain`，经过两次哈希查找得到对象的引用计数表，将引用计数 +1。


#### addc
```objc
static ALWAYS_INLINE uintptr_t 
addc(uintptr_t lhs, uintptr_t rhs, uintptr_t carryin, uintptr_t *carryout)
{
    return __builtin_addcl(lhs, rhs, carryin, carryout);
}
```
如果`isa`是`nonpointer `，就会调用`addc`将`extra_rc`中的引用计数 +1。这个函数的作用就是增加引用计数。


#### objc_object::rootRetain_overflow
```objc
NEVER_INLINE id 
objc_object::rootRetain_overflow(bool tryRetain)
{
    return rootRetain(tryRetain, true);
}
```
如果`extra_rc`中存储满了，就会调用`rootRetain_overflow`，该函数又调用了`rootRetain`，但参数`handleOverflow`传`true`。



#### objc_object::sidetable_addExtraRC_nolock
```objc
// Move some retain counts to the side table from the isa field.
// Returns true if the object is now pinned.
// 将一些引用计数从 isa 中转移到 sidetable
bool 
objc_object::sidetable_addExtraRC_nolock(size_t delta_rc)
{
    ASSERT(isa.nonpointer);
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;
    // isa-side bits should not be set here
    ASSERT((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    ASSERT((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    if (oldRefcnt & SIDE_TABLE_RC_PINNED) return true;

    uintptr_t carry;
    size_t newRefcnt = 
        addc(oldRefcnt, delta_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) {
        refcntStorage =
            SIDE_TABLE_RC_PINNED | (oldRefcnt & SIDE_TABLE_FLAG_MASK);
        return true;
    }
    else {
        refcntStorage = newRefcnt;
        return false;
    }
}
```
如果`extra_rc`中存储满了，就会调用`sidetable_addExtraRC_nolock`将`extra_rc`中的`RC_HALF`（`extra_rc`满值的一半）个引用计数转移到`sidetable`中存储，也是调用`addc`对`refcnt`引用计数表进行引用计数增加操作。


>**小结：**`retain`方法：
>* 如果`isa`不是`nonpointer`，那么就对`Sidetable`中的引用计数进行 +1；
>* 如果`isa`是`nonpointer`，就将`isa`中的`extra_rc`存储的引用计数进行 +1，如果溢出，就将`extra_rc`中`RC_HALF`（`extra_rc`满值的一半）个引用计数转移到`sidetable`中存储。
从`rootRetain`函数中我们可以看到，如果`extra_rc`溢出，设置它的值为`RC_HALF`，这时候又对`sidetable`中的`refcnt`增加引用计数`RC_HALF`。`extra_rc`是`19`位，而`RC_HALF`宏是`(1ULL<<18)`，实际上相等于进行了 +1 操作。



### release
当我们在不需要使用（持有）对象的时候，需要调用一下`release`方法进行释放。`release`方法会将对象的引用计数 -1。

`release`方法的函数调用栈为：
```
// NSObject.mm
① objc_release
// objc-object.h 
② objc_object::release
// NSObject.mm
③ release
④ _objc_rootRelease
// objc-object.h
⑤ objc_object::rootRelease
// NSObject.mm
⑥ objc_object::sidetable_release
   subc // objc-os.h
   objc_object::rootRelease_underflow
   objc_object::sidetable_subExtraRC_nolock
   objc_object::overrelease_error
```

#### ① objc_release
```objc
#if __OBJC2__
__attribute__((aligned(16), flatten, noinline))
void 
objc_release(id obj)
{
    if (!obj) return;
    if (obj->isTaggedPointer()) return;
    return obj->release();
}
#else
void objc_release(id obj) { [obj release]; }
#endif
```
如果是`__OBJC2__`，则调用`objc_object::release`函数；否则调用`release`方法。

#### ② objc_object::release
```objc
// Equivalent to calling [this release], with shortcuts if there is no override
inline void
objc_object::release()
{
    ASSERT(!isTaggedPointer());

    if (fastpath(!ISA()->hasCustomRR())) {
        rootRelease();
        return;
    }

    ((void(*)(objc_object *, SEL))objc_msgSend)(this, @selector(release));
}
```
如果方法没有被重写，直接调用`objc_object::rootRelease`，这是快捷方式；否则调用`release`方法。


#### ③ release

```objc
// Replaced by ObjectAlloc
- (oneway void)release {
    _objc_rootRelease(self);
}
```

#### ④ _objc_rootRelease
```objc
NEVER_INLINE void
_objc_rootRelease(id obj)
{
    ASSERT(obj);

    obj->rootRelease();
}
```
#### ⑤ objc_object::rootRelease
```objc
ALWAYS_INLINE bool 
objc_object::rootRelease()
{
    return rootRelease(true, false);
}

ALWAYS_INLINE bool 
objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    // 如果是 tagged pointer，直接返回 false
    if (isTaggedPointer()) return false; 

    bool sideTableLocked = false;

    isa_t oldisa;
    isa_t newisa;

 retry:
    do {
        // 获取 isa
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        // 如果 isa 不是 nonpointer
        if (slowpath(!newisa.nonpointer)) { 
            ClearExclusive(&isa.bits);
            if (rawISA()->isMetaClass()) return false;
            if (sideTableLocked) sidetable_unlock();
            // 调用 sidetable_release
            return sidetable_release(performDealloc); 
        }
        // don't check newisa.fast_rr; we already called any RR overrides
        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        // 如果发现溢出的情况，这里是下溢，指 extra_rc 中的引用计数已经为 0 了
        if (slowpath(carry)) { 
            // don't ClearExclusive()
            // 执行 underflow 处理下溢
            goto underflow; 
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, 
                                             oldisa.bits, newisa.bits))); // 保存更新后的 isa.bits

    if (slowpath(sideTableLocked)) sidetable_unlock();
    return false;

 underflow:
    // newisa.extra_rc-- underflowed: borrow from side table or deallocate
    // abandon newisa to undo the decrement
    // extra_rc-- 下溢，从 sidetable 借用或者 dealloc 对象
    newisa = oldisa;

    // 如果 isa 的 has_sidetable_rc 字段值为 1
    if (slowpath(newisa.has_sidetable_rc)) { 
        // 如果 handleUnderflow == false，调用 rootRelease_underflow
        if (!handleUnderflow) { 
            ClearExclusive(&isa.bits);
            return rootRelease_underflow(performDealloc); 
        }

        // Transfer retain count from side table to inline storage.
        // 将引用计数从 sidetable 中转到 extra_rc 中存储

        if (!sideTableLocked) {
            ClearExclusive(&isa.bits);
            sidetable_lock();
            sideTableLocked = true;
            // Need to start over to avoid a race against 
            // the nonpointer -> raw pointer transition.
            goto retry;
        }

        // Try to remove some retain counts from the side table.        
        // 尝试从 sidetable 中删除（借出）一些引用计数，传入 RC_HALF
        // borrowed 为 sidetable 实际删除（借出）的引用计数
        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF); 

        // To avoid races, has_sidetable_rc must remain set 
        // even if the side table count is now zero.
        // 为了避免竞争，has_sidetable_rc 必须保持设置
        // 即使 sidetable 中的引用计数现在是 0

        if (borrowed > 0) { // 如果 borrowed > 0
            // Side table retain count decreased.
            // Try to add them to the inline count.
            // 将它进行 -1，赋值给 extra_rc 
            newisa.extra_rc = borrowed - 1;  // redo the original decrement too
            // 存储更改后的 isa.bits
            bool stored = StoreReleaseExclusive(&isa.bits, 
                                                oldisa.bits, newisa.bits); 
            // 如果存储失败，立刻重试一次
            if (!stored) { 
                // Inline update failed. 
                // Try it again right now. This prevents livelock on LL/SC 
                // architectures where the side table access itself may have 
                // dropped the reservation.
                isa_t oldisa2 = LoadExclusive(&isa.bits);
                isa_t newisa2 = oldisa2;
                if (newisa2.nonpointer) {
                    uintptr_t overflow;
                    newisa2.bits = 
                        addc(newisa2.bits, RC_ONE * (borrowed-1), 0, &overflow);
                    if (!overflow) {
                        stored = StoreReleaseExclusive(&isa.bits, oldisa2.bits, 
                                                       newisa2.bits);
                    }
                }
            }
            // 如果还是存储失败，把引用计数再重新保存到 sidetable 中
            if (!stored) {
                // Inline update failed.
                // Put the retains back in the side table.
                sidetable_addExtraRC_nolock(borrowed);
                goto retry;
            }

            // Decrement successful after borrowing from side table.
            // This decrement cannot be the deallocating decrement - the side 
            // table lock and has_sidetable_rc bit ensure that if everyone 
            // else tried to -release while we worked, the last one would block.
            sidetable_unlock();
            return false;
        }
        else {
            // Side table is empty after all. Fall-through to the dealloc path.
        }
    }

    // 如果引用计数为 0，dealloc 对象
    // Really deallocate.
    // 如果当前 newisa 处于 deallocating 状态，保证对象只会 dealloc 一次
    if (slowpath(newisa.deallocating)) { 
        ClearExclusive(&isa.bits);
        if (sideTableLocked) sidetable_unlock();
        // 调用 overrelease_error
        return overrelease_error(); 
        // does not actually return
    }
    // 设置 newisa 为 deallocating 状态
    newisa.deallocating = true; 
    // 如果存储失败，继续重试
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry; 

    if (slowpath(sideTableLocked)) sidetable_unlock();

    __c11_atomic_thread_fence(__ATOMIC_ACQUIRE);

    // 如果 performDealloc == true，给对象发送一条 dealloc 消息
    if (performDealloc) { 
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, @selector(dealloc));
    }
    return true;
}
```

#### ⑥ objc_object::sidetable_release
```objc
// rdar://20206767
// return uintptr_t instead of bool so that the various raw-isa 
// -release paths all return zero in eax
uintptr_t
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    ASSERT(!isa.nonpointer);
#endif
    // 获取 SideTable
    SideTable& table = SideTables()[this]; 

    bool do_dealloc = false; // 标识是否需要执行 dealloc 方法

    table.lock();
    auto it = table.refcnts.try_emplace(this, SIDE_TABLE_DEALLOCATING);
    // 获取 refcnts
    auto &refcnt = it.first->second; 
    if (it.second) {
        do_dealloc = true;
    // 如果对象处于 deallocating 状态
    } else if (refcnt < SIDE_TABLE_DEALLOCATING) { 
        // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
        do_dealloc = true;
        refcnt |= SIDE_TABLE_DEALLOCATING;
    // 如果引用计数有值
    } else if (! (refcnt & SIDE_TABLE_RC_PINNED)) { 
        // 引用计数 -1
        refcnt -= SIDE_TABLE_RC_ONE; 
    }
    table.unlock();
    // 如果符合判断条件，dealloc 对象
    if (do_dealloc  &&  performDealloc) { 
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, @selector(dealloc));
    }
    return do_dealloc;
}
```
如果`isa`不是`nonpointer `，那么就对`Sidetable`中的引用计数进行 -1，如果引用计数 =0，就`dealloc`对象；

#### subc
```objc
static ALWAYS_INLINE uintptr_t 
subc(uintptr_t lhs, uintptr_t rhs, uintptr_t carryin, uintptr_t *carryout)
{
    return __builtin_subcl(lhs, rhs, carryin, carryout);
}
```
`subc`就是`addc`的反操作，用来减少引用计数。


#### objc_object::rootRelease_underflow
```objc
NEVER_INLINE bool 
objc_object::rootRelease_underflow(bool performDealloc)
{
    return rootRelease(performDealloc, true);
}
```
如果`extra_rc`下溢，就会调用`rootRelease_underflow`，该函数又调用了`rootRelease`，但参数`handleUnderflow `传`true`。


#### objc_object::sidetable_subExtraRC_nolock
```objc
// Move some retain counts from the side table to the isa field.
// Returns the actual count subtracted, which may be less than the request.
size_t 
objc_object::sidetable_subExtraRC_nolock(size_t delta_rc)
{
    ASSERT(isa.nonpointer);
    // 获取 SideTable
    SideTable& table = SideTables()[this];

    // 获取 refcnt
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()  ||  it->second == 0) {
        // Side table retain count is zero. Can't borrow.
        return 0;
    }
    size_t oldRefcnt = it->second;

    // isa-side bits should not be set here
    ASSERT((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    ASSERT((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    // 减少引用计数
    size_t newRefcnt = oldRefcnt - (delta_rc << SIDE_TABLE_RC_SHIFT);
    ASSERT(oldRefcnt > newRefcnt);  // shouldn't underflow
    it->second = newRefcnt;
    return delta_rc;
}
```
`sidetable_subExtraRC_nolock`目的就是请求将`sidetable`中存储的一些引用计数值转移到`isa`中。返回减去的实际引用计数，该值可能小于请求值。

#### objc_object::overrelease_error
```objc
NEVER_INLINE uintptr_t
objc_object::overrelease_error()
{
    _objc_inform_now_and_on_crash("%s object %p overreleased while already deallocating; break on objc_overrelease_during_dealloc_error to debug", object_getClassName((id)this), this);
    objc_overrelease_during_dealloc_error();
    return 0; // allow rootRelease() to tail-call this
}
```
如果当前对象处于`deallocating`状态，再次`release`就会执行`overrelease_error`，该函数就是用来在过度调用`release`的时候报错用的。

>**小结：**`release`方法：
>* 如果`isa`不是`nonpointer `，那么就对`Sidetable`中的引用计数进行 -1，如果引用计数 =0，就`dealloc`对象；
>* 如果`isa`是`nonpointer`，就将`isa`中的`extra_rc`存储的引用计数进行 -1。如果下溢，即`extra_rc`中的引用计数已经为 0，判断`has_sidetable_rc`是否为`true`即是否有使用`Sidetable`存储。如果有的话就申请从`Sidetable`中申请`RC_HALF`个引用计数转移到`extra_rc`中存储，如果不足`RC_HALF`就有多少申请多少，然后将`Sidetable`中的引用计数值减去`RC_HALF`（或是小于`RC_HALF`的实际值），将实际申请到的引用计数值 -1 后存储到`extra_rc`中。如果`extra_rc`中引用计数为 0 且`has_sidetable_rc`为`false`或者`Sidetable`中的引用计数也为 0 了，那就`dealloc`对象。<br><br>
> 为什么需要这么做呢？直接先从`Sidetable`中对引用计数进行 -1 操作不行吗？
> 我想应该是为了性能吧，毕竟访问对象的`isa`更快。

### autorelease
`autorelease`方法的函数调用栈为：
```objc
// NSObject.mm
① objc_autorelease
// objc-object.h 
② objc_object::autorelease
// NSObject.mm
③ autorelease
④ _objc_rootAutorelease
// objc-object.h
⑤ objc_object::rootAutorelease
// NSObject.mm
⑥ objc_object::rootAutorelease2
⑦ AutoreleasePoolPage::autorelease
```
#### ① objc_autorelease
```objc
#if __OBJC2__
__attribute__((aligned(16), flatten, noinline))
id
objc_autorelease(id obj)
{
    if (!obj) return obj;
    if (obj->isTaggedPointer()) return obj;
    return obj->autorelease();
}
#else
id objc_autorelease(id obj) { return [obj autorelease]; }
#endif
```
如果是`__OBJC2__`，则调用`objc_object::autorelease`函数；否则调用`autorelease`方法。

#### ② objc_object::autorelease
```objc
// Equivalent to [this autorelease], with shortcuts if there is no override
inline id 
objc_object::autorelease()
{
    ASSERT(!isTaggedPointer());
    if (fastpath(!ISA()->hasCustomRR())) {
        return rootAutorelease();
    }

    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, @selector(autorelease));
}
```
如果方法没有被重写，直接调用`objc_object::rootAutorelease`，这是快捷方式；否则调用`autorelease`方法。

#### ③ autorelease
```objc
// Replaced by ObjectAlloc
- (id)autorelease {
    return _objc_rootAutorelease(self);
}
```
#### ④ _objc_rootAutorelease
```objc
NEVER_INLINE id
_objc_rootAutorelease(id obj)
{
    ASSERT(obj);
    return obj->rootAutorelease();
}
```

#### ⑤ objc_object::rootAutorelease
```objc
// Base autorelease implementation, ignoring overrides.
inline id 
objc_object::rootAutorelease()
{
    if (isTaggedPointer()) return (id)this;
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;

    return rootAutorelease2();
}
```
#### ⑥ objc_object::rootAutorelease2
```objc
__attribute__((noinline,used))
id 
objc_object::rootAutorelease2()
{
    assert(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this);
}
```
在该函数中调用了`AutoreleasePoolPage`类的`autorelease`方法。
关于`AutoreleasePoolPage`类以及`autorelease`与`@autoreleasepool`，可参阅[《iOS - 聊聊 autorelease 和 @autoreleasepool》](https://juejin.im/post/6844904094503567368)。




### dealloc
`dealloc`方法的函数调用栈为：
```objc
// NSObject.mm
① dealloc
② _objc_rootDealloc
// objc-object.h
③ rootDealloc
// objc-runtime-new.mm
④ object_dispose
⑤ objc_destructInstance
// objc-object.h
⑥ clearDeallocating
// NSObject.mm
⑦ sidetable_clearDeallocating
   clearDeallocating_slow
```


#### ① dealloc
```objc
// Replaced by NSZombies
- (void)dealloc {
    _objc_rootDealloc(self);
}
```
#### ② _objc_rootDealloc
```objc
void
_objc_rootDealloc(id obj)
{
    ASSERT(obj);

    obj->rootDealloc();
}
```

#### ③ rootDealloc
```objc
inline void
objc_object::rootDealloc()
{
    // 判断是否为 TaggerPointer 内存管理方案，是的话直接 return
    if (isTaggedPointer()) return;  // fixme necessary? * 

    if (fastpath(isa.nonpointer  &&          // 如果 isa 为 nonpointer
                 !isa.weakly_referenced  &&  // 没有弱引用
                 !isa.has_assoc  &&          // 没有关联对象
                 !isa.has_cxx_dtor  &&       // 没有 C++ 的析构函数
                 !isa.has_sidetable_rc))     // 没有额外采用 SideTabel 进行引用计数存储
    {
        assert(!sidetable_present());
        free(this);               // 如果以上条件成立，直接调用 free 函数销毁对象
    } 
    else {
        object_dispose((id)this); // 如果以上条件不成立，调用 object_dispose 函数
    }
}
```

#### ④ object_dispose
```objc
/***********************************************************************
* object_dispose
* fixme
* Locking: none
**********************************************************************/
id 
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj); // 调用 objc_destructInstance 函数
    free(obj);                  // 调用 free 函数销毁对象

    return nil;
}
```
#### ⑤ objc_destructInstance
```objc
/***********************************************************************
* objc_destructInstance
* Destroys an instance without freeing memory. 
* Calls C++ destructors.
* Calls ARC ivar cleanup.
* Removes associative references.
* Returns `obj`. Does nothing if `obj` is nil.
**********************************************************************/
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);           // 如果有 C++ 的析构函数，调用 object_cxxDestruct 函数
        if (assoc) _object_remove_assocations(obj); // 如果有关联对象，调用 _object_remove_assocations 函数，移除关联对象
        obj->clearDeallocating();                   // 调用 clearDeallocating 函数
    }

    return obj;
}
```

#### ⑥ clearDeallocating
```objc
inline void 
objc_object::clearDeallocating()
{
    // 如果 isa 不是 nonpointer
    if (slowpath(!isa.nonpointer)) {     
        // Slow path for raw pointer isa.
        // 调用 sidetable_clearDeallocating 函数
        sidetable_clearDeallocating();   
    }
    // 如果 isa 是 nonpointer，且有弱引用或者有额外使用 SideTable 存储引用计数
    else if (slowpath(isa.weakly_referenced  ||  isa.has_sidetable_rc)) { 
        // Slow path for non-pointer isa with weak refs and/or side table data.
        // 调用 clearDeallocating_slow 函数
        clearDeallocating_slow();        
    }

    assert(!sidetable_present());
}
```
#### ⑦ sidetable_clearDeallocating
```objc
void 
objc_object::sidetable_clearDeallocating()
{
    // 获取 SideTable
    SideTable& table = SideTables()[this]; 

    // clear any weak table items
    // clear extra retain count and deallocating bit
    // (fixme warn or abort if extra retain count == 0 ?)
    table.lock();
    // 获取 refcnts
    RefcountMap::iterator it = table.refcnts.find(this); 
    if (it != table.refcnts.end()) {
        if (it->second & SIDE_TABLE_WEAKLY_REFERENCED) {
            // 调用 weak_clear_no_lock：将指向该对象的弱引用指针置为 nil
            weak_clear_no_lock(&table.weak_table, (id)this); 
        }
        // 调用 table.refcnts.erase：从引用计数表中擦除该对象的引用计数
        table.refcnts.erase(it); 
    }
    table.unlock();
}
```

#### clearDeallocating_slow
```objc
// Slow path of clearDeallocating() 
// for objects with nonpointer isa
// that were ever weakly referenced 
// or whose retain count ever overflowed to the side table.
NEVER_INLINE void
objc_object::clearDeallocating_slow()
{
    ASSERT(isa.nonpointer  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    // 获取 SideTable
    SideTable& table = SideTables()[this]; 
    table.lock();
    // 如果有弱引用
    if (isa.weakly_referenced) { 
        // 调用 weak_clear_no_lock：将指向该对象的弱引用指针置为 nil
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    // 如果有使用 SideTable 存储引用计数
    if (isa.has_sidetable_rc) {  
        // 调用 table.refcnts.erase：从引用计数表中擦除该对象的引用计数
        table.refcnts.erase(this);
    }
    table.unlock();
}
```

>**小结：**`dealloc`方法：
>* ① 判断 5 个条件（1.`isa`为`nonpointer`；2.没有弱引用；3.没有关联对象；4.没有`C++`的析构函数；5.没有额外采用`SideTabel`进行引用计数存储），如果这 5 个条件都成立，直接调用`free`函数销毁对象，否则调用`object_dispose`做一些释放对象前的处理；
>* ② 
>    * 1.如果有`C++`的析构函数，调用`object_cxxDestruct`；
>    * 2.如果有关联对象，调用`_object_remove_assocations`函数，移除关联对象；
>    * 3.调用`weak_clear_no_lock`将指向该对象的弱引用指针置为`nil`；
>    * 4.调用`table.refcnts.erase`从引用计数表中擦除该对象的引用计数（如果`isa`为`nonpointer`，还要先判断`isa.has_sidetable_rc`）
>* ③ 调用`free`函数销毁对象。
>
>根据`dealloc`过程，`__weak`修饰符的变量在对象被`dealloc`时，会将该`__weak`置为`nil`。可见，如果大量使用`__weak`变量的话，则会消耗相应的 CPU 资源，所以建议只在需要避免循环引用的时候使用`__weak`修饰符。
>
>在`《iOS - 老生常谈内存管理（三）：ARC 面世 —— 所有权修饰符》`章节中提到，`__weak`对性能会有一定的消耗，当一个对象`dealloc`时，需要遍历对象的`weak`表，把表里的所有`weak`指针变量值置为`nil`，指向对象的`weak`指针越多，性能消耗就越多。所以`__unsafe_unretained`比`__weak`快。当明确知道对象的生命周期时，选择`__unsafe_unretained`会有一些性能提升。

### weak

#### 清除 weak 
以上从`dealloc`方法实现我们知道了在对象`dealloc`的时候，会调用`weak_clear_no_lock`函数将指向该对象的弱引用指针置为`nil`，那么该函数的具体实现是怎样的呢？

##### weak_clear_no_lock

```objc
// objc-weak.mm
/** 
 * Called by dealloc; nils out all weak pointers that point to the 
 * provided object so that they can no longer be used.
 * 
 * @param weak_table 
 * @param referent The object being deallocated. 
 */
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    // 获得 weak 指向的地址，即对象内存地址
    objc_object *referent = (objc_object *)referent_id; 
    
    // 找到管理 referent 的 entry 容器
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent); 
    // 如果 entry == nil，表示没有弱引用需要置为 nil，直接返回
    if (entry == nil) { 
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) { 
        // referrers 是一个数组，存储所有指向 referent_id 的弱引用
        referrers = entry->referrers; 
        // 弱引用数组长度
        count = TABLE_SIZE(entry);    
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    // 遍历弱引用数组，将所有指向 referent_id 的弱引用全部置为 nil
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    
    // 从 weak_table 中移除对应的弱引用的管理容器
    weak_entry_remove(weak_table, entry);
}
```
>**小结：** 清除`weak`。
>
>当一个对象被销毁时，在`dealloc`方法内部经过一系列的函数调用栈，通过两次哈希查找，第一次根据对象的地址找到它所在的`Sidetable`，第二次根据对象的地址在`Sidetable`的`weak_table`中找到它的弱引用表。弱引用表中存储的是对象的地址（作为`key`）和`weak`指针地址的数组（作为`value`）的映射。`weak_clear_no_lock`函数中遍历弱引用数组，将指向对象的地址的`weak`变量全都置为`nil`。



#### 添加 weak 

接下来我们来看一下`weak`变量是怎样添加到弱引用表中的。

一个被声明为`__weak`的指针，在经过编译之后。通过`objc_initWeak`函数初始化附有`__weak`修饰符的变量，在变量作用域结束时通过`objc_destroyWeak`函数销毁该变量。

```objc
{
    id obj = [[NSObject alloc] init];
    id __weak obj1 = obj;
}
    /*----- 编译 -----*/
    id obj1;
    objc_initWeak(&obj1,obj);
    objc_destroyWeak(&obj1);
```


`objc_initWeak`函数调用栈如下：
```objc
// NSObject.mm
① objc_initWeak
② storeWeak
// objc-weak.mm
③ weak_register_no_lock
   weak_unregister_no_lock
```
##### ① objc_initWeak
```objc
/** 
 * Initialize a fresh weak pointer to some object location. 
 * It would be used for code like: 
 *
 * (The nil case) 
 * __weak id weakPtr;
 * (The non-nil case) 
 * NSObject *o = ...;
 * __weak id weakPtr = o;
 * 
 * This function IS NOT thread-safe with respect to concurrent 
 * modifications to the weak variable. (Concurrent weak clear is safe.)
 *
 * @param location Address of __weak ptr. 
 * @param newObj Object ptr. 
 */
id
objc_initWeak(id *location, id newObj) // *location 为 __weak 指针地址，newObj 为对象地址
{
    // 如果对象为 nil，那就将 weak 指针置为 nil
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
```

##### ② storeWeak
```objc
// Update a weak variable.
// If HaveOld is true, the variable has an existing value 
//   that needs to be cleaned up. This value might be nil.
// If HaveNew is true, there is a new value that needs to be 
//   assigned into the variable. This value might be nil.
// If CrashIfDeallocating is true, the process is halted if newObj is 
//   deallocating or newObj's class does not support weak references. 
// If CrashIfDeallocating is false, nil is stored instead.
// 更新 weak 变量
// 如果 HaveOld == true，表示变量有旧值，它需要被清理，这个旧值可能为 nil
// 如果 HaveNew == true，表示一个新值需要赋值给变量，这个新值可能为 nil
// 如果 CrashIfDeallocating == true，则如果对象正在销毁或者对象不支持弱引用，则停止更新
// 如果 CrashIfDeallocating == false，则存储 nil
enum CrashIfDeallocating {
    DontCrashIfDeallocating = false, DoCrashIfDeallocating = true
};
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;  // 旧表，用来存放已有的 weak 变量
    SideTable *newTable;  // 新表，用来存放新的 weak 变量

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    // 分别获取新旧值相关联的弱引用表
    // 如果 weak 变量有旧值，获取已有对象（该旧值对象）和旧表
    if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    // 如果有新值要赋值给变量，创建新表
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
    
    // 对 haveOld 和 haveNew 分别加锁
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    // 判断 oldObj 和 location 指向的值是否相等，即是否是同一对象，如果不是就重新获取旧值相关联的表
    if (haveOld  &&  *location != oldObj) {
        // 解锁
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    // 如果有新值，判断新值所属的类是否已经初始化
    // 如果没有初始化，则先执行初始化，防止 +initialize 内部调用 storeWeak 产生死锁
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            class_initialize(cls, (id)newObj);

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // 如果有旧值，调用 weak_unregister_no_lock 清除旧值
    // Clean up old value, if any.
    if (haveOld) {
        // 移除所有指向旧值的 weak 引用，而不是赋值为 nil
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // 如果有新值要赋值，调用 weak_register_no_lock 将所有 weak 指针重新指向新的对象
    // Assign new value, if any.
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // 如果存储成功
        // 如果对象是 Tagged Pointer，不做操作
        // 如果 isa 不是 nonpointer，设置 SideTable 中弱引用标志位
        // 如果 isa 是 nonpointer，设置 isa 的 weakly_referenced 弱引用标志位
        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }
 
        // 将 location 指向新的对象
        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    // 解锁
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```
`store_weak`函数的执行过程如下：
* 分别获取新旧值相关联的弱引用表；
* 如果有旧值，就调用`weak_unregister_no_lock`函数清除旧值，移除所有指向旧值的`weak`引用，而不是赋值为`nil`；
* 如果有新值，就调用`weak_register_no_lock`函数分配新值，将所有`weak`指针重新指向新的对象；
* 判断`isa`是否为`nonpointer`来设置弱引用标志位。如果不是`nonpointer`，设置`SideTable`中的弱引用标志位，否则设置`isa`的`weakly_referenced`弱引用标志位。

##### ③ weak_register_no_lock
```objc
/** 
 * Registers a new (object, weak pointer) pair. Creates a new weak
 * object entry if it does not exist.
 * 
 * @param weak_table The global weak table.
 * @param referent The object pointed to by the weak reference.
 * @param referrer The weak pointer address.
 */
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
```
`weak_register_no_lock`用来保存弱引用信息，具体实现如下：

* 判断对象是否正在释放，是否支持弱引用`allowsWeakReference`，如果实例对象的`allowsWeakReference`方法返回`NO`，则调用`_objc_fatal`并在控制台打印`"Cannot form weak reference to instance (%p) of class %s. It is possible that this object was over-released, or is in the process of deallocation."`；<br>（关于`allowsWeakReference`已经在[《iOS - 老生常谈内存管理（三）：ARC 面世》](https://juejin.im/post/6844904130431942670)中讲到）
* 查询`weak_table`，判断弱引用表中是否已经保存有与对象相关联的弱引用信息；
* 如果已经有相关弱引用信息，则调用`append_referrer`函数将弱引用信息添加进现在`entry `容器中；如果没有相关联信息，则创建一个`entry `，并且插入到`weak_table`弱引用表中。


##### weak_unregister_no_lock
```objc
/** 
 * Unregister an already-registered weak reference.
 * This is used when referrer's storage is about to go away, but referent
 * isn't dead yet. (Otherwise, zeroing referrer later would be a
 * bad memory access.)
 * Does nothing if referent/referrer is not a currently active weak reference.
 * Does not zero referrer.
 * 
 * FIXME currently requires old referent value to be passed in (lame)
 * FIXME unregistration should be automatic if referrer is collected
 * 
 * @param weak_table The global weak table.
 * @param referent The object.
 * @param referrer The weak reference.
 */
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;

    if (!referent) return;

    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        remove_referrer(entry, referrer);
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }

        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }

    // Do not set *referrer = nil. objc_storeWeak() requires that the 
    // value not change.
}
```
`weak_unregister_no_lock`用来移除弱引用信息，具体实现如下：

* 查询`weak_table`，判断弱引用表中是否已经保存有与对象相关联的弱引用信息；
* 如果有，则调用`remove_referrer`方法移除相关联的弱引用信息；接着判断存储数组是否为空，如果为空，则调用`weak_entry_remove`移除`entry`容器。



`objc_destroyWeak`函数调用栈如下：
```objc
// NSObject.mm
① objc_destroyWeak
② storeWeak
```

##### objc_destroyWeak
```objc
/** 
 * Destroys the relationship between a weak pointer
 * and the object it is referencing in the internal weak
 * table. If the weak pointer is not referencing anything, 
 * there is no need to edit the weak table. 
 *
 * This function IS NOT thread-safe with respect to concurrent 
 * modifications to the weak variable. (Concurrent weak clear is safe.)
 * 
 * @param location The weak pointer address. 
 */
void
objc_destroyWeak(id *location)
{
    (void)storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating>
        (location, nil);
}
```
`objc_initWeak`和`objc_destroyWeak`函数中都调用了`storeWeak`，但是传的参数不同。
* `objc_initWeak`将对象地址传入，且`DontHaveOld`、`DoHaveNew`、`DoCrashIfDeallocating`；
* `objc_destroyWeak`将`nil`传入，且`DoHaveOld`、`DontHaveNew`、`DontCrashIfDeallocating`。

`storeWeak`函数把参数二的赋值的对象地址作为`key`，把参数一的附有`__weak`修饰符的变量的地址注册到`weak`表中。如果参数二为`nil`，则把变量的地址从`weak`表中删除。


>**小结：** 添加`weak`。
>
>一个被标记为`__weak`的指针，在经过编译之后会调用`objc_initWeak`函数，`objc_initWeak`函数中初始化`weak`变量后调用`storeWeak`。添加`weak`的过程如下：
>
>经过一系列的函数调用栈，最终在`weak_register_no_lock()`函数当中，进行弱引用变量的添加，具体添加的位置是通过哈希算法来查找的。如果对应位置已经存在当前对象的弱引用表（数组），那就把弱引用变量添加进去；如果不存在的话，就创建一个弱引用表，然后将弱引用变量添加进去。

### 总结

以上就是内存管理方法的具体实现，接下来做个小总结：

| 内存管理方法                 | 具体实现        |
|:------------------------------------- | :------------------------ |
|alloc | 经过一系列的函数调用栈，最终通过调用 C 函数`calloc`来申请内存空间，并初始化对象的`isa`，但并没有设置对象的引用计数值为 1。                                                                                                                                          |
| init                                  | 基类的`init`方法啥都没干，只是将`alloc`创建的对象返回。我们可以重写`init`方法来对`alloc`创建的实例做一些初始化操作。                                                                                |
| new                                   | `new`方法很简单，只是嵌套了`alloc`和`init`。                                                                                                           |
| copy、mutableCopy                    | 调用了`copyWithZone`和`mutableCopyWithZone`方法。                                                                                                            |
| retainCount                           | ① 如果`isa`不是`nonpointer`，引用计数值 = `SideTable`中的引用计数表中存储的值 + 1；<br>② 如果`isa`是`nonpointer`，引用计数值 = `isa`中的`extra_rc`存储的值 + 1 +`SideTable`中的引用计数表中存储的值。                                                                           |
| retain                                | ① 如果`isa`不是`nonpointer`，就对`Sidetable`中的引用计数进行 +1；<br>② 如果`isa`是`nonpointer`，就将`isa`中的`extra_rc`存储的引用计数进行 +1，如果溢出，就将`extra_rc`中`RC_HALF`（`extra_rc`满值的一半）个引用计数转移到`sidetable`中存储。                                                                                                                                                                                     |
| release                               | ① 如果`isa`不是`nonpointer `，就对`Sidetable`中的引用计数进行 -1，如果引用计数 =0，就`dealloc`对象；<br>② 如果`isa`是`nonpointer`，就将`isa`中的`extra_rc`存储的引用计数进行 -1。如果下溢，即`extra_rc`中的引用计数已经为 0，判断`has_sidetable_rc`是否为`true`即是否有使用`Sidetable`存储。如果有的话就申请从`Sidetable`中申请`RC_HALF`个引用计数转移到`extra_rc`中存储，如果不足`RC_HALF`就有多少申请多少，然后将`Sidetable`中的引用计数值减去`RC_HALF`（或是小于`RC_HALF`的实际值），将实际申请到的引用计数值 -1 后存储到`extra_rc`中。如果`extra_rc`中引用计数为 0 且`has_sidetable_rc`为`false`或者`Sidetable`中的引用计数也为 0 了，那就`dealloc`对象。 |
| dealloc                               | ① 判断销毁对象前有没有需要处理的东西（如弱引用、关联对象、`C++`的析构函数、`SideTabel`的引用计数表等等）；<br>② 如果没有就直接调用`free`函数销毁对象；<br>③ 如果有就先调用`object_dispose`做一些释放对象前的处理（置弱引用指针置为`nil`、移除关联对象、`object_cxxDestruct`、在`SideTabel`的引用计数表中擦出引用计数等等），再用`free`函数销毁对象。 |
| 清除`weak`，`weak`指针置为`nil`的过程 | 当一个对象被销毁时，在`dealloc`方法内部经过一系列的函数调用栈，通过两次哈希查找，第一次根据对象的地址找到它所在的`Sidetable`，第二次根据对象的地址在`Sidetable`的`weak_table`中找到它的弱引用表。遍历弱引用数组，将指向对象的地址的`weak`变量全都置为`nil`。 |
| 添加`weak`                          | 经过一系列的函数调用栈，最终在`weak_register_no_lock()`函数当中，进行弱引用变量的添加，具体添加的位置是通过哈希算法来查找的。如果对应位置已经存在当前对象的弱引用表（数组），那就把弱引用变量添加进去；如果不存在的话，就创建一个弱引用表，然后将弱引用变量添加进去。 |

建议大家自己通过`objc4`源码看一遍，这样印象会更深一些。另外本篇文章的源码分析并没有分析得很细节，如果大家感兴趣可以自己研究一遍，刨根问底固然是好。如果以后有时间，我会再具体分析并更新本文章。