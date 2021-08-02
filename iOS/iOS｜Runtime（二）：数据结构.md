## Runtime 系列文章
[深入浅出 Runtime（一）：初识](https://juejin.im/post/6844904071480999949)<br>
[深入浅出 Runtime（二）：数据结构](https://juejin.im/post/6844904072215003143)<br>
[深入浅出 Runtime（三）：消息机制](https://juejin.im/post/6844904072235974663)<br>
[深入浅出 Runtime（四）：super 的本质](https://juejin.im/post/6844904072252751880)<br>
[深入浅出 Runtime（五）：相关面试题](https://juejin.im/post/6844904072428912653)

![](https://user-gold-cdn.xitu.io/2020/2/28/17087c4d8e1b438a?w=1600&h=900&f=jpeg&s=357120)


## 1. objc_object
`Objective-C`的面向对象都是基于`C/C++`的数据结构——结构体实现的。<br>
我们平时使用的所有对象都是`id`类型，`id`类型对象对应到`runtime`中，就是`objc_object`结构体。
```objc
// A pointer to an instance of a class.
typedef struct objc_object *id;
```
```objc
struct objc_object {
private:
    isa_t isa;
    /*...
      isa操作相关
      弱引用相关
      关联对象相关
      内存管理相关
      ...
     */
};
```

## 2. objc_class
`Class`指针用来指向一个 Objective-C 的类，它是`objc_class`结构体类型，所以`class、meta-class`底层结构都是`objc_class`结构体，`objc_class`继承自`objc_object`，所以它也有`isa`指针，它也是对象。
```objc
// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
```
```objc
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;          // 指向父类
    cache_t cache;             // 方法缓存 formerly cache pointer and vtable
    class_data_bits_t bits;    // 用于获取具体的类信息 class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() { 
        return bits.data();
    }
};
```
### 2.1 class_data_bits_t
* `class_data_bits_t`主要是对`class_rw_t`的封装，可以通过`bits & FAST_DATA_MASK`获得`class_rw_t`。
```objc
struct class_data_bits_t {
    // Values are the FAST_ flags above.
    uintptr_t bits;
public:
    class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }
};
```
* `class_rw_t`代表了类相关的读写信息，它是对`class_ro_t`的封装；
* `class_rw_t`中主要存储着类的方法列表、属性列表、协议列表等； 
* `class_rw_t`里面的`methods`、`properties`、`protocols`都继承于`list_array_tt`二维数组，是可读可写的，包含了类的初始内容、分类的内容。
```objc
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;       // 方法列表
    property_array_t properties;  // 属性列表
    protocol_array_t protocols;   // 协议列表

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
};
```
* `class_ro_t`代表了类相关的只读信息；
* `class_ro_t`中主要存储着类的成员变量列表、类名等；
* `class_ro_t`里面的`baseMethodList`、`baseProtocols`、`ivars`、`baseProperties`是一维数组，是只读的，包含了类的初始内容；
* 一开始类的信息都存放在`class_ro_t`里，当程序运行时，经过一系列的函数调用栈，在`realizeClass()`函数中，将`class_ro_t`里的东西和分类的东西合并起来放到`class_rw_t`里，并让`bits`指向`class_rw_t`。

```objc
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;  // instance对象占用的内存空间
#ifdef __LP64__
    uint32_t reserved;
#endif
    const uint8_t * ivarLayout;    
    const char * name;  // 类名
    method_list_t * baseMethodList;  
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;  // 成员变量列表
    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```

* `method_array_t`与`method_list_t`。

![method_array_t 与 method_list_t](https://user-gold-cdn.xitu.io/2020/2/25/1707be7d42e8cd5f?w=1240&h=693&f=png&s=109578)

### 2.2 cache_t
* 用于快速查找方法执行函数；
* 是可增量扩展的哈希表结构，用哈希表来缓存曾经使用过的方法，可以提高方法的查找速度（空间换时间：牺牲内存空间来换取执行效率）；
* 是局部性原理的最佳应用（比如一些方法调用的频率高，存放到`cache`中，下一次调用这些方法的命中率就会更高些）；
* hash 函数式为 `f(@selector()) = index, @selector() & _mask`；
* 当我们调用过一个方法后，`runtime`会将这个方法缓存到`cache`中，下次再调用此方法的时候，`runtime`会优先去`cache`中查找。
```objc
struct cache_t {
    struct bucket_t *_buckets;  // 哈希表
    mask_t _mask;               // 哈希表的长度 - 1
    mask_t _occupied;           // 已经缓存的方法数量
};

struct bucket_t {
private:
    cache_key_t _key;  // SEL
    IMP _imp;          // IMP 函数的内存地址
};
```
#### 2.2.1 缓存查找流程
```objc
//objc-cache.mm（objc4）
bucket_t * cache_t::find(cache_key_t k, id receiver)  // 根据 k 即 @selector 进行查找
{
    assert(k != 0);

    bucket_t *b = buckets();          // 获取_buckets
    mask_t m = mask();                // 获取_mask
    mask_t begin = cache_hash(k, m);  // 计算起始索引
    mask_t i = begin;
    do {
        // 根据索引 i 从 _buckets 哈希表中取值
        // 如果取出来的 bucket_t 的 _key = 0，说明在索引的位置上还没有缓存过方法，返回该 bucket_t，中止缓存查询，用于 cache_fill_nolock() 函数
        // 如果取出来的 bucket_t 的 _key = k，说明查询成功，返回该 bucket_t
        if (b[i].key() == 0  ||  b[i].key() == k) {
            return &b[i];
        }
      // 在 __arm64__ 下将索引 i -1，继续查找，反向遍历 _buckets 哈希表
      // 直到 i 指向首个元素即索引 = 0 时，将 mask 赋值给 i，使其指向哈希表最后一个元素，继续反向遍历
      // 如果此时还没有找到 k 对应的 bucket_t ，或者是空的 bucket_t ，则循环结束，查找失败，调用 bad_cache() 函数
      // 接下来去类对象中 class_rw_t 中的 methods 查找
    } while ((i = cache_next(i, m)) != begin);

    // hack
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)k, cls);
}


static inline mask_t cache_hash(cache_key_t key, mask_t mask) 
{
    return (mask_t)(key & mask);
}
static inline mask_t cache_next(mask_t i, mask_t mask) {
    // return (i+1) & mask;  // __arm__  ||  __x86_64__  ||  __i386__
    return i ? i-1 : mask;   // __arm64__
}
```
#### 2.2.2 缓存添加流程
```objc
//objc-cache.mm（objc4）
static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver)
{
    cacheUpdateLock.assertLocked();

    // Never cache before +initialize is done
    if (!cls->isInitialized()) return;   // 如果类还未初始化，直接返回

    // Make sure the entry wasn't added to the cache by some other thread 
    // before we grabbed the cacheUpdateLock.
    if (cache_getImp(cls, sel)) return;  // 可能有其它线程抢先将该方法缓存了，所以要检查一次缓存，如果存在，直接返回

    cache_t *cache = getCache(cls);  // ️取出该 class 的 cache_t
    cache_key_t key = getKey(sel);   // ️根据 sel 获得 _key

    // Use the cache as-is if it is less than 3/4 full
    mask_t newOccupied = cache->occupied() + 1;  // 将 cache_t 的 _occupied 即已经缓存的方法数量 + 1，这里只是为了判断 +1 后缓存容量是否满
    mask_t capacity = cache->capacity();  // 获得缓存容量 = _mask + 1
    if (cache->isConstantEmptyCache()) {  // 如果缓存是只读的，重新申请缓存空间
        // Cache is read-only. Replace it.
        cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE);  // 申请新的缓存空间，并释放旧的
    }
    else if (newOccupied <= capacity / 4 * 3) {  // ️如果当前已经缓存的方法数量 +1 <= 缓存容量的 3/4，就继续往下操作
        // Cache is less than 3/4 full. Use it as-is.
    }
    else {  // ️如果以上条件不满足，说明缓存已满，进行缓存扩容
        // Cache is too full. Expand it.
        cache->expand();
    }

    // Scan for the first unused slot and insert there.     // 扫描第一个未使用的插槽（bucket_t）并将其插入
    // There is guaranteed to be an empty slot because the  // 必然会有一个空的插槽（bucket_t）
    // minimum size is 4 and we resized at 3/4 full.        // 因为最小大小是4，我们调整为3/4满
    bucket_t *bucket = cache->find(key, receiver);       // ️调用 find() 函数进行一次缓存查找，必然会得到一个空的 bucket_t
    if (bucket->key() == 0) cache->incrementOccupied();  // ️如果该 bucket_t 为空，将 _occupied 即已经缓存的方法数量 + 1
    bucket->set(key, imp);  // ️添加缓存
}

void cache_fill(Class cls, SEL sel, IMP imp, id receiver)
{
#if !DEBUG_TASK_THREADS
    mutex_locker_t lock(cacheUpdateLock);
    cache_fill_nolock(cls, sel, imp, receiver);
#else
    _collecting_in_critical();
    return;
#endif
}
```
#### 2.2.3 缓存扩容流程
* ① 设置新的缓存`bucket_t`，容量 = 旧的两倍；
* ② 设置新的`_mask`=`bucket_t`长度 - 1；
* ③ 释放旧的缓存（在`runtime`动态交换方法实现时也会释放缓存）。
```objc
//objc-cache.mm（objc4）
void cache_t::expand()
{
    cacheUpdateLock.assertLocked();
    
    uint32_t oldCapacity = capacity();
    // ️将缓存扩容为原来的两倍，如果是首次调用，设置缓存容量的初始值为 4
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;

    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        // mask overflow - can't grow further
        // fixme this wastes one bit of mask
        newCapacity = oldCapacity;
    }

    reallocate(oldCapacity, newCapacity);  // ️申请新的缓存空间，并释放旧的
}

enum {
    INIT_CACHE_SIZE_LOG2 = 2,
    INIT_CACHE_SIZE      = (1 << INIT_CACHE_SIZE_LOG2)
};

void cache_t::reallocate(mask_t oldCapacity, mask_t newCapacity)
{
    bool freeOld = canBeFreed();  // ️判断一下缓存是不是空的，如果为空，就没必要释放空间

    bucket_t *oldBuckets = buckets();
    bucket_t *newBuckets = allocateBuckets(newCapacity);

    // Cache's old contents are not propagated. 
    // This is thought to save cache memory at the cost of extra cache fills.
    // fixme re-measure this

    assert(newCapacity > 0);
    assert((uintptr_t)(mask_t)(newCapacity-1) == newCapacity-1);

    setBucketsAndMask(newBuckets, newCapacity - 1);
    
    if (freeOld) {
        cache_collect_free(oldBuckets, oldCapacity);
        cache_collect(false);
    }
}

bool cache_t::canBeFreed()
{
    return !isConstantEmptyCache();
}

bool cache_t::isConstantEmptyCache()
{
    return 
        occupied() == 0  &&  
        buckets() == emptyBucketsForCapacity(capacity(), false);
}
```
* 更多关于`cache_t`的内容，请查看：<br>
[深入浅出 Runtime（三）：消息机制](https://juejin.im/post/6844904072235974663)



## 3. isa 指针
* `isa`指针用来维护对象和类之间的关系，并确保对象和类能够通过`isa`指针找到对应的方法、实例变量、属性、协议等；
* 在 arm64 架构之前，`isa`就是一个普通的指针，直接指向`objc_class`，存储着`Class`、`Meta-Class`对象的内存地址。`instance`对象的`isa`指向`class`对象，`class`对象的`isa`指向`meta-class`对象；
* 从 arm64 架构开始，对`isa`进行了优化，变成了一个共用体（`union`）结构，还使用位域来存储更多的信息。将 64 位的内存数据分开来存储着很多的东西，其中的 33 位才是拿来存储`class`、`meta-class`对象的内存地址信息。要通过位运算将`isa`的值`& ISA_MASK`掩码，才能得到`class`、`meta-class`对象的内存地址。
```objc
struct objc_object {
    Class isa;  // 在 arm64 架构之前
};

struct objc_object {
private:
    isa_t isa;  // 在 arm64 架构开始
};

union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

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
#   define ISA_MASK        0x0000000ffffffff8ULL  // 用来取出 Class、Meta-Class 对象的内存地址
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1;  // 0：代表普通的指针，存储着 Class、Meta-Class 对象的内存地址
                                          // 1：代表优化过，使用位域存储更多的信息
        uintptr_t has_assoc         : 1;  // 是否有设置过关联对象，如果没有，释放时会更快
        uintptr_t has_cxx_dtor      : 1;  // 是否有C++的析构函数（.cxx_destruct），如果没有，释放时会更快
        uintptr_t shiftcls          : 33; // 存储着 Class、Meta-Class 对象的内存地址信息
        uintptr_t magic             : 6;  // 用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1;  // 是否有被弱引用指向过，如果没有，释放时会更快
        uintptr_t deallocating      : 1;  // 对象是否正在释放
        uintptr_t has_sidetable_rc  : 1;  // 如果为1，代表引用计数过大无法存储在 isa 中，那么超出的引用计数会存储在一个叫 SideTable 结构体的 RefCountMap（引用计数表）哈希表中
        uintptr_t extra_rc          : 19; // 里面存储的值是引用计数 retainCount - 1
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };
};
```
### 3.1 isa 与 superclass 指针指向
![isa 与 superclass 指针指向](https://user-gold-cdn.xitu.io/2020/2/25/1707be7d4709e537?w=1041&h=664&f=png&s=282120)

### 3.2 类对象（class）与元类对象（meta-class）
* `class`、`meta-class`底层结构都是`objc_class`结构体，`objc_class`继承自`objc_object`，所以它也有`isa`指针，所以它也是对象；
* `class`中存储着实例方法、成员变量、属性、协议等信息，
`meta-class`中存储着类方法等信息；
* `isa`指针和`superclass`指针的指向（如上图）；
* 基类的`meta-class`的`superclass`指向基类的`class`，
决定了一个性质：当我们调用一个类方法，会通过`class`的`isa`指针找到`meta-class`，在`meta-class`中查找有无该类方法，如果没有，再通过`meta-class`的`superclass`指针逐级查找父`meta-class`，一直找到基类的`meta-class`如果还没找到该类方法的话，就会去找基类的`class`中同名的实例方法的实现。

### 3.3 获得 class 或者 meta-class 的方式
* 获得 class 有 3 种方式
```objc
- (Class)class;
+ (Class)class;
Class object_getClass(id obj);  // 传参：instance 对象
```
* 获得 meta-class 只有 1 种方式
```objc
Class object_getClass(id obj);  // 传参：Class 对象
```
示例代码如下
```objc
    NSObject *object1 = [NSObject alloc] init];
    NSObject *object2 = [NSObject alloc] init];
    // objectClass1 ~ objectClass5 都是 NSObject 的类对象
    Class objectClass1 = [object1 class];
    Class objectClass2 = [object2 class];
    Class objectClass3 = [NSObject class];
    Class objectClass4 = object_getClass(object1);
    Class objectClass5 = object_getClass(object2);  
    // objectMetaClass1 ~ objectMetaClass4 都是 NSObject 的元类对象
    Class objectMetaClass1 = object_getClass([object1 class];    
    Class objectMetaClass2 = object_getClass([NSObject class]);    
    Class objectMetaClass3 = object_getClass(object_getClass(object1));    
    Class objectMetaClass4 = object_getClass(objectClass5);    
```
方法实现
```
- (Class)class {
    return object_getClass(self);
}

+ (Class)class {
    return self;
}

Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
objc_object::getIsa() 
{
    if (!isTaggedPointer()) return ISA();
    ......
}
objc_object::ISA() 
{
    assert(!isTaggedPointer()); 
#if SUPPORT_INDEXED_ISA
    if (isa.nonpointer) {
        uintptr_t slot = isa.indexcls;
        return classForIndex((unsigned)slot);
    }
    return (Class)isa.bits;
#else
    return (Class)(isa.bits & ISA_MASK);
#endif
}
#if __ARM_ARCH_7K__ >= 2
#   define SUPPORT_INDEXED_ISA 1
#else
#   define SUPPORT_INDEXED_ISA 0
#endif
```

### 3.4 为什么要设计 meta-class ？
目的是将实例和类的相关方法列表以及构建信息区分开来，方便各司其职，符合单一职责设计原则。


## 4. method_t
* `Method`就是`method_t`类型的指针；
* `method_t`是对方法/函数的封装（函数四要素：函数名、返回值、参数、函数体）。
```
typedef struct method_t *Method;
```
```objc
struct method_t {
    SEL name;  // 方法名
    const char *types;  // 编码（返回值类型、参数类型）
    IMP imp;   // 方法的地址/实现
};
```
### 4.1 SEL
* SEL 又称“选择器”，它是一个指向方法的`selector`的指针，代表方法/函数名；
* SEL 维护在一个全局的 Map 中，所以它是全局唯一的，不同类中相同名字的方法的 SEL 是相同的。
```objc
typedef struct objc_selector *SEL;
```
* SEL 可以通过以下方式获得
```objc
    SEL sel1 = @selector(selector);
    SEL sel2 = sel_registerName("selector");
    SEL sel3 = NSSelectorFromString(@"selector");
```
* SEL 可以通过以下方式转换成字符串
```objc
    char *string1 = sel_getName(sel1);
    NSString *string2 = NSStringFromSelector(sel1);
```
### 4.2 IMP
* IMP 是指向方法实现的函数指针；
* 我们调用方法，实际上就是根据方法 SEL 查找 IMP；
* `method_t`实际上相当于在 SEL 和 IMP 之间做了一个映射。
```objc
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
#endif
```
### 4.3 Type Encodings
* Type Encodings 编码技术就是配合`runtime`的技术，把一个方法的返回值类型、参数类型通过字符串的形式描述；
* `@encode()`指令可以将类型转换为 Type Encodings 字符串编码，
如`@encode(int)`=`i`；
* `OC`方法都有两个隐式参数，方法调用者`(id)self`和方法名`(SEL) _cmd`，所以我们才能在方法中使用`self`和`_cmd`；
* 如 `-(void)test`，它的编码为“`v16@0:8`”，可以简写为“`v@:`”
<br>`v`：代表返回值类型为 void
<br>`@`：代表参数 1 类型为 id
<br>`:`：代表参数 2 类型为 SEL
<br>`16`：代表所有参数所占的总字节数
<br>`0`：代表参数 1 从第几个字节开始存储
<br>`8`：代表参数 2 从第几个字节开始存储
* 下图为类型对应的 Type Encodings 编码：

![Objective-C type encodings](https://user-gold-cdn.xitu.io/2020/2/25/1707be7d470bce9b?w=1240&h=1614&f=jpeg&s=137498)
* Type Encodings 在`runtime`的消息转发中会使用到；
* 更多关于 Type Encodings 的内容，可以查看官方文档 [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)。

&nbsp;

