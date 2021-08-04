## 前言

作为 iOS 开发者，在面试过程中经常会碰到这样一个问题：在 ARC 环境下`autorelease`对象在什么时候释放？如果你还不知道怎么回答，或者你只有比较模糊的概念，那么你绝对不能错过本文。

本文将通过`Runtime objc4-756.2`版本源码、macOS 与 iOS 工程示例来分析`@autoreleasepool`的底层原理。并在最后针对有关`autorelease`和`@autoreleasepool`的一些问题进行解答。

![网络配图.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fde6c60c7078492797be9dc58d721909~tplv-k3u1fbpfcp-zoom-1.image)

## 1. 简单聊聊 ARC 与 MRC

苹果在 iOS 5 中引入了`ARC（Automatic Reference Counting）`自动引用计数内存管理技术，通过`LLVM`编译器和`Runtime`协作来进行自动管理内存。`LLVM`编译器会在编译时在合适的地方为 OC 对象插入`retain`、`release`和`autorelease`代码，省去了在`MRC（Manual Reference Counting）`手动引用计数下手动插入这些代码的工作，减轻了开发者的工作量。

在`MRC`下，当我们不需要一个对象的时候，要调用`release`或`autorelease`方法来释放它。调用`release`会立即让对象的引用计数减 1 ，如果此时对象的引用计数为 0，对象就会被销毁。调用`autorelease`会将该对象添加进自动释放池中，它会在一个恰当的时刻自动给对象调用`release`，所以`autorelease`相当于延迟了对象的释放。

在`ARC`下，`autorelease`方法已被禁用，我们可以使用`__autoreleasing`修饰符修饰对象将对象注册到自动释放池中。详情请参阅[《iOS - 老生常谈内存管理（三）：ARC 面世 —— 所有权修饰符》](https://juejin.im/post/6844904130431942670)。


## 2. 自动释放池
>**官方文档**
>
>The Application Kit creates an autorelease pool on the main thread at the beginning of every cycle of the event loop, and drains it at the end, thereby releasing any autoreleased objects generated while processing an event. If you use the Application Kit, you therefore typically don’t have to create your own pools. If your application creates a lot of temporary autoreleased objects within the event loop, however, it may be beneficial to create “local” autorelease pools to help to minimize the peak memory footprint.

以上是苹果对自动释放池的一段介绍，其意思为：AppKit 和 UIKit 框架在事件循环(`RunLoop`)的每次循环开始时，在主线程创建一个自动释放池，并在每次循环结束时销毁它，在销毁时释放自动释放池中的所有`autorelease`对象。通常情况下我们不需要手动创建自动释放池，但是如果我们在循环中创建了很多临时的`autorelease`对象，则手动创建自动释放池来管理这些对象可以很大程度地减少内存峰值。


![事件循环图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f421094329644eb9cea2ff982fc7b5a~tplv-k3u1fbpfcp-zoom-1.image)


### 创建一个自动释放池
* 在`MRC`下，可以使用`NSAutoreleasePool`或者`@autoreleasepool`。建议使用`@autoreleasepool`，苹果说它比`NSAutoreleasePool`快大约六倍。
```objc
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
// Code benefitting from a local autorelease pool.
[pool release];
```

>**Q：** 释放`NSAutoreleasePool`对象，使用`[pool release]`与`[pool drain]`的区别？<br><br>
>Objective-C 语言本身是支持 GC 机制的，但有平台局限性，仅限于 MacOS 开发中，iOS 开发用的是 RC 机制。在 iOS 的 RC 环境下`[pool release]`和`[pool drain]`效果一样，但在 GC 环境下`drain`会触发 GC 而`release`不做任何操作。使用`[pool drain]`更佳，一是它的功能对系统兼容性更强，二是这样可以跟普通对象的`release`区别开。（注意：苹果在引入`ARC`时称，已在 OS X Mountain Lion v10.8 中弃用`GC`机制，而使用`ARC`替代）

* 而在`ARC`下，已经禁止使用`NSAutoreleasePool`类创建自动释放池，只能使用`@autoreleasepool`。
```objc
@autoreleasepool {
    // Code benefitting from a local autorelease pool.
}
```

## 3. 原理分析
下面我们先通过`macOS`工程来分析`@autoreleasepool`的底层原理。
`macOS`工程中的`main()`函数什么都没做，只是放了一个`@autoreleasepool`。

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {}
    return 0;
}
```
### __AtAutoreleasePool
通过 Clang `clang -rewrite-objc main.m` 将以上代码转换为 C++ 代码。

```objc
struct __AtAutoreleasePool {
    __AtAutoreleasePool() {
        atautoreleasepoolobj = objc_autoreleasePoolPush();
    }
    ~__AtAutoreleasePool() {
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    void * atautoreleasepoolobj;
};

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ 
    { __AtAutoreleasePool __autoreleasepool;  }
    return 0;
}
```
可以看到：
* `@autoreleasepool`底层是创建了一个`__AtAutoreleasePool`结构体对象；
* 在创建`__AtAutoreleasePool`结构体时会在构造函数中调用`objc_autoreleasePoolPush()`函数，并返回一个`atautoreleasepoolobj`(`POOL_BOUNDARY`存放的内存地址，下面会讲到)；
* 在释放`__AtAutoreleasePool`结构体时会在析构函数中调用`objc_autoreleasePoolPop()`函数，并将`atautoreleasepoolobj`传入。

### AutoreleasePoolPage

下面我们进入`Runtime objc4`源码查看以上提到的两个函数的实现。

```objc
// NSObject.mm
void * objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```
可以得知，`objc_autoreleasePoolPush()`和`objc_autoreleasePoolPop()`两个函数其实是调用了`AutoreleasePoolPage`类的两个类方法`push()`和`pop()`。所以`@autoreleasepool`底层就是使用`AutoreleasePoolPage`类来实现的。

下面我们来看一下`AutoreleasePoolPage`类的定义：
```objc
class AutoreleasePoolPage 
{
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)  // EMPTY_POOL_PLACEHOLDER：表示一个空自动释放池的占位符
#   define POOL_BOUNDARY nil                // POOL_BOUNDARY：哨兵对象
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;   // 用来标记已释放的对象
    static size_t const SIZE =              // 每个 Page 对象占用 4096 个字节内存
