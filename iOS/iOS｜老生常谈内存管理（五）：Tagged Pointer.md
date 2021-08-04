在 `objc4` 源码中，我们经常会在函数中看到 `Tagged Pointer`。`Tagged Pointer` 究竟是何方神圣？希望本文能帮助你对它有一定的了解。

## 1. Tagged Pointer 是什么？

以下是苹果在 WWDC2013 《Session 404 Advances in Objective-C》中对`Tagged Pointer`的介绍：
<br>【视频链接：[https://developer.apple.com/videos/play/wwdc2013/404/](https://developer.apple.com/videos/play/wwdc2013/404/)，`Tagged Pointer`部分从 36:50 左右开始】

![](https://user-gold-cdn.xitu.io/2020/4/20/1719736320970dc3?w=1240&h=667&f=png&s=201334)


为了节省内存和提高执行效率，苹果在`64bit`程序中引入了`Tagged Pointer`技术，用于优化`NSNumber`、`NSDate`、`NSString`等小对象的存储。

### 在引入 Tagged Pointer 技术之前
`NSNumber`等对象存储在堆上，`NSNumber`的指针中存储的是堆中`NSNumber`对象的地址值。

#### 从内存占用来看
基本数据类型所需的内存不大。比如`NSInteger`变量，它所占用的内存是与 CPU 的位数有关，如下。在 32 bit 下占用 4 个字节，而在 64 bit 下占用 8 个字节。指针类型的大小通常也是与 CPU 位数相关，一个指针所在 32 bit 下占用 4 个字节，在 64 bit 下占用 8 个字节。
```objc
#if __LP64__ || 0 || NS_BUILD_32_LIKE_64
typedef long NSInteger;
typedef unsigned long NSUInteger;
#else
typedef int NSInteger;
typedef unsigned int NSUInteger;
#endif
```
假设我们通过`NSNumber`对象存储一个`NSInteger`的值，系统实际上会给我们分配多少内存呢？
由于`Tagged Pointer`无法禁用，所以以下将变量`i`设了一个很大的数，以让`NSNumber`对象存储在堆上。

>**备注：** 可以通过设置环境变量`OBJC_DISABLE_TAGGED_POINTERS`为`YES`来禁用`Tagged Pointer`，但如果你这么做，运行就`Crash`。
>```objc
>objc[39337]: tagged pointers are disabled
>(lldb) 
>```
>因为`Runtime`在程序运行时会判断`Tagged Pointer`是否被禁用，如果是的话就会调用`_objc_fatal()`函数杀死进程。所以，虽然苹果提供了`OBJC_DISABLE_TAGGED_POINTERS`这个环境变量给我们，但是`Tagged Pointer`还是无法禁用。
```objc
    NSInteger i = 0xFFFFFFFFFFFFFF;
    NSNumber *number = [NSNumber numberWithInteger:i];
    NSLog(@"%zd", malloc_size((__bridge const void *)(number))); // 32
    NSLog(@"%zd", sizeof(number)); // 8
```
由于`NSNumber`继承自`NSObject`，所有它有`isa`指针，加上内存对齐的处理，系统给`NSNumber`对象分配了 32 个字节内存。通过 LLDB 指令读取它的内存，实际上它并没有用完 32 个字节。


![](https://user-gold-cdn.xitu.io/2020/4/20/1719736320d656f0?w=1060&h=192&f=png&s=55308)
从以上可以得知，在 64 bit 下，如果没有使用`Tagged Pointer`的话，为了使用一个`NSNumber`对象就需要 8 个字节指针内存和 32 个字节对象内存。而直接使用一个`NSInteger`变量只要 8 个字节内存，相差好几倍。但总不能弃用`NSNumber`对象而改用基本数据类型吧。

#### 从效率上来看
为了使用一个`NSNumber`对象，需要在堆上为其分配内存，还要维护它的引用计数，管理它的生命周期，实在是影响执行效率。

### 在引入 Tagged Pointer 技术之后

`NSNumber`等对象的值直接存储在了指针中，不必在堆上为其分配内存，节省了很多内存开销。在性能上，有着 3 倍空间效率的提升以及 106 倍创建和销毁速度的提升。

`NSNumber`等对象的指针中存储的数据变成了`Tag`+`Data`形式（`Tag`为特殊标记，用于区分`NSNumber`、`NSDate`、`NSString`等对象类型；`Data`为对象的值）。这样使用一个`NSNumber`对象只需要 8 个字节指针内存。当指针的 8 个字节不够存储数据时，才会在将对象存储在堆上。


我们再来看一下如果使用了`Tagged Pointer`，系统会给`NSNumber`对象分配多少内存。
```objc
     NSInteger i = 1;
     NSNumber *number = [NSNumber numberWithInteger:i];
     NSLog(@"%zd", malloc_size((__bridge const void *)(number))); // 0
     NSLog(@"%zd", sizeof(number)); // 8
```
可见，使用了`Tagged Pointer`，`NSNumber`对象的值直接存储在了指针上，不会在堆上申请内存。则使用一个`NSNumber`对象只需要指针的 8 个字节内存就够了，大大的节省了内存占用。

## 2. Tagged Pointer 的原理

### 2.1 关闭 Tagged Pointer 的数据混淆

在现在的版本中，为了保证数据安全，苹果对 Tagged Pointer 做了数据混淆，开发者通过打印指针无法判断它是不是一个`Tagged Pointer`，更无法读取`Tagged Pointer`的存储数据。

所以在分析`Tagged Pointer`之前，我们需要先关闭`Tagged Pointer`的数据混淆，以方便我们调试程序。通过设置环境变量`OBJC_DISABLE_TAG_OBFUSCATION`为`YES`。

![](https://user-gold-cdn.xitu.io/2020/4/20/1719736324f0f62d?w=1240&h=698&f=png&s=225861)



### 2.2 MacOS 分析

#### NSNumber
```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSNumber *number1 = @1;
        NSNumber *number2 = @2;
        NSNumber *number3 = @3;
        NSNumber *number4 = @(0xFFFFFFFFFFFFFFFF);
    
        NSLog(@"%p %p %p %p", number1, number2, number3, number4);
    }
    return 0;
}
// 关闭 Tagged Pointer 数据混淆后：0x127 0x227 0x327 0x600003a090e0
// 关闭 Tagged Pointer 数据混淆前：0xaca2838a63a4fb34 0xaca2838a63a4fb04 0xaca2838a63a4fb14 0x600003a090e0
```
从以上打印结果可以看出，`number1～number3`指针为`Tagged Pointer`类型，可以看到对象的值都存储在了指针中，对应`0x1`、`0x2`、`0x3`。而`number4`由于数据过大，指针的`8`个字节不够存储，所以在堆中分配了内存。

>**注意：** `MacOS`与`iOS`平台下的`Tagged Pointer`有差别，下面会讲到。


**0x127 中的 2 和 7 表示什么？**

我们先来看这个`7`，`0x127`为十六进制表示，`7`的二进制为`0111`。
最后一位`1`是`Tagged Pointer`标识位，代表这个指针是`Tagged Pointer`。
前面的`011`是类标识位，对应十进制为`3`，表示`NSNumber`类。

>**备注：** `MacOS`下采用 LSB（Least Significant Bit，即最低有效位）为`Tagged Pointer`标识位，而`iOS`下则采用 MSB（Most Significant Bit，即最高有效位）为`Tagged Pointer`标识位。



可以在`Runtime`源码`objc4`中查看`NSNumber`、`NSDate`、`NSString`等类的标识位。

```objc
// objc-internal.h
{
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6,
    ......
}
```
**代码验证：**

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSNumber *number = @1;
        NSString *string = [NSString stringWithFormat:@"a"];
    
        NSLog(@"%p %p", number, string);
    }
    return 0;
}
// 0x127 0x6115
```
以上打印的`string`指针值为`0x6115`，`61`是`a`的 ASCII 码，最后一位`5`的二进制为`0101`，其中最后一位`1`是代表这个指针是`Tagged Pointer`前面已经说过，`010`对应十进制为`2`，表示`NSString`类。

**0x127 中的 2（即倒数第二位）又代表什么呢？**

倒数第二位用来表示数据类型。

示例：
```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        char a = 1;
        short b = 1;
        int c = 1;
        long d = 1;
        float e = 1.0;
        double f = 1.00;
        
        NSNumber *number1 = @(a);
        NSNumber *number2 = @(b);
        NSNumber *number3 = @(c);
        NSNumber *number4 = @(d);
        NSNumber *number5 = @(e);
        NSNumber *number6 = @(f);

        NSLog(@"%p %p %p %p %p %p", number1, number2, number3, number4, number5, number6);
    }
    return 0;
}
// 0x107 0x117 0x127 0x137 0x147 0x157
```
`Tagged Pointer`倒数第二位对应数据类型：

| Tagged Pointer 倒数第二位  | 对应数据类型           |
|:----------------------|:----------------|
| 0                    |char            |
| 1                    |short     |
| 2                    |int     |
| 3                    |long     |
| 4                    |float     |
| 5                    |double     |

下图是`MacOS`下`NSNumber`的`Tagged Pointer`位视图：

![Tagged Pointer 位视图](https://user-gold-cdn.xitu.io/2020/4/20/17197363250ca111?w=1240&h=336&f=png&s=15874)


#### NSString
接下来我们来分析一下`Tagged Pointer`在`NSString`中的应用。同`NSNumber`一样，在`64 bit`的`MacOS`下，如果一个`NSString`对象指针为`Tagged Pointer`，那么它的后 4 位（0-3）作为标识位，第 4-7 位表示字符串长度，剩余的 56 位就可以用来存储字符串。

示例：
```objc
// MRC 环境
#define HTLog(_var) \
{ \
    NSString *name = @#_var; \
    NSLog(@"%@: %p, %@, %lu", name, _var, [_var class], [_var retainCount]); \
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *a = @"a";
        NSMutableString *b = [a mutableCopy];
        NSString *c = [a copy];
        NSString *d = [[a mutableCopy] copy];
        NSString *e = [NSString stringWithString:a];
        NSString *f = [NSString stringWithFormat:@"f"];
        NSString *string1 = [NSString stringWithFormat:@"abcdefg"];
        NSString *string2 = [NSString stringWithFormat:@"abcdefghi"];
        NSString *string3 = [NSString stringWithFormat:@"abcdefghij"];
        HTLog(a);
        HTLog(b);
        HTLog(c);
        HTLog(d);
        HTLog(e);
        HTLog(f);
        HTLog(string1);
        HTLog(string2);
        HTLog(string3);
    }
    return 0;
}
/*
a: 0x100002038, __NSCFConstantString, 18446744073709551615
b: 0x10071f3c0, __NSCFString, 1
c: 0x100002038, __NSCFConstantString, 18446744073709551615
d: 0x6115, NSTaggedPointerString, 18446744073709551615
e: 0x100002038, __NSCFConstantString, 18446744073709551615
f: 0x6615, NSTaggedPointerString, 18446744073709551615
string1: 0x6766656463626175, NSTaggedPointerString, 18446744073709551615
string2: 0x880e28045a54195, NSTaggedPointerString, 18446744073709551615
string3: 0x10071f6d0, __NSCFString, 1 */
```

从打印结果来看，有三种`NSString`类型：

|类型 | 描述|
|:--|:--|
|__NSCFConstantString|1. 常量字符串，存储在字符串常量区，继承于 __NSCFString。相同内容的 __NSCFConstantString 对象的地址相同，也就是说常量字符串对象是一种单例，可以通过 == 判断字符串内容是否相同。<br>2. 这种对象一般通过字面值`@"..."`创建。如果使用 __NSCFConstantString 来初始化一个字符串，那么这个字符串也是相同的 __NSCFConstantString。
|__NSCFString|1. 存储在堆区，需要维护其引用计数，继承于 NSMutableString。<br>2. 通过`stringWithFormat:`等方法创建的`NSString`对象（且字符串值过大无法使用`Tagged Pointer`存储）一般都是这种类型。
|NSTaggedPointerString|`Tagged Pointer`，字符串的值直接存储在了指针上。

打印结果分析：

|NSString 对象|类型| 分析|
|:--|:--|:--|
|a|__NSCFConstantString| 通过字面量`@"..."`创建
|b| __NSCFString|a 的深拷贝，指向不同的内存地址，被拷贝到堆区
|c| __NSCFConstantString|a 的浅拷贝，指向同一块内存地址
|d| NSTaggedPointerString|单独对 a 进行 copy（如 c），浅拷贝是指向同一块内存地址，所以不会产生`Tagged Pointer`；单独对 a 进行 mutableCopy（如 b），复制出来是可变对象，内容大小可以扩展；而`Tagged Pointer`存储的内容大小有限，因此无法满足可变对象的存储要求。
|e| __NSCFConstantString|使用 __NSCFConstantString 来初始化的字符串
|f| NSTaggedPointerString|通过`stringWithFormat:`方法创建，指针足够存储字符串的值。
|string1| NSTaggedPointerString|通过`stringWithFormat:`方法创建，指针足够存储字符串的值。
|string2| NSTaggedPointerString |通过`stringWithFormat:`方法创建，指针足够存储字符串的值。
|string3| __NSCFString |通过`stringWithFormat:`方法创建，指针不足够存储字符串的值。

可以看到，为`Tagged Pointer`的有`d`、`f`、`string1`、`string2`指针。它们的指针值分别为
`0x6115`、`0x6615 `、`0x6766656463626175`、`0x880e28045a54195`。

其中`0x61`、`0x66`、`0x67666564636261`分别对应字符串的 ASCII 码。

最后一位`5`的二进制为`0101`，最后一位`1`是代表这个指针是`Tagged Pointer`，`010`对应十进制为`2`，表示`NSString`类。

倒数第二位`1`、`1`、`7`、`9`代表字符串长度。

对于`string2`的指针值`0x880e28045a54195`，虽然从指针中看不出来字符串的值，但其也是一个`Tagged Pointer`。


下图是`MacOS`下`NSString`的`Tagged Pointer`位视图：


![Tagged Pointer 位视图](https://user-gold-cdn.xitu.io/2020/4/20/1719736328407082?w=1240&h=364&f=png&s=15856)




### 2.3 iOS 分析


#### NSNumber
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
               
    NSNumber *number1 = @1;
    NSNumber *number2 = @2;
    NSNumber *number3 = @79;
    NSNumber *number4 = @(0xFFFFFFFFFFFFFFFF);
    
    NSLog(@"%p %p %p %p", number1, number2, number3, number4);   
}
// 0xb000000000000012 0xb000000000000022 0xb0000000000004f2 0x600000678480
```