#if PROTECT_AUTORELEASEPOOL                 // PAGE_MAX_SIZE = 4096
        PAGE_MAX_SIZE;  // must be muliple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);  // Page 的个数

    magic_t const magic;                // 用来校验 Page 的结构是否完整
    id *next;                           // 指向下一个可存放 autorelease 对象地址的位置，初始化指向 begin()
    pthread_t const thread;             // 指向当前线程
    AutoreleasePoolPage * const parent; // 指向父结点，首结点的 parent 为 nil
    AutoreleasePoolPage *child;         // 指向子结点，尾结点的 child  为 nil
    uint32_t const depth;               // Page 的深度，从 0 开始递增
    uint32_t hiwat;
    ......
}
```

>**备注：** 本文使用的是`objc4-756.2`版本源码进行分析。在`objc4-779.1`版本中，`AutoreleasePoolPage`继承自`AutoreleasePoolPageData`，如下。不过原理不变，不影响分析。

```objc
// objc4-779.1
// NSObject.mm
class AutoreleasePoolPage : private AutoreleasePoolPageData
{
	friend struct thread_data_t;

public:
	static size_t const SIZE =
#if PROTECT_AUTORELEASEPOOL
		PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
		PAGE_MIN_SIZE;  // size and alignment, power of 2
#endif
    
private:
	static pthread_key_t const key = AUTORELEASE_POOL_KEY;
	static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
	static size_t const COUNT = SIZE / sizeof(id);

    // EMPTY_POOL_PLACEHOLDER is stored in TLS when exactly one pool is 
    // pushed and it has never contained any objects. This saves memory 
    // when the top level (i.e. libdispatch) pushes and pops pools but 
    // never uses them.
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

#   define POOL_BOUNDARY nil

    // SIZE-sizeof(*this) bytes of contents follow
    ......
}

// NSObject-internal.h
class AutoreleasePoolPage;
struct AutoreleasePoolPageData
{
	magic_t const magic;
	__unsafe_unretained id *next;
	pthread_t const thread;
	AutoreleasePoolPage * const parent;
	AutoreleasePoolPage *child;
	uint32_t const depth;
	uint32_t hiwat;

	AutoreleasePoolPageData(__unsafe_unretained id* _next, pthread_t _thread, AutoreleasePoolPage* _parent, uint32_t _depth, uint32_t _hiwat)
		: magic(), next(_next), thread(_thread),
		  parent(_parent), child(nil),
		  depth(_depth), hiwat(_hiwat)
	{
	}
};
```


整个程序运行过程中，可能会有多个`AutoreleasePoolPage`对象。从它的定义可以得知：
* 自动释放池（即所有的`AutoreleasePoolPage`对象）是以`栈`为结点通过`双向链表`的形式组合而成；
* 自动释放池与线程一一对应；
* 每个`AutoreleasePoolPage`对象占用`4096`字节内存，其中`56`个字节用来存放它内部的成员变量，剩下的空间（`4040`个字节）用来存放`autorelease`对象的地址。

其内存分布图如下：

![AutoreleasePoolPage 双向链表结构](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ef1f81e82c84949a3735c06b27b46fe~tplv-k3u1fbpfcp-zoom-1.image)



下面我们通过源码来分析`push()`、`pop()`以及`autorelease`方法的实现。


#### POOL_BOUNDARY
在分析这些方法之前，先介绍一下`POOL_BOUNDARY`。

* `POOL_BOUNDARY`的前世叫做`POOL_SENTINEL`，称为哨兵对象或者边界对象；
* `POOL_BOUNDARY`用来区分不同的自动释放池，以解决自动释放池嵌套的问题；
* 每当创建一个自动释放池，就会调用`push()`方法将一个`POOL_BOUNDARY`入栈，并返回其存放的内存地址；
* 当往自动释放池中添加`autorelease`对象时，将`autorelease`对象的内存地址入栈，它们前面至少有一个`POOL_BOUNDARY`；
* 当销毁一个自动释放池时，会调用`pop()`方法并传入一个`POOL_BOUNDARY`，会从自动释放池中最后一个对象开始，依次给它们发送`release`消息，直到遇到这个`POOL_BOUNDARY`。


#### push
```objc
    static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) { // 出错时进入调试状态
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);  // 传入 POOL_BOUNDARY 哨兵对象
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }
```
当创建一个自动释放池时，会调用`push()`方法。`push()`方法中调用了`autoreleaseFast()`方法并传入了`POOL_BOUNDARY`哨兵对象。



下面我们来看一下`autoreleaseFast()`方法的实现：
```objc
    static inline id *autoreleaseFast(id obj)
    {
        AutoreleasePoolPage *page = hotPage();     // 双向链表中的最后一个 Page
        if (page && !page->full()) {        // 如果当前 Page 存在且未满
            return page->add(obj);                 // 将 autorelease 对象入栈，即添加到当前 Page 中；
        } else if (page) {                  // 如果当前 Page 存在但已满
            return autoreleaseFullPage(obj, page); // 创建一个新的 Page，并将 autorelease 对象添加进去
        } else {                            // 如果当前 Page 不存在，即还没创建过 Page
            return autoreleaseNoPage(obj);         // 创建第一个 Page，并将 autorelease 对象添加进去
        }
    }
```
`autoreleaseFast()`中先是调用了`hotPage()`方法获得未满的`Page`，从`AutoreleasePoolPage`类的定义可知，每个`Page`的内存大小为`4096`个字节，每当`Page`满了的时候，就会创建一个新的`Page`。`hotPage()`方法就是用来获得这个新创建的未满的`Page`。
`autoreleaseFast()`在执行过程中有三种情况：
* ① 当前`Page`存在且未满时，通过`page->add(obj)`将`autorelease`对象入栈，即添加到当前`Page`中；
* ② 当前`Page`存在但已满时，通过`autoreleaseFullPage(obj, page)`创建一个新的`Page`，并将`autorelease`对象添加进去；
* ③ 当前`Page`不存在，即还没创建过`Page`，通过`autoreleaseNoPage(obj)`创建第一个`Page`，并将`autorelease`对象添加进去。

下面我们来看一下以上提到的三个方法的实现：
```objc
    id *add(id obj)
    {
        assert(!full());
        unprotect();
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        protect();
        return ret;
    }
```
`page->add(obj)`其实就是将`autorelease`对象添加到`Page`中的`next`指针所指向的位置，并将`next`指针指向这个对象的下一个位置，然后将该对象的位置返回。
```objc
    static __attribute__((noinline))
    id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
    {
        // The hot page is full. 
        // Step to the next non-full page, adding a new page if necessary.
        // Then add the object to that page.
        assert(page == hotPage());
        assert(page->full()  ||  DebugPoolAllocation);

        do {
            if (page->child) page = page->child;
            else page = new AutoreleasePoolPage(page);
        } while (page->full());

        setHotPage(page);
        return page->add(obj);
    }
```
`autoreleaseFullPage()`方法中通过`while`循环，通过`Page`的`child`指针找到最后一个`Page`。
* 如果最后一个`Page`未满，就通过`page->add(obj)`将`autorelease`对象添加到最后一个`Page`中；
* 如果最后一个`Page`已满，就创建一个新的`Page`并将该`Page`设置为`hotPage`，通过`page->add(obj)`将`autorelease`对象添加进去。
```objc
    static __attribute__((noinline))
    id *autoreleaseNoPage(id obj)
    {
        // "No page" could mean no pool has been pushed
        // or an empty placeholder pool has been pushed and has no contents yet
        assert(!hotPage());

        bool pushExtraBoundary = false;
        if (haveEmptyPoolPlaceholder()) {
            // We are pushing a second pool over the empty placeholder pool
            // or pushing the first object into the empty placeholder pool.
            // Before doing that, push a pool boundary on behalf of the pool 
            // that is currently represented by the empty placeholder.
            pushExtraBoundary = true;
        }
        else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
            // We are pushing an object with no pool in place, 
            // and no-pool debugging was requested by environment.
            _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                         "autoreleased with no pool in place - "
                         "just leaking - break on "
                         "objc_autoreleaseNoPool() to debug", 
                         pthread_self(), (void*)obj, object_getClassName(obj));
            objc_autoreleaseNoPool(obj);
            return nil;
        }
        else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
            // We are pushing a pool with no pool in place,
            // and alloc-per-pool debugging was not requested.
            // Install and return the empty pool placeholder.
            return setEmptyPoolPlaceholder();
        }

        // We are pushing an object or a non-placeholder'd pool.

        // Install the first page.
        AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
        setHotPage(page);
        
        // Push a boundary on behalf of the previously-placeholder'd pool.
        if (pushExtraBoundary) {
            page->add(POOL_BOUNDARY);
        }
        
        // Push the requested object or pool.
        return page->add(obj);
    }
```
`autoreleaseNoPage()`方法中会创建第一个`Page`。该方法会判断是否有空的自动释放池存在，如果没有会通过`setEmptyPoolPlaceholder()`生成一个占位符，表示一个空的自动释放池。接着创建第一个`Page`，设置它为`hotPage`。最后将一个`POOL_BOUNDARY`添加进`Page`中，并返回`POOL_BOUNDARY`的下一个位置。

>**小结：** 以上就是`push`操作的实现，往自动释放池中添加一个`POOL_BOUNDARY`，并返回它存放的内存地址。接着每有一个对象调用`autorelease`方法，会将它的内存地址添加进自动释放池中。

#### autorelease
`autorelease`方法的函数调用栈如下，详细请参阅[《iOS - 老生常谈内存管理（四）：内存管理方法源码分析》](https://juejin.im/post/6844904131719593998)。
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
`AutoreleasePoolPage`类的`autorelease`方法实现如下：
```
    static inline id autorelease(id obj)
    {
        assert(obj);
        assert(!obj->isTaggedPointer());
        id *dest __unused = autoreleaseFast(obj);
        assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
        return obj;
    }