从以上打印结果可以看出，`number1～number3`指针为`Tagged Pointer`类型，可以看到对象的值都存储在了指针中，对应倒数第二位开始的`1`、`2`、`4f`。而`number4`由于数据过大，指针的`8`个字节不够存储，所以在堆中分配了内存。

最后一位用来表示数据类型。

第一位`b`的二进制为`1011`，其中第一位`1`是`Tagged Pointer`标识位。后面的`011`是类标识位，对应十进制为3，表示`NSNumber`类。


下图是`iOS`下`NSNumber`的`Tagged Pointer`位视图：
![Tagged Pointer 位视图](https://user-gold-cdn.xitu.io/2020/4/20/171973632884aac5?w=1240&h=348&f=png&s=15943)




#### NSString
同理，不再分析。

下图是`iOS`下`NSString`的`Tagged Pointer`位视图：
![Tagged Pointer 位视图](https://user-gold-cdn.xitu.io/2020/4/20/171973635afa91d9?w=1240&h=361&f=png&s=16884)


## 3. 如何判断 Tagged Pointer ？
前面已经说过了，通过`Tagged Pointer`标识位。

在`objc4`源码中找到判断`Tagged Pointer`的函数：
```objc
// objc-internal.h
static inline bool 
_objc_isTaggedPointer(const void * _Nullable ptr)
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
```
可以看到，它是将指针值与一个`_OBJC_TAG_MASK`掩码进行按位与运算，查看该掩码：
```objc
#if (TARGET_OS_OSX || TARGET_OS_IOSMAC) && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0  // MacOS
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1  // iOS
#endif

#define _OBJC_TAG_INDEX_MASK 0x7
// array slot includes the tag bit itself
#define _OBJC_TAG_SLOT_COUNT 16
#define _OBJC_TAG_SLOT_MASK 0xf

#define _OBJC_TAG_EXT_INDEX_MASK 0xff
// array slot has no extra bits
#define _OBJC_TAG_EXT_SLOT_COUNT 256
#define _OBJC_TAG_EXT_SLOT_MASK 0xff

#if OBJC_MSB_TAGGED_POINTERS
#   define _OBJC_TAG_MASK (1UL<<63)  // _OBJC_TAG_MASK
#   define _OBJC_TAG_INDEX_SHIFT 60
#   define _OBJC_TAG_SLOT_SHIFT 60
#   define _OBJC_TAG_PAYLOAD_LSHIFT 4
#   define _OBJC_TAG_PAYLOAD_RSHIFT 4
#   define _OBJC_TAG_EXT_MASK (0xfUL<<60)
#   define _OBJC_TAG_EXT_INDEX_SHIFT 52
#   define _OBJC_TAG_EXT_SLOT_SHIFT 52
#   define _OBJC_TAG_EXT_PAYLOAD_LSHIFT 12
#   define _OBJC_TAG_EXT_PAYLOAD_RSHIFT 12
#else
#   define _OBJC_TAG_MASK 1UL       // _OBJC_TAG_MASK
#   define _OBJC_TAG_INDEX_SHIFT 1
#   define _OBJC_TAG_SLOT_SHIFT 0
#   define _OBJC_TAG_PAYLOAD_LSHIFT 0
#   define _OBJC_TAG_PAYLOAD_RSHIFT 4
#   define _OBJC_TAG_EXT_MASK 0xfUL
#   define _OBJC_TAG_EXT_INDEX_SHIFT 4
#   define _OBJC_TAG_EXT_SLOT_SHIFT 4
#   define _OBJC_TAG_EXT_PAYLOAD_LSHIFT 0
#   define _OBJC_TAG_EXT_PAYLOAD_RSHIFT 12
#endif
```
由此我们可以验证：

* `MacOS`下采用 LSB（Least Significant Bit，即最低有效位）为`Tagged Pointer`标识位；
* `iOS`下则采用 MSB（Most Significant Bit，即最高有效位）为`Tagged Pointer`标识位。

而存储在堆空间的对象由于内存对齐，它的内存地址的最低有效位为 0。由此可以辨别`Tagged Pointer`和一般对象指针。

![](https://user-gold-cdn.xitu.io/2020/4/20/171973635ed660f4?w=1240&h=673&f=png&s=180155)



在`objc4`源码中，我们经常会在函数中看到`Tagged Pointer`。比如`objc_msgSend`函数：

```objc
	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame

	cmp	p0, #0			// nil check and tagged pointer check
#if SUPPORT_TAGGED_POINTERS
	b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
#else
	b.eq	LReturnZero
#endif
	ldr	p13, [x0]		// p13 = isa
	GetClassFromIsa_p16 p13		// p16 = class
LGetIsaDone:
	// calls imp or objc_msgSend_uncached
	CacheLookup NORMAL, _objc_msgSend

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
	b.eq	LReturnZero		// nil check

	// tagged
	adrp	x10, _objc_debug_taggedpointer_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_classes@PAGEOFF
	ubfx	x11, x0, #60, #4
	ldr	x16, [x10, x11, LSL #3]
	adrp	x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGE
	add	x10, x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGEOFF
	cmp	x10, x16
	b.ne	LGetIsaDone

	// ext tagged
	adrp	x10, _objc_debug_taggedpointer_ext_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_ext_classes@PAGEOFF
	ubfx	x11, x0, #52, #8
	ldr	x16, [x10, x11, LSL #3]
	b	LGetIsaDone
// SUPPORT_TAGGED_POINTERS
#endif
```
`objc_msgSend`能识别`Tagged Pointer`，比如`NSNumber`的`intValue`方法，直接从指针提取数据，不会进行`objc_msgSend`的三大流程，节省了调用开销。

内存管理相关的，如`retain`方法中调用的`rootRetain`：
```objc
ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    // 如果是 tagged pointer，直接返回 this
    if (isTaggedPointer()) return (id)this; 

    bool sideTableLocked = false;
    bool transcribeToSideTable = false; 

    isa_t oldisa;
    isa_t newisa;
    ......
```
来看一下`isTaggedPointer()`函数实现：
```objc
inline bool 
objc_object::isTaggedPointer() 
{
    return _objc_isTaggedPointer(this);
}
```
该函数就是调用了`_objc_isTaggedPointer`。


## 4. Tagged Pointer 注意点
我们知道，所有`OC`对象都有`isa`指针，而`Tagged Pointer`并不是真正的对象，它没有`isa`指针，所以如果你直接访问`Tagged Pointer`的`isa`成员的话，在编译时将会有如下警告：

![](https://user-gold-cdn.xitu.io/2020/4/20/1719736362798ba4?w=1240&h=689&f=png&s=356230)

对于`Tagged Pointer`，应该换成相应的方法调用，如`isKindOfClass`和`object_getClass`。只要避免在代码中直接访问`Tagged Pointer`的`isa`，即可避免这个问题。

当然现在也不允许我们在代码中直接访问对象的`isa`了，否则编译不通过。

我们通过 LLDB 打印`Tagged Pointer`的`isa`，会提示如下错误：
![](https://user-gold-cdn.xitu.io/2020/4/20/171973636efb98ba?w=1240&h=133&f=png&s=49765)

而打印`OC`对象的`isa`没有问题：
![](https://user-gold-cdn.xitu.io/2020/4/20/171973638570aa55)



## 相关题目

#### Q：执行以下两段代码，有什么区别？
```objc
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch_async(queue, ^{
            self.name = [NSString stringWithFormat:@"abcdefghij"];
        });
    }
```
```objc
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch_async(queue, ^{
            self.name = [NSString stringWithFormat:@"abcdefghi"];
        });
    }
```
心里一万个草泥马🦙奔腾而过～～～，两段代码差别不就是字符串长度少了一位吗，哪有什么差别？

结果一运行，哎呀？第一段代码居然`Crash`，而第二段却没有问题，奇了怪了。
分别打印两段代码的`self.name`类型看看，原来第一段代码中`self.name`为`__NSCFString`类型，而第二段代码中为`NSTaggedPointerString`类型。

我们来看一下第一段代码`Crash`的地方：

![](https://user-gold-cdn.xitu.io/2020/4/20/171973639d1ee91e?w=1240&h=396&f=png&s=262562)

想必你已经猜到了，`__NSCFString`存储在堆上，它是个正常对象，需要维护引用计数的。`self.name`通过`setter`方法为其赋值。而`setter`方法的实现如下：
```objc
- (void)setName:(NSString *)name {
    if(_name != name) {
        [_name release];
        _name = [name retain]; // or [name copy]
    }
}
```
我们异步并发执行`setter`方法，可能就会有多条线程同时执行`[_name release]`，连续`release`两次就会造成对象的过度释放，导致`Crash`。

解决办法：
1. 使用`atomic`属性关键字。
2. 加锁
```objc
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch_async(queue, ^{
            // 加锁
            self.name = [NSString stringWithFormat:@"abcdefghij"];
            // 解锁
        });
    }
```

而第二段代码中的`NSString`为`NSTaggedPointerString`类型，在`objc_release`函数中会判断指针是不是`TaggedPointer`类型，是的话就不对对象进行`release`操作，也就避免了因过度释放对象而导致的`Crash`，因为根本就没执行释放操作。
```objc
__attribute__((aligned(16), flatten, noinline))
void 
objc_release(id obj)
{
    if (!obj) return;
    if (obj->isTaggedPointer()) return;
    return obj->release();
}
```
关于`release`方法的函数调用栈可阅读文章[《iOS - 老生常谈内存管理（四）：内存管理方法源码分析》](https://juejin.im/post/6844904131719593998)。