```

可以看到，调用了`autorelease`方法的对象，也是通过以上解析的`autoreleaseFast()`方法添加进`Page`中。

#### pop
```objc
    static inline void pop(void *token) 
    {
        AutoreleasePoolPage *page;
        id *stop;

        if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
            // Popping the top-level placeholder pool.
            if (hotPage()) {
                // Pool was used. Pop its contents normally.
                // Pool pages remain allocated for re-use as usual.
                pop(coldPage()->begin());
            } else {
                // Pool was never used. Clear the placeholder.
                setHotPage(nil);
            }
            return;
        }

        page = pageForPointer(token);
        stop = (id *)token;
        if (*stop != POOL_BOUNDARY) {
            if (stop == page->begin()  &&  !page->parent) {
                // Start of coldest page may correctly not be POOL_BOUNDARY:
                // 1. top-level pool is popped, leaving the cold page in place
                // 2. an object is autoreleased with no pool
            } else {
                // Error. For bincompat purposes this is not 
                // fatal in executables built with old SDKs.
                return badPop(token);
            }
        }

        if (PrintPoolHiwat) printHiwat();

        page->releaseUntil(stop);

        // memory: delete empty children
        if (DebugPoolAllocation  &&  page->empty()) {
            // special case: delete everything during page-per-pool debugging
            AutoreleasePoolPage *parent = page->parent;
            page->kill();
            setHotPage(parent);
        } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
            // special case: delete everything for pop(top) 
            // when debugging missing autorelease pools
            page->kill();
            setHotPage(nil);
        } 
        else if (page->child) {
            // hysteresis: keep one empty child if page is more than half full
            if (page->lessThanHalfFull()) {
                page->child->kill();
            }
            else if (page->child->child) {
                page->child->child->kill();
            }
        }
    }
```
`pop()`方法的传参`token`即为`POOL_BOUNDARY`对应在`Page`中的地址。当销毁自动释放池时，会调用`pop()`方法将自动释放池中的`autorelease`对象全部释放（实际上是从自动释放池的中的最后一个入栈的`autorelease`对象开始，依次给它们发送一条`release`消息，直到遇到这个`POOL_BOUNDARY`）。`pop()`方法的执行过程如下：

* ① 判断`token`是不是`EMPTY_POOL_PLACEHOLDER`，是的话就清空这个自动释放池；
* ② 如果不是的话，就通过`pageForPointer(token)`拿到`token`所在的`Page`（自动释放池的首个`Page`）；
* ③ 通过`page->releaseUntil(stop)`将自动释放池中的`autorelease`对象全部释放，传参`stop`即为`POOL_BOUNDARY`的地址；
* ④ 判断当前`Page`是否有子`Page`，有的话就销毁。

`pop()`方法中释放`autorelease`对象的过程在`releaseUntil()`方法中，下面来看一下这个方法的实现：
```objc
    void releaseUntil(id *stop) 
    {
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
        
        while (this->next != stop) {
            // Restart from hotPage() every time, in case -release 
            // autoreleased more objects
            AutoreleasePoolPage *page = hotPage();

            // fixme I think this `while` can be `if`, but I can't prove it
            while (page->empty()) {
                page = page->parent;
                setHotPage(page);
            }

            page->unprotect();
            id obj = *--page->next;  // next指针是指向最后一个对象的后一个位置，所以需要先减1
            memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
            page->protect();

            if (obj != POOL_BOUNDARY) {
                objc_release(obj);
            }
        }

        setHotPage(this);

#if DEBUG
        // we expect any children to be completely empty
        for (AutoreleasePoolPage *page = child; page; page = page->child) {
            assert(page->empty());
        }
#endif
    }
```
`releaseUntil()`方法其实就是通过一个`while`循环，从最后一个入栈的`autorelease`对象开始，依次给它们发送一条`release`消息，直到遇到这个`POOL_BOUNDARY`。


#### AutoreleasePoolPage()
我们来看一下创建一个`Page`的过程。`AutoreleasePoolPage()`方法的参数为`parentPage`，新创建的`Page`的`depth`加一，`next`指针的初始位置指向`begin`，将新创建的`Page`的`parent`指针指向`parentPage`。将`parentPage`的`child`指针指向自己，这就形成了`双向链表`的结构。
```objc
    AutoreleasePoolPage(AutoreleasePoolPage *newParent) 
        : magic(), next(begin()), thread(pthread_self()),
          parent(newParent), child(nil), 
          depth(parent ? 1+parent->depth : 0), 
          hiwat(parent ? parent->hiwat : 0)
    { 
        if (parent) {
            parent->check();
            assert(!parent->child);
            parent->unprotect();
            parent->child = this;
            parent->protect();
        }
        protect();
    }
```

#### begin、end、empty、full
下面再来看一下`begin`、`end`、`empty`、`full`这些方法的实现。
* `begin`的地址为：`Page`自己的地址+`Page`对象的大小`56`个字节；
* `end`的地址为：`Page`自己的地址+`4096`个字节；
* `empty`判断`Page`是否为空的条件是`next`地址是不是等于`begin`；
* `full`判断`Page`是否已满的条件是`next`地址是不是等于`end`（栈顶）。
```objc
    id * begin() {
        return (id *) ((uint8_t *)this+sizeof(*this));
    }

    id * end() {
        return (id *) ((uint8_t *)this+SIZE);
    }

    bool empty() {
        return next == begin();
    }

    bool full() { 
        return next == end();
    }
```
> **小结：** 
>* `push`操作是往自动释放池中添加一个`POOL_BOUNDARY`，并返回它存放的内存地址；
> * 接着每有一个对象调用`autorelease`方法，会将它的内存地址添加进自动释放池中。
>* `pop`操作是传入一个`POOL_BOUNDARY`的内存地址，从最后一个入栈的`autorelease`对象开始，将自动释放池中的`autorelease`对象全部释放（实际上是给它们发送一条`release`消息），直到遇到这个`POOL_BOUNDARY `。

## 4. 查看自动释放池的情况
可以通过以下私有函数来查看自动释放池的情况：
```objc
extern void _objc_autoreleasePoolPrint(void);
```

## 5. 使用 macOS 工程示例分析
由于`iOS`工程中，系统在自动释放池中注册了一些对象。为了排除这些干扰，接下来我们通过`macOS`工程代码示例，结合`AutoreleasePoolPage`的内存分布图以及`_objc_autoreleasePoolPrint()`私有函数，来帮助我们更好地理解`@autoreleasepool`的原理。

> **注意：**
>* 由于`ARC`环境下不能调用`autorelease`等方法，所以需要将工程切换为`MRC`环境。
>* 如果使用`ARC`，则可以使用`__autoreleasing`所有权修饰符替代`autorelease`方法。

#### 单个 @autoreleasepool
```objc
int main(int argc, const char * argv[]) {
    _objc_autoreleasePoolPrint();     // print1
    @autoreleasepool {
        _objc_autoreleasePoolPrint(); // print2
        HTPerson *p1 = [[[HTPerson alloc] init] autorelease];
        HTPerson *p2 = [[[HTPerson alloc] init] autorelease];
        _objc_autoreleasePoolPrint(); // print3
    }
    _objc_autoreleasePoolPrint();     // print4
    return 0;
}
```

![内存分布图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2c4cddb72714a51ae8060f15f33c898~tplv-k3u1fbpfcp-zoom-1.image)


```objc
// 自动释放池的情况
objc[68122]: ############## (print1)
objc[68122]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68122]: 0 releases pending. //当前自动释放池中没有任何对象
objc[68122]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68122]: ##############

objc[68122]: ############## (print2)
objc[68122]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68122]: 1 releases pending. //当前自动释放池中有1个对象，这个对象为POOL_BOUNDARY
objc[68122]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68122]: [0x102802038]  ################  POOL 0x102802038  //POOL_BOUNDARY
objc[68122]: ##############

objc[68122]: ############## (print3)
objc[68122]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68122]: 3 releases pending. //当前自动释放池中有3个对象
objc[68122]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68122]: [0x102802038]  ################  POOL 0x102802038  //POOL_BOUNDARY
objc[68122]: [0x102802040]       0x100704a10  HTPerson          //p1
objc[68122]: [0x102802048]       0x10075cc30  HTPerson          //p2
objc[68122]: ##############

objc[68156]: ############## (print4)
objc[68156]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68156]: 0 releases pending. //当前自动释放池中没有任何对象，因为@autoreleasepool作用域结束，调用pop方法释放了对象
objc[68156]: [0x100810000]  ................  PAGE  (hot) (cold)
objc[68156]: ##############
```

### 嵌套 @autoreleasepool
```objc
int main(int argc, const char * argv[]) {
    _objc_autoreleasePoolPrint();             // print1
    @autoreleasepool { //r1 = push()
        _objc_autoreleasePoolPrint();         // print2
        HTPerson *p1 = [[[HTPerson alloc] init] autorelease];
        HTPerson *p2 = [[[HTPerson alloc] init] autorelease];
        _objc_autoreleasePoolPrint();         // print3
        @autoreleasepool { //r2 = push()
            HTPerson *p3 = [[[HTPerson alloc] init] autorelease];
            _objc_autoreleasePoolPrint();     // print4
            @autoreleasepool { //r3 = push()
                HTPerson *p4 = [[[HTPerson alloc] init] autorelease];
                _objc_autoreleasePoolPrint(); // print5
            } //pop(r3)
            _objc_autoreleasePoolPrint();     // print6
        } //pop(r2)
        _objc_autoreleasePoolPrint();         // print7
    } //pop(r1)
    _objc_autoreleasePoolPrint();             // print8
    return 0;
}
```

![内存分布图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c867e1bd1f949eeb8856e30b1bbc17d~tplv-k3u1fbpfcp-zoom-1.image)


```objc
// 自动释放池的情况
objc[68285]: ############## (print1)
objc[68285]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68285]: 0 releases pending. //当前自动释放池中没有任何对象
objc[68285]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68285]: ##############

objc[68285]: ############## (print2)
objc[68285]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68285]: 1 releases pending. //当前自动释放池中有1个对象
objc[68285]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68285]: [0x102802038]  ################  POOL 0x102802038  //POOL_BOUNDARY
objc[68285]: ##############

objc[68285]: ############## (print3)
objc[68285]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68285]: 3 releases pending. //当前自动释放池中有3个对象（1个@autoreleasepool）
objc[68285]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68285]: [0x102802038]  ################  POOL 0x102802038  //POOL_BOUNDARY
objc[68285]: [0x102802040]       0x100707d80  HTPerson          //p1
objc[68285]: [0x102802048]       0x100707de0  HTPerson          //p2
objc[68285]: ##############

objc[68285]: ############## (print4)
objc[68285]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68285]: 5 releases pending. //当前自动释放池中有5个对象（2个@autoreleasepool）
objc[68285]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68285]: [0x102802038]  ################  POOL 0x102802038  //POOL_BOUNDARY
objc[68285]: [0x102802040]       0x100707d80  HTPerson          //p1
objc[68285]: [0x102802048]       0x100707de0  HTPerson          //p2
objc[68285]: [0x102802050]  ################  POOL 0x102802050  //POOL_BOUNDARY
objc[68285]: [0x102802058]       0x1005065b0  HTPerson          //p3
objc[68285]: ##############

objc[68285]: ############## (print5)
objc[68285]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68285]: 7 releases pending. //当前自动释放池中有7个对象（3个@autoreleasepool）
objc[68285]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68285]: [0x102802038]  ################  POOL 0x102802038  //POOL_BOUNDARY
objc[68285]: [0x102802040]       0x100707d80  HTPerson          //p1
objc[68285]: [0x102802048]       0x100707de0  HTPerson          //p2
objc[68285]: [0x102802050]  ################  POOL 0x102802050  //POOL_BOUNDARY
objc[68285]: [0x102802058]       0x1005065b0  HTPerson          //p3
objc[68285]: [0x102802060]  ################  POOL 0x102802060  //POOL_BOUNDARY
objc[68285]: [0x102802068]       0x100551880  HTPerson          //p4
objc[68285]: ##############

objc[68285]: ############## (print6)
objc[68285]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68285]: 5 releases pending. //当前自动释放池中有5个对象（第3个@autoreleasepool已释放）
objc[68285]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68285]: [0x102802038]  ################  POOL 0x102802038
objc[68285]: [0x102802040]       0x100707d80  HTPerson
objc[68285]: [0x102802048]       0x100707de0  HTPerson
objc[68285]: [0x102802050]  ################  POOL 0x102802050
objc[68285]: [0x102802058]       0x1005065b0  HTPerson
objc[68285]: ##############

objc[68285]: ############## (print7)
objc[68285]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68285]: 3 releases pending. //当前自动释放池中有3个对象（第2、3个@autoreleasepool已释放）
objc[68285]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68285]: [0x102802038]  ################  POOL 0x102802038
objc[68285]: [0x102802040]       0x100707d80  HTPerson
objc[68285]: [0x102802048]       0x100707de0  HTPerson
objc[68285]: ##############

objc[68285]: ############## (print8)
objc[68285]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[68285]: 0 releases pending. //当前自动释放池没有任何对象（3个@autoreleasepool都已释放）
objc[68285]: [0x102802000]  ................  PAGE  (hot) (cold)
objc[68285]: ##############
```

### 复杂情况 @autoreleasepool
由`AutoreleasePoolPage`类的定义可知，自动释放池（即所有的`AutoreleasePoolPage`对象）是以`栈`为结点通过`双向链表`的形式组合而成。每当`Page`满了的时候，就会创建一个新的`Page`，并设置它为`hotPage`，而首个`Page`为`coldPage`。接下来我们来看一下多个`Page`和多个`@autoreleasepool`嵌套的情况。

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool { //r1 = push()
        for (int i = 0; i < 600; i++) {
            HTPerson *p = [[[HTPerson alloc] init] autorelease];
        }
        @autoreleasepool { //r2 = push()
            for (int i = 0; i < 500; i++) {
                HTPerson *p = [[[HTPerson alloc] init] autorelease];
            }
            @autoreleasepool { //r3 = push()
                for (int i = 0; i < 200; i++) {
                    HTPerson *p = [[[HTPerson alloc] init] autorelease];
                }
                _objc_autoreleasePoolPrint();
            } //pop(r3)
        } //pop(r2)
    } //pop(r1)
    return 0;
}
```
一个`AutoreleasePoolPage`对象的内存大小为`4096`个字节，它自身成员变量占用内存`56`个字节，所以剩下的`4040`个字节用来存储`autorelease`对象的内存地址。又因为`64bit`下一个`OC`对象的指针所占内存为`8`个字节，所以一个`Page`可以存放`505`个对象的地址。`POOL_BOUNDARY`也是一个对象，因为它的值为`nil`。所以以上代码的自动释放池内存分布图如下所示。


![内存分布图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de9ef264f16148b4a91d8881012ae23c~tplv-k3u1fbpfcp-zoom-1.image)



```objc
objc[69731]: ##############
objc[69731]: AUTORELEASE POOLS for thread 0x1000aa5c0
objc[69731]: 1303 releases pending. //当前自动释放池中有1303个对象（3个POOL_BOUNDARY和1300个HTPerson实例）
objc[69731]: [0x100806000]  ................  PAGE (full)  (cold) /* 第一个PAGE，full代表已满，cold代表coldPage */
objc[69731]: [0x100806038]  ################  POOL 0x100806038    //POOL_BOUNDARY
objc[69731]: [0x100806040]       0x10182a040  HTPerson            //p1
objc[69731]: [0x100806048]       .....................            //...
objc[69731]: [0x100806ff8]       0x101824e40  HTPerson            //p504
objc[69731]: [0x102806000]  ................  PAGE (full)         /* 第二个PAGE */
objc[69731]: [0x102806038]       0x101824e50  HTPerson            //p505
objc[69731]: [0x102806040]       .....................            //...
objc[69731]: [0x102806330]       0x101825440  HTPerson            //p600
objc[69731]: [0x102806338]  ################  POOL 0x102806338    //POOL_BOUNDARY
objc[69731]: [0x102806340]       0x101825450  HTPerson            //p601
objc[69731]: [0x102806348]       .....................            //...
objc[69731]: [0x1028067e0]       0x101825d90  HTPerson            //p1008
objc[69731]: [0x102804000]  ................  PAGE  (hot)         /* 第三个PAGE，hot代表hotPage */
objc[69731]: [0x102804038]       0x101826dd0  HTPerson            //p1009
objc[69731]: [0x102804040]       .....................            //...
objc[69731]: [0x102804310]       0x101827380  HTPerson            //p1100
objc[69731]: [0x102804318]  ################  POOL 0x102804318    //POOL_BOUNDARY
objc[69731]: [0x102804320]       0x101827390  HTPerson            //p1101
objc[69731]: [0x102804328]       .....................            //...
objc[69731]: [0x102804958]       0x10182b160  HTPerson            //p1300
objc[69731]: ##############
```

## 6. 使用 iOS 工程示例分析
从以上`macOS`工程示例可以得知，在`@autoreleasepool`大括号结束的时候，就会调用`Page`的`pop()`方法，给`@autoreleasepool`中的`autorelease`对象发送`release`消息。

那么在`iOS`工程中，方法里的`autorelease`对象是什么时候释放的呢？有系统干预释放和手动干预释放两种情况。
* 系统干预释放是不指定`@autoreleasepool`，所有`autorelease`对象都由主线程的`RunLoop`创建的`@autoreleasepool`来管理。
* 手动干预释放就是将`autorelease`对象添加进我们手动创建的`@autoreleasepool`中。

下面还是在`MRC`环境下进行分析。

### 系统干预释放


我们先来看以下 Xcode 11 版本的`iOS`程序中的`main()`函数，和旧版本的差异。
```objc
// Xcode 11
int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    @autoreleasepool {
        // Setup code that might create autoreleased objects goes here.
        appDelegateClassName = NSStringFromClass([AppDelegate class]);
    }
    return UIApplicationMain(argc, argv, nil, appDelegateClassName);
}
```
```objc
// Xcode 旧版本
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
>**注意：** <br><br>
>**网上对于`iOS`工程的`main()`函数中的`@autoreleasepool`有一种解释：**
><br>在`iOS`工程的`main()`函数中有一个`@autoreleasepool`，这个`@autoreleasepool`负责了应用程序所有`autorelease`对象的释放。<br>
><br>**其实这个解释是错误的。**
><br>如果你的程序使用了`AppKit`或`UIKit`框架，那么主线程的`RunLoop`就会在每次事件循环迭代中创建并处理`@autoreleasepool`。也就是说，应用程序所有`autorelease`对象的都是由`RunLoop`创建的`@autoreleasepool`来管理。而`main()`函数中的`@autoreleasepool`只是负责管理它的作用域中的`autorelease`对象。<br>在以上`《使用 MacOS 工程示例分析》`章节中提到了嵌套`@autoreleasepool`的情况。Xcode 旧版本的`main`函数中是将整个应用程序运行（`UIApplicationMain`）放在`@autoreleasepool`内，而主线程的`RunLoop`就是在`UIApplicationMain`中创建，所以`RunLoop`创建的`@autoreleasepool`是嵌套在`main`函数的`@autoreleasepool`内的。`RunLoop`会在每次事件循环中对自动释放池进行`pop`和`push`（以下会详细讲解），但是它的`pop`只会释放掉它的`POOL_BOUNDARY`之后的对象，它并不会影响到外层即`main`函数中`@autoreleasepool`。
><br><br>**新版本 Xcode 11 中的 main 函数发生了哪些变化？**
><br>旧版本是将整个应用程序运行放在`@autoreleasepool`内，由于`RunLoop`的存在，要`return`即程序结束后`@autoreleasepool`作用域才会结束，这意味着程序结束后`main`函数中的`@autoreleasepool`中的`autorelease`对象才会释放。<br>而在 Xcode 11中，触发主线程`RunLoop`的`UIApplicationMain`函数放在了`@autoreleasepool`外面，这可以保证`@autoreleasepool`中的`autorelease`对象在程序启动后立即释放。正如新版本的`@autoreleasepool`中的注释所写 “`Setup code that might create autoreleased objects goes here.`”（如上代码），可以将`autorelease`对象放在此处。

接着我们来看 “系统干预释放” 情况的示例：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];    
    HTPerson *person = [[[HTPerson alloc] init] autorelease];    
    NSLog(@"%s", __func__);
}

- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];    
    NSLog(@"%s", __func__);
}

- (void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];    
    NSLog(@"%s", __func__);
}

// -[ViewController viewDidLoad]
// -[ViewController viewWillAppear:]
// -[HTPerson dealloc]
// -[ViewController viewDidAppear:]
```

可以看到，调用了`autorelease`方法的`person`对象不是在`viewDidLoad`方法结束后释放，而是在`viewWillAppear`方法结束后释放，说明在`viewWillAppear`方法结束的时候，调用了`pop()`方法释放了`person`对象。其实这是由`RunLoop`控制的，下面来讲解一下`RunLoop`和`@autoreleasepool`的关系。

### RunLoop 与 @autoreleasepool

>学习这个知识点之前，需要先搞懂`RunLoop`的事件循环机制以及它的`6`种活动状态，可以查看我的文章：<br>
[《深入浅出 RunLoop（二）：数据结构》](https://juejin.im/post/6844904073930473480)<br> 
[《深入浅出 RunLoop（三）：事件循环机制》](https://juejin.im/post/6844904073938878477)



`iOS`在主线程的`RunLoop`中注册了两个`Observer`。
* 第1个`Observer`监听了`kCFRunLoopEntry`事件，会调用`objc_autoreleasePoolPush()`；
* 第2个`Observer`<br>
① 监听了`kCFRunLoopBeforeWaiting`事件，会调用`objc_autoreleasePoolPop()`、`objc_autoreleasePoolPush()`；<br>
② 监听了`kCFRunLoopBeforeExit`事件，会调用`objc_autoreleasePoolPop()`。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6b4445283444fa488024565f6a5411f~tplv-k3u1fbpfcp-zoom-1.image)

所以，在`iOS`工程中系统干预释放的`autorelease`对象的释放时机是由`RunLoop`控制的，会在当前`RunLoop`每次循环结束时释放。以上`person`对象在`viewWillAppear`方法结束后释放，说明`viewDidLoad`和`viewWillAppear`方法在同一次循环里。


* `kCFRunLoopEntry`：在即将进入`RunLoop`时，会自动创建一个`__AtAutoreleasePool`结构体对象，并调用`objc_autoreleasePoolPush()`函数。
* `kCFRunLoopBeforeWaiting`：在`RunLoop`即将休眠时，会自动销毁一个`__AtAutoreleasePool`对象，调用`objc_autoreleasePoolPop()`。然后创建一个新的`__AtAutoreleasePool`对象，并调用`objc_autoreleasePoolPush()`。
* `kCFRunLoopBeforeExit`，在即将退出`RunLoop`时，会自动销毁最后一个创建的`__AtAutoreleasePool`对象，并调用`objc_autoreleasePoolPop()`。

### 手动干预释放
我们再来看一下手动干预释放的情况。
```objc
- (void)viewDidLoad {
    [super viewDidLoad];    
    @autoreleasepool {
        HTPerson *person = [[[HTPerson alloc] init] autorelease];  
    }  
    NSLog(@"%s", __func__);
}

- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];    
    NSLog(@"%s", __func__);
}

- (void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];    
    NSLog(@"%s", __func__);
}

// -[HTPerson dealloc]
// -[ViewController viewDidLoad]
// -[ViewController viewWillAppear:]
// -[ViewController viewDidAppear:]
```
可以看到，添加进手动指定的`@autoreleasepool`中的`autorelease`对象，在`@autoreleasepool`大括号结束时就会释放，不受`RunLoop`控制。

## 相关问题



### Q：ARC 环境下，autorelease 对象在什么时候释放？
回到我们最初的面试题，在`ARC`环境下，`autorelease`对象在什么时候释放？我们就分`系统干预释放`和`手动干预释放`两种情况回答。

### Q：ARC 环境下，需不需要手动添加 @autoreleasepool？
AppKit 和 UIKit 框架会在`RunLoop`每次事件循环迭代中创建并处理`@autoreleasepool`，因此，你通常不必自己创建`@autoreleasepool`，甚至不需要知道创建`@autoreleasepool`的代码怎么写。但是，有些情况需要自己创建`@autoreleasepool`。

例如，如果我们需要在循环中创建了很多临时的`autorelease`对象，则手动添加`@autoreleasepool`来管理这些对象可以很大程度地减少内存峰值。比如在`for`循环中`alloc`图片数据等内存消耗较大的场景，需要手动添加`@autoreleasepool`。

>苹果给出了三种需要手动添加`@autoreleasepool`的情况：
>* ① 如果你编写的程序不是基于 UI 框架的，比如说命令行工具；
>* ② 如果你编写的循环中创建了大量的临时对象；
>	<br>你可以在循环内使用`@autoreleasepool`在下一次迭代之前处理这些对象。在循环中使用`@autoreleasepool`有助于减少应用程序的最大内存占用。
>* ③ 如果你创建了辅助线程。
>	<br>一旦线程开始执行，就必须创建自己的`@autoreleasepool`；否则，你的应用程序将存在内存泄漏。


### Q：如果对 NSAutoreleasePool 对象调用 autorelease 方法会发生什么情况？
```objectivec
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    [pool autorelease];
 ```
答：抛出异常`NSInvalidArgumentException`并导致程序`Crash`，异常原因：不能对`NSAutoreleasePool`对象调用`autorelease`。
 ```
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', 
reason: '*** -[NSAutoreleasePool autorelease]: Cannot autorelease an autorelease pool' 
 ```

