## 前言

`ARC`全称`Automatic Reference Counting`，自动引用计数内存管理，是苹果在 iOS 5、OS X Lion 引入的新的内存管理技术。`ARC`是一种编译器功能，它通过`LLVM`编译器和`Runtime`协作来进行自动管理内存。`LLVM`编译器会在编译时在合适的地方为 OC 对象插入`retain`、`release`和`autorelease`代码来自动管理对象的内存，省去了在`MRC`手动引用计数下手动插入这些代码的工作，减轻了开发者的工作量，让开发者可以专注于应用程序的代码、对象图以及对象间的关系上。

本文通过讲解`MRC`到`ARC`的转变、`ARC`规则以及使用注意，来帮助大家掌握`iOS`的内存管理。

下图是苹果官方文档给出的从`MRC`到`ARC`的转变。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf76b8af9fbe4dbcb4ae9da633b3b4e3~tplv-k3u1fbpfcp-zoom-1.image)


## 摘要

`ARC`的工作原理是在编译时添加相关代码，以确保对象能够在必要时存活，但不会一直存活。从概念上讲，它通过为你添加适当的内存管理方法调用来遵循与`MRC`相同的内存管理规则。

为了让编译器生成正确的代码，`ARC`限制了一些方法的使用以及你使用桥接(`toll-free bridging`)的方式，请参阅[ Managing Toll-Free Bridging ](https://juejin.cn/post/6844904130431942670#heading-31)章节。`ARC`还为对象引用和属性声明引入了新的生命周期修饰符。

`ARC`在`Xcode 4.2 for OS X v10.6 and v10.7 (64-bit applications) `以及`iOS 4 and iOS 5`应用程序中提供支持。但`OS X v10.6 and iOS 4`不支持`weak`弱引用。

Xcode 提供了一个迁移工具，可以自动将`MRC`代码转换为`ARC`代码（如删除`retain`和`release`调用），而不用重新再创建一个项目（选择 Edit > Convert > To Objective-C ARC）。迁移工具会将项目中的所有文件转换为使用`ARC`的模式。如果对于某些文件使用`MRC`更方便的话，你可以选择仅在部分文件中使用`ARC`。


## ARC 概述

`ARC`会分析对象的生存期需求，并在编译时自动插入适当的内存管理方法调用的代码，而不需要你记住何时使用`retain`、`release`、`autorelease`方法。编译器还会为你生成合适的`dealloc`方法。一般来说，如果你使用`ARC`，那么只有在需要与使用`MRC`的代码进行交互操作时，传统的 Cocoa 命名约定才显得重要。

Person 类的完整且正确的实现可能如下所示：
```objc
@interface Person : NSObject
@property NSString *firstName;
@property NSString *lastName;
@property NSNumber *yearOfBirth;
@property Person *spouse;
@end
 
@implementation Person
@end
```
默认情况下，对象属性是`strong`。关于`strong`请参阅[ 所有权修饰符 ](https://juejin.cn/post/6844904130431942670#heading-12)章节。

使用`ARC`，你可以这样实现 contrived 方法，如下所示：
```objc
- (void)contrived {
    Person *aPerson = [[Person alloc] init];
    [aPerson setFirstName:@"William"];
    [aPerson setLastName:@"Dudney"];
    [aPerson setYearOfBirth:[[NSNumber alloc] initWithInteger:2011]];
    NSLog(@"aPerson: %@", aPerson);
}
```
`ARC`会负责内存管理，因此 Person 和 NSNumber 对象都不会泄露。

你还可以这样安全地实现 Person 的 takeLastNameFrom: 方法，如下所示：

```objc
- (void)takeLastNameFrom:(Person *)person {
    NSString *oldLastname = [self lastName];
    [self setLastName:[person lastName]];
    NSLog(@"Lastname changed from %@ to %@", oldLastname, [self lastName]);
}
```
`ARC`会确保在 NSLog 语句之前不释放 oldLastName 对象。

### ARC 实施新规则

`ARC`引入了一些在使用其他编译器模式时不存在的新规则。这些规则旨在提供完全可靠的内存管理模型。有时候，它们直接地带来了最好的实践体验，也有时候它们简化了代码，甚至在你丝毫没有关注内存管理问题的时候帮你解决了问题。在`ARC`下必须遵守以下规则，如果违反这些规则，就会编译错误。

* 不能使用 retain / release / retainCount / autorelease
* 不能使用 NSAllocateObject / NSDeallocateObject
* 须遵守内存管理的方法命名规则
* 不能显式调用 dealloc
* 使用 @autoreleasepool 块替代 NSAutoreleasePool
* 不能使用区域（NSZone）
* 对象型变量不能作为 C 语言结构体（struct / union）的成员
* 显式转换 “id” 和 “void *” —— 桥接

#### 不能使用 retain / release / retainCount / autorelease
在`ARC`下，禁止开发者手动调用这些方法，也禁止使用`@selector(retain)`，`@selector(release) `等，否则编译不通过。但你仍然可以对 Core Foundation 对象使用`CFRetain`、`CFRelease`等相关函数（请参阅`《Managing Toll-Free Bridging》`章节）。


#### 不能使用 NSAllocateObject / NSDeallocateObject
在`ARC`下，禁止开发者手动调用这些函数，否则编译不通过。
你可以使用`alloc`创建对象，而`Runtime`会负责`dealloc`对象。


#### 须遵守内存管理的方法命名规则

在`MRC`下，通过 `alloc / new / copy / mutableCopy` 方法创建对象会直接持有对象，我们定义一个 “创建并持有对象” 的方法也必须以 `alloc / new / copy / mutableCopy` 开头命名，并且必须返回给调用方所应当持有的对象。如果在`ARC`下需要与使用`MRC`的代码进行交互，则也应该遵守这些规则。

为了允许与`MRC`代码进行交互操作，`ARC`对方法命名施加了约束：
访问器方法的方法名不能以`new`开头。这意味着你不能声明一个名称以`new`开头的属性，除非你指定一个不同的`getterName`：

```objc
// Won't work:
@property NSString *newTitle;
 
// Works:
@property (getter = theNewTitle) NSString *newTitle;
```




#### 不能显式调用 dealloc
无论在`MRC`还是`ARC`下，当对象引用计数为 0，系统就会自动调用`dealloc`方法。大多数情况下，我们会在`dealloc`方法中移除通知或观察者对象等。

在`MRC`下，我们可以手动调用`dealloc`。但在`ARC`下，这是禁止的，否则编译不通过。

在`MRC`下，我们实现`dealloc`，必须在实现末尾调用`[super dealloc]`。
```objc
// MRC
- (void)dealloc
{
    // 其他处理
    [super dealloc];
}
```
而在`ARC`下，`ARC`会自动对此处理，因此我们不必也禁止写`[super dealloc]`，否则编译错误。
```objc
// ARC
- (void)dealloc
{
    // 其他处理
    [super dealloc]; // 编译错误：ARC forbids explicit message send of 'dealloc'
}
```

#### 使用 @autoreleasepool 块替代 NSAutoreleasePool
在`ARC`下，自动释放池应使用`@autoreleasepool`，禁止使用`NSAutoreleasePool`，否则编译错误。
```objc
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init]; 
// error：'NSAutoreleasePool' is unavailable: not available in automatic reference counting mode
```

>关于`@autoreleasepool`的原理，可以参阅[《iOS - 聊聊 autorelease 和 @autoreleasepool》](https://juejin.im/post/6844904094503567368)。

#### 不能使用区域（NSZone）
对于现在的运行时系统（编译器宏 __ OBJC2 __ 被设定的环境），不管是`MRC`还是`ARC`下，区域（NSZone）都已单纯地被忽略。

>**NSZone：** 摘自《Objective-C 高级编程：iOS 与 OS X 多线程和内存管理》<br><br>
>NSZone 是为防止内存碎片化而引入的结构。对内存分配的区域本身进行多重化管理，根据使用对象的目的、对象的大小分配内存，从而提高了内存管理的效率。
><br>但是，现在的运行时系统已经忽略了区域的概念。运行时系统中的内存管理本身已极具效率，使用区域来管理内存反而会引起内存使用效率低下以及源代码复杂化等问题。
><br>下图是使用多重区域防止内存碎片化的例子：
>![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d69ee2a30d5b4fe29bd9d1313d13a1cb~tplv-k3u1fbpfcp-zoom-1.image)



#### 对象型变量不能作为 C 语言结构体（struct / union）的成员
C 语言的结构体（struct / union）成员中，如果存在 Objective-C 对象型变量，便会引起编译错误。
>**备注：** Xcode10 开始支持在 ARC 模式下在 C Struct 里面引用 Objective-C 对象。之前可以用 Objective-C++。
```objc
struct Data {
    NSMutableArray *mArray;
};
// error：ARC forbids Objective-C objs in struct or unions NSMutableArray *mArray;
```
虽然是 LLVM 编译器 3.0，但不论怎样，C 语言的规约上没有方法来管理结构体成员的生存周期。因为`ARC`把内存管理的工作分配给编译器，所以编译器必须能够知道并管理对象的生存周期。例如 C 语言的自动变量（局部变量）可使用该变量的作用域管理对象。但是对于 C 语言的结构体成员来说，这在标准上就是不可实现的。因此，必须要在结构体释放之前将结构体中的对象类型的成员释放掉，但是编译器并不能可靠地做到这一点，所以对象型变量不能作为 C 语言结构体的成员。


这个问题有以下三种解决方案：
* ① 使用 Objective-C 对象替代结构体。这是最好的解决方案。<br>如果你还是坚持使用结构体，并把对象型变量加入到结构体成员中，可以使用以下两种方案：
* ② 将 Objective-C 对象通过`Toll-Free Bridging`强制转换为`void *`类型，请参阅`《Managing Toll-Free Bridging》`章节。
* ③ 对 Objective-C 对象附加`__unsafe_unretained`修饰符。
```objc
struct Data {
    NSMutableArray __unsafe_unretained *mArray;
};
```
附有`__unsafe_unretained`修饰符的变量不属于编译器的内存管理对象。如果管理时不注意赋值对象的所有者，便有可能遭遇内存泄漏或者程序崩溃。这点在使用时应多加注意。
```objc
struct x { NSString * __unsafe_unretained S; int X; }
```
`__unsafe_unretained`指针在对象被销毁后是不安全的，但它对诸如字符串常量之类的从一开始就确定永久存活的对象非常有用。

#### 显式转换 “id” 和 “void *” —— 桥接
在`MRC`下，我们可以直接在 `id` 和 `void *` 变量之间进行强制转换。
```objc
    id obj = [[NSObject alloc] init];
    void *p = obj;
    id o = p;
    [o release];
```
但在`ARC`下，这样会引起编译报错：在`Objective-C`指针类型`id`和`C`指针类型`void *`之间进行转换需要使用`Toll-Free Bridging`，请参阅[ Managing Toll-Free Bridging ](https://juejin.cn/post/6844904130431942670#heading-31)章节。
```objc
    id obj = [[NSObject alloc] init];
    void *p = obj; // error：Implicit conversion of Objective-C pointer type 'id' to C pointer type 'void *' requires a bridged cast
    id o = p;      // error：Implicit conversion of C pointer type 'void *' to Objective-C pointer type 'id' requires a bridged cast
    [o release];   // error：'release' is unavailable: not available in automatic reference counting mode
```

### 所有权修饰符

`ARC`为对象引入了几个新的生命周期修饰符（我们称为 “所有权修饰符”）以及弱引用功能。弱引用`weak`不会延长它指向的对象的生命周期，并且该对象没有强引用（即`dealloc`）时自动置为`nil`。
<br>你应该利用这些修饰符来管理程序中的对象图。特别是，`ARC`不能防止强引用循环（以前称为`Retain Cycles`，请参阅[《从 MRC 说起 —— 使用弱引用来避免 Retain Cycles》](https://juejin.cn/post/6844904129676984334#heading-17)章节）。明智地使用弱引用`weak`将有助于确保你不会创建循环引用。


#### 属性关键字

`ARC`中引入了新的属性关键字`strong`和`weak`，如下所示：
```objc
// 以下声明同：@property(retain) MyClass *myObject;
@property(strong) MyClass *myObject;
 
// 以下声明类似于：@property（assign）MyClass *myObject；
// 不同的是，如果 MyClass 实例被释放，属性值赋值为 nil，而不像 assign 一样产生悬垂指针。
@property(weak) MyClass *myObject;
```
`strong`和`weak`属性关键字分别对应`__strong`和`__weak`所有权修饰符。在`ARC`下，`strong`是对象类型的属性的默认关键字。

在`ARC`中，对象类型的变量都附有所有权修饰符，总共有以下 4 种。
```objc
__strong
__weak
__unsafe_unretained
__autoreleasing
```
* `__strong`是默认修饰符。只要有强指针指向对象，对象就会保持存活。
* `__weak`指定一个不使引用对象保持存活的引用。当一个对象没有强引用时，弱引用`weak`会自动置为`nil`。
* `__unsafe_unretained`指定一个不使引用对象保持存活的引用，当一个对象没有强引用时，它不会置为`nil`。如果它引用的对象被销毁，就会产生悬垂指针。
* `__autoreleasing`用于表示通过引用（`id *`）传入，并在返回时（`autorelease`）自动释放的参数。

在对象变量的声明中使用所有权修饰符时，正确的格式为：
```objc
    ClassName * qualifier variableName;
```
例如：
```objc
    MyClass * __weak myWeakReference;
    MyClass * __unsafe_unretained myUnsafeReference;
```
其它格式在技术上是不正确的，但编译器会 “原谅”。也就是说，以上才是标准写法。


#### __strong
`__strong`修饰符为强引用，会持有对象，使其引用计数 +1。该修饰符是对象类型变量的默认修饰符。如果我们没有明确指定对象类型变量的所有权修饰符，其默认就为`__strong`修饰符。
```objc
    id obj = [NSObject alloc] init];
    // -> id __strong obj = [NSObject alloc] init];
```

#### __weak
如果单单靠`__strong`完成内存管理，那必然会发生循环引用的情况造成内存泄漏，这时候`__weak`就出来解决问题了。
`__weak`修饰符为弱引用，不会持有对象，对象的引用计数不会增加。`__weak`可以用来防止循环引用。

以下单纯地使用`__weak`修饰符修饰变量，编译器会给出警告，因为`NSObject`的实例创建出来没有强引用，就会立即释放。
```objc
    id __weak weakObj = [[NSObject alloc] init]; // ⚠️Assigning retained object to weak variable; object will be released after assignment
    NSLog(@"%@", obj);
    //  (null)
```
以下`NSObject`的实例已有强引用，再赋值给`__weak`修饰的变量就不会有警告了。
```objc
    id __strong strongObj = [[NSObject alloc] init];
    id __weak weakObj = strongObj;
```
当对象被`dealloc`时，指向该对象的`__weak`变量会被赋值为`nil`。（具体的执行过程可以参阅：[《iOS - 老生常谈内存管理（四）：源码分析内存管理方法》](https://juejin.cn/post/6844904131719593998)）

>**备注：**`__weak`仅在`ARC`中才能使用，在`MRC`中是使用`__unsafe_unretained`修饰符来代替。

#### __unsafe_unretained
`__unsafe_unretained`修饰符的特点正如其名所示，不安全且不会持有对象。

>**注意：** 尽管`ARC`内存管理是编译器的工作，但是附有`__unsafe_unretained`修饰符的变量不属于编译器的内存管理对象。这一点在使用时要注意。

“不会持有对象” 这一特点使它和`__weak`的作用相似，可以防止循环引用。
<br>“不安全“ 这一特点是它和`__weak`的区别，那么它不安全在哪呢？

我们来看代码：
```objc
    id __weak weakObj = nil;
    id __unsafe_unretained uuObj = nil;
    {
        id __strong strongObj = [[NSObject alloc] init];
        weakObj = strongObj;
        unsafeUnretainedObj = strongObj;
        NSLog(@"strongObj:%@", strongObj);
        NSLog(@"weakObj:%@", weakObj);
        NSLog(@"unsafeUnretainedObj:%@", unsafeUnretainedObj);
    }
    NSLog(@"-----obj dealloc-----");
    NSLog(@"weakObj:%@", weakObj);
    NSLog(@"unsafeUnretainedObj:%@", unsafeUnretainedObj); // Crash:EXC_BAD_ACCESS

/*
strongObj:<NSObject: 0x6000038f4340>
weakObj:<NSObject: 0x6000038f4340>
unsafeUnretainedObj:<NSObject: 0x6000038f4340>
-----obj dealloc-----
weakObj:(null)
(lldb) 
*/
```
以上代码运行崩溃。原因是`__unsafe_unretained`修饰的对象在被销毁之后，指针仍然指向原对象地址，我们称它为 “悬垂指针”。这时候如果继续通过指针访问原对象的话，就会导致`Crash`。而`__weak`修饰的对象在被释放之后，会将指向该对象的所有`__weak`指针变量全都置为`nil`。这就是`__unsafe_unretained`不安全的原因。所以，在使用`__unsafe_unretained`修饰符修饰的对象时，需要确保它未被销毁。

>**Q：** 既然 __weak 更安全，那么为什么已经有了 __weak 还要保留 __unsafe_unretained ？
>* `__weak`仅在`ARC`中才能使用，而`MRC`只能使用`__unsafe_unretained`；
>* `__unsafe_unretained`主要跟 C 代码交互；
>* `__weak`对性能会有一定的消耗，当一个对象`dealloc`时，需要遍历对象的`weak`表，把表里的所有`weak`指针变量值置为`nil`，指向对象的`weak`指针越多，性能消耗就越多。所以`__unsafe_unretained`比`__weak`快。当明确知道对象的生命周期时，选择`__unsafe_unretained`会有一些性能提升。
><br>A 持有 B 对象，当 A 销毁时 B 也销毁。这样当 B 存在，A 就一定会存在。而 B 又要调用 A 的接口时，B 就可以存储 A 的`__unsafe_unretained`指针。
>比如，MyViewController 持有 MyView，MyView 需要调用 MyViewController 的接口。MyView 中就可以存储`__unsafe_unretained MyViewController *_viewController`。
><br><br>虽然这种性能上的提升是很微小的。但当你很清楚这种情况下，`__unsafe_unretained`也是安全的，自然可以快一点就是一点。而当情况不确定的时候，应该优先选用`__weak`。


#### __autoreleasing 

##### 自动释放池
首先讲一下自动释放池，在`ARC`下已经禁止使用`NSAutoreleasePool`类创建自动释放池，而用`@autoreleasepool`替代。

* `MRC`下可以使用`NSAutoreleasePool`或者`@autoreleasepool`。建议使用`@autoreleasepool`，苹果说它比`NSAutoreleasePool`快大约六倍。
```objc
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    // Code benefitting from a local autorelease pool.
    [pool release]; // [pool drain]
```
>**Q：** 释放`NSAutoreleasePool`对象，使用`[pool release]`与`[pool drain]`的区别？<br><br>
>Objective-C 语言本身是支持 GC 机制的，但有平台局限性，仅限于 MacOS 开发中，iOS 开发用的是 RC 机制。在 iOS 的 RC 环境下`[pool release]`和`[pool drain]`效果一样，但在 GC 环境下`drain`会触发 GC 而`release`不做任何操作。使用`[pool drain]`更佳，一是它的功能对系统兼容性更强，二是这样可以跟普通对象的`release`区别开。（注意：苹果已在 OS X Mountain Lion v10.8 中弃用`GC`机制，而使用`ARC`替代）

* `ARC`下只能使用`@autoreleasepool`。
```objc
    @autoreleasepool {
        // Code benefitting from a local autorelease pool.
    }
```
>关于`@autoreleasepool`的底层原理，可以参阅[《iOS - 聊聊 autorelease 和 @autoreleasepool》](https://juejin.im/post/6844904094503567368)。


##### __autoreleasing 使用

在`MRC`中我们可以给对象发送`autorelease`消息来将它注册到`autoreleasepool`中。而在`ARC`中`autorelease`已禁止调用，我们可以使用`__autoreleasing`修饰符修饰对象将对象注册到`autoreleasepool`中。
```objc
    @autoreleasepool {
        id __autoreleasing obj = [[NSObject alloc] init];
    }
```
以上代码在`MRC`中等价于:

```objc
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    id obj = [[NSObject alloc] init];
    [obj autorelease];
    [pool drain];
    // 或者
    @autoreleasepool {
        id obj = [[NSObject alloc] init];
        [obj autorelease];
    }
```

##### __autoreleasing 是二级指针类型的默认修饰符

前面我们说过，对象指针的默认所有权修饰符是`__strong`。
而二级指针类型（`ClassName **`或`id *`）的默认所有权修饰符是`__autoreleasing`。如果我们没有明确指定二级指针类型的所有权修饰符，其默认就会附加上`__autoreleasing`修饰符。

比如，我们经常会在开发中使用到`NSError`打印错误信息，我们通常会在方法的参数中传递`NSError`对象的指针。如`NSString`的`stringWithContentsOfFile`类方法，其参数`NSError **`使用的是`__autoreleasing`修饰符。
```objc
    NSString *str = [NSString stringWithContentsOfFile:<#(nonnull NSString *)#>
                                              encoding:<#(NSStringEncoding)#> 
                                              error:<#(NSError *__autoreleasing  _Nullable * _Nullable)#>];
```
示例：我们声明一个参数为`NSError **`的方法，但不指定其所有权修饰符。
```objc
- (BOOL)performOperationWithError:(NSError **)error;
```
接着我们尝试调用该方法，发现智能提示中的参数`NSError **`附有`__autoreleasing`修饰符。可见，如果我们没有明确指定二级指针类型的所有权修饰符，其默认就会附加上`__autoreleasing`修饰符。
```objc
    NSError *error = nil;
    BOOL result = [self performOperationWithError:<#(NSError *__autoreleasing *)#>];
```

##### 注意

需要注意的是，赋值给二级指针类型时，所有权修饰符必须一致，否则会编译错误。

```objc
    NSError *error = nil;
    NSError **error1 = &error;                 // error：Pointer to non-const type 'NSError *' with no explicit ownersh
    NSError *__autoreleasing *error2 = &error; // error：Initializing 'NSError *__autoreleasing *' with an expression of type 'NSError *__strong *' changes retain/release properties of pointer
    NSError *__weak *error3 = &error;          // error：Initializing 'NSError *__weak *' with an expression of type 'NSError *__strong *' changes retain/release properties of pointer
    NSError *__strong *error4 = &error;        // 编译通过
```
```objc
    NSError *__weak error = nil;
    NSError *__weak *error1 = &error;          // 编译通过
```
```objc
    NSError *__autoreleasing error = nil;
    NSError *__autoreleasing *error1 = &error; // 编译通过
```
我们前面说过，二级指针类型的默认修饰符是`__autoreleasing`。那为什么我们调用方法传入`__strong`修饰的参数就可以编译通过呢？

```objc
    NSError *__strong error = nil;
    BOOL result = [self performOperationWithError:<#(NSError *__autoreleasing *)#>];
```
其实，编译器自动将我们的代码转化成了以下形式：
```objc
    NSError *__strong error = nil;
    NSError *__autoreleasing tmp = error;
    BOOL result = [self performOperationWithError:&tmp];
    error = tmp;
```
可见，当局部变量声明（`__strong`）和参数（`__autoreleasing`）之间不匹配时，会导致编译器创建临时变量。你可以将显示地指定局部变量所有权修饰符为`__autoreleasing`~~或者不显式指定（因为其默认就为`__autoreleasing`）~~，或者显示地指定参数所有权修饰符为`__strong`，来避免编译器创建临时变量。

```objc
- (BOOL)performOperationWithError:(NSError *__strong *)error;
```
但是在`MRC`引用计数内存管理规则中：使用`alloc/new/copy/mutableCopy`等方法创建的对象，创建并持有对象；其他情况创建对象但并不持有对象。由谁创建就由谁负责释放，很明显这里的 NSError为了在使用参数获得对象时，遵循此规则，我们应该指定二级指针类型参数修饰符为`__autoreleasing`。

在`《从 MRC 说起 —— 你不持有通过引用返回的对象》`章节中也说到，Cocoa 中的一些方法指定通过引用返回对象（即，它们采用`ClassName **`或`id *`类型的参数），常见的就是使用`NSError`对象。当你调用这些方法时，你不会创建该`NSError`对象，因此你不持有该对象，也无需释放它。而`__strong`代表持有对象，因此应该使用`__autoreleasing`。


另外，我们在显示指定`__autoreleasing`修饰符时，必须注意对象变量要为自动变量（包括局部变量、函数以及方法参数），否则编译不通过。
```objc
    static NSError __autoreleasing *error = nil; // Global variables cannot have __autoreleasing ownership
```

#### 使用所有权修饰符来避免循环引用
前面已经说过`__weak`和`__unsafe_unretained `修饰符可以用来循环引用，这里再来啰嗦几句。

如果两个对象互相强引用，就产生了循环引用，导致两个对象都不能被销毁，内存泄漏。或者多个对象，每个对象都强引用下一个对象直到回到第一个，产生大环循环引用，这些对象也均不能被销毁。
>在`ARC`中，“循环引用” 是指两个对象都通过`__strong`持有对方。

解决 “循环引用” 问题就是采用 “断环” 的方式，让其中一方持有另一方的弱引用。同`MRC`，父对象对它的子对象持有强引用，而子对象对父对象持有弱引用。

>在`ARC`中，“弱引用” 是指`__weak`或`__unsafe_unretained `。

##### delegate 避免循环引用
`delegate`避免循环引用，就是在委托方声明`delegate`属性时，使用`weak`关键字。
```objc
@property (nonatomic, weak) id<protocolName> delegate;
```
##### block 避免循环引用
>**Q：** 为什么 block 会产生循环引用？

* ① 相互循环引用： 如果当前`block`对当前对象的某一成员变量进行捕获的话，可能会对它产生强引用。根据`block`的变量捕获机制，如果`block`被拷贝到堆上，且捕获的是对象类型的`auto`变量，则会连同其所有权修饰符一起捕获，所以如果对象是`__strong`修饰，则`block`会对它产生强引用（如果`block`在栈上就不会强引用）。而当前`block`可能又由于当前对象对其有一个强引用，就产生了相互循环引用的问题；
* ② 大环引用： 我们如果使用`__block`的话，在`ARC`下可能会产生循环引用（`MRC`则不会）。由于`__block`修饰符会将变量包装成一个对象，如果`block`被拷贝到堆上，则会直接对`__block`变量产生强引用，而`__block`如果修饰的是对象的话，会根据对象的所有权修饰符做出相应的操作，形成强引用或者弱引用。如果对象是`__strong`修饰（如`__block id x`），则`__block`变量对它产生强引用（在`MRC`下则不会），如果这时候该对象是对`block`持有强引用的话，就产生了大环引用的问题。在`ARC`下可以通过断环的方式去解除循环引用，可以在`block`中将指针置为`nil`（`MRC`不会循环引用，则不用解决）。但是有一个弊端，如果该`block`一直得不到调用，循环引用就一直存在。

**ARC 下的解决方式：**

* 用`__weak`或者`__unsafe_unretained`解决：
```objc
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        NSLog(@"%p",weakSelf);
    };
```
```objc
    __unsafe_unretained id uuSelf = self;
    self.block = ^{
        NSLog(@"%p",uuSelf);
    };
```
![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/961eb7c4378741698b08abc341fa02c5~tplv-k3u1fbpfcp-zoom-1.image)


>**注意**：`__unsafe_unretained`会产生悬垂指针，建议使用`weak`。

&emsp;对于 non-trivial cycles，我们需要这样做：
```objc
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if(!strongSelf) return;
        NSLog(@"%p",weakSelf);
    };
```

* 用`__block`解决（必须要调用`block`）：
缺点：必须要调用`block`，而且`block`里要将指针置为`nil`。如果一直不调用`block`，对象就会一直保存在内存中，造成内存泄漏。
```objc
    __block id blockSelf = self;
    self.block = ^{
        NSLog(@"%p",blockSelf);
        blockSelf = nil;
    };
    self.block();
```
![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3beaf083a8e4469fb31f4814bd711460~tplv-k3u1fbpfcp-zoom-1.image)

**MRC 下的解决方式：**
* 用`__unsafe_unretained`解决：同`ARC`。
* 用`__block`解决（在`MRC`下使用`__block`修饰对象类型，在`block`内部不会对该对象进行`retain`操作，所以在`MRC`环境下可以通过`__block`解决循环引用的问题）
```objc
    __block id blockSelf = self;
    self.block = ^{
        NSLog(@"%p",blockSelf);
    };
```
更多关于`block`的内容，可以参阅[《OC - Block 详解》](https://juejin.im/post/6844904070746996750#heading-2)。


### 属性
说到属性，不得不提一下`@synthesize`和`@dynamic`这两个指令。

#### @synthesize 和 @dynamic
* `@property`：帮我们自动生成属性的`setter`和`getter`方法的声明。
* `@synthesize`：帮我们自动生成`setter`和`getter`方法的实现以及下划线成员变量。
以前我们需要手动对每个`@property`添加`@synthesize`，而在 iOS 6 之后 LLVM 编译器引入了 “`property autosynthesis`”，即属性自动合成。换句话说，就是编译器会自动为每个`@property`添加`@synthesize`。
>**Q：** `@synthesize`现在有什么作用呢？<br><br>
>如果我们同时重写了`setter`和`getter`方法，则编译器就不会为这个`@property`添加`@synthesize`，这时候就不存在下划线成员变量，所以我们需要手动添加`@synthesize`。
>```objc
>@synthesize propertyName = _propertyName;
>```

有时候我们不希望编译器为我们`@synthesize`，我们希望在程序运行过程中再去决定该属性存取方法的实现，就可以使用`@dynamic`。

* `@dynamic` ：告诉编译器不用自动进行`@synthesize`，等到运行时再添加方法实现，但是它不会影响`@property`生成的`setter`和`getter`方法的声明。`@dynamic`是 OC 为动态运行时语言的体现。动态运行时语言与编译时语言的区别：动态运行时语言将函数决议推迟到运行时，编译时语言在编译器进行函数决议。
```objc
@dynamic propertyName;
```

#### 属性“内存管理”关键字与所有权修饰符的对应关系

属性“内存管理”关键字|所有权修饰符
:--|:--
assign | __unsafe_unretained
unsafe_unretained | __unsafe_unretained
weak | __weak
retain | __strong
strong | __strong
copy | __strong

更多关于属性关键字的内容，可以参阅[《OC - 属性关键字和所有权修饰符》](https://juejin.im/post/6844904067425124366)。


### 管理 Outlets 的模式在 iOS 和 OS X 平台下变得一致

在`ARC`下，`iOS`和`OS X`平台中声明`outlets`的模式变得一致。你应该采用的模式为：在`nib`或者`storyboard`中，除了来自文件所有者的`top-level`对象的`outlets`应该使用`strong`，其它情况下应该使用`weak`修饰`outlets`。（详情见 [Nib Files](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/LoadingResources/CocoaNibs/CocoaNibs.html#//apple_ref/doc/uid/10000051i-CH4) in [Resource Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/LoadingResources/Introduction/Introduction.html#//apple_ref/doc/uid/10000051i)）

### 栈变量初始化为 nil
使用`ARC`，`strong`、`weak`和`autoreleasing`的栈变量现在会默认初始化为`nil`。例如：
```objc
- (void)myMethod {
    NSString *name;
    NSLog(@"name: %@", name);
}
```
打印`name`的值为`null`，而不是程序`Crash`。

### 使用编译器标志启用和禁用 ARC
使用`-fobjc-arc`编译器标志启用`ARC`。如果对你来说，某些文件使用`MRC`更方便，那你可以仅对部分文件使用`ARC`。对于使用`ARC`作为默认方式的项目，可以使用`-fno-objc-arc`编译器标志为指定文件禁用`ARC`。如下图所示：

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01353d2ec5c348849c8d55764bb6b55b~tplv-k3u1fbpfcp-zoom-1.image)


`ARC`支持 Xcode 4.2 及更高版本、OS X v10.6 及更高版本 (64-bit applications) 、iOS 4 及更高版本。但 OS X v10.6 和 iOS 4 不支持`weak`弱引用。Xcode 4.1 及更早版本中不支持`ARC`。


## Managing Toll-Free Bridging

### Toll-Free Bridging

你在项目中可能会使用到`Core Foundation`样式的对象，它可能来自`Core Foundation`框架或者采用`Core Foundation`约定标准的其它框架如`Core Graphics`。

编译器不会自动管理`Core Foundation`对象的生命周期，你必须根据`Core Foundation`内存管理规则调用`CFRetain`和`CFRelease`。请参阅[《Memory Management Programming Guide for Core Foundation》](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i)。

在`MRC`下，我们可以直接在`Objective-C`指针类型`id`和`C`指针类型`void *`之间进行强制转换，如`Foundation`对象和`Core Foundation`对象进行转换。由于都是手动管理内存，无须关心内存管理权的移交问题。

在`ARC`下，进行`Foundation`对象和`Core Foundation`对象的类型转换，需要使用`Toll-Free Bridging`（`桥接`）告诉编译器对象的所有权语义。你要选择使用`__bridge`、`__bridge_retained`、`__bridge_transfer`这三种`桥接`方案中的一种来确定对象的内存管理权移交问题，它们的作用分别如下：

桥接方案|用法|内存管理权|引用计数
:--:|:--:|:--:|:--:
__bridge               | F <=> CF |  不改变                           | 不改变
__bridge_retained<br>(或 CFBridgingRetain) | F => CF   |  ARC 管理 => 手动管理<br>(你负责调用 CFRelease 或<br>相关函数来放弃对象所有权） | +1 
__bridge_transfer<br>(或 CFBridgingRelease) | CF => F   |  手动管理 => ARC 管理<br>(ARC 负责放弃对象的所有权)  | +1 再 -1，不改变


* **__bridge**（常用）：不改变对象的内存管理权所有者。
本来由`ARC`管理的`Foundation`对象，转换成`Core Foundation`对象后继续由`ARC`管理；
本来由开发者手动管理的`Core Foundation`对象，转换成`Foundation`对象后继续由开发者手动管理。
下面以`NSMutableArray`对象和`CFMutableArrayRef`对象为例：

```objc
    // 本来由 ARC 管理
    NSMutableArray *mArray = [[NSMutableArray alloc] init];
    // 转换后继续由 ARC 管理            
    CFMutableArrayRef cfMArray = (__bridge CFMutableArrayRef)(mArray); 

    // 本来由开发者手动管理
    CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL); 
    // 转换后继续由开发者手动管理
    NSMutableArray *mArray = (__bridge NSMutableArray *)(cfMArray); 
    ...
    // 在不需要该对象时需要手动释放
    CFRelease(cfMArray); 
```
`__bridge`的安全性与赋值给`__unsafe_unretained`修饰符相近甚至更低，如果使用不当没有注意对象的释放，就会因悬垂指针而导致`Crash`。

`__bridge`转换后不改变对象的引用计数，比如我们将`id`类型转换为`void *`类型，我们在使用`void *`之前该对象被销毁了，那么我们再使用`void *`访问该对象肯定会`Crash`。所以`void *`指针建议立即使用，如果我们要保存这个`void *`指针留着以后使用，那么建议使用`__bridge_retain`。

而在使用`__bridge`将`void *`类型转换为`id`类型时，一定要注意此时对象的内存管理还是由开发者手动管理，记得在不需要对象时进行释放，否则内存泄漏！

以下给出几个 “使用`__bridge`将`void *`类型转换为`id`类型” 的示例代码，要注意转换后还是由开发者手动管理内存，所以即使离开作用域，该对象还保存在内存中。
```objc
    // 使用 __strong
    CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
    NSMutableArray *mArray = (__bridge NSMutableArray *)(cfMArray);
    
    NSLog(@"%ld", CFGetRetainCount(cfMArray));     // 2, 因为 mArray 是 __strong, 所以增加引用计数
    NSLog(@"%ld", _objc_rootRetainCount(mArray));  // 1, 但是使用 _objc_rootRetainCount 打印出来是 1 ？
     
    // 在不需要该对象时进行释放
    CFRelease(cfMArray); 
    NSLog(@"%ld", CFGetRetainCount(cfMArray));     // 1, 因为 __strong 作用域还没结束，还有强指针引用着
    NSLog(@"%ld", _objc_rootRetainCount(mArray));  // 1
    
   // 在 __strong 作用域结束前，还可以访问该对象
   // 等 __strong 作用域结束，该对象就会销毁，再访问就会崩溃
```
```objc
    // 使用 __strong
    CFMutableArrayRef cfMArray;
    {
        cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
        NSMutableArray *mArray = (__bridge NSMutableArray *)(cfMArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray));  // 2, 因为 mArray 是 __strong, 所以增加引用计数
    }
    NSLog(@"%ld", CFGetRetainCount(cfMArray));      // 1, __strong 作用域结束
    CFRelease(cfMArray); // 释放对象，否则内存泄漏
    // 可以使用 CFShow 函数打印 CF 对象
    CFShow(cfMArray);    // 再次访问就会崩溃
```
```objc
    // 使用 __weak
    CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
    NSMutableArray __weak *mArray = (__bridge NSMutableArray *)(cfMArray);
    
    NSLog(@"%ld", CFGetRetainCount(cfMArray));     // 1, 因为 mArray 是 __weak, 所以不增加引用计数
    NSLog(@"%ld", _objc_rootRetainCount(mArray));  // 1
    
    /*
     * 使用 mArray
     */
    
    // 在不需要该对象时进行释放
    CFRelease(cfMArray);
    
    NSLog(@"%@",mArray);  // nil, 这就是使用 __weak 的好处，在指向的对象被销毁的时候会自动置指针为 nil，再次访问也不会崩溃
```
```objc
    // 使用 __weak
    NSMutableArray __weak *mArray;
    {
        CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
        mArray = (__bridge NSMutableArray *)(cfMArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray));  // 1, 因为 mArray 是 __weak, 所以不增加引用计数
    }
    CFMutableArrayRef cfMArray =  (__bridge CFMutableArrayRef)(mArray);
    NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 1, 可见即使出了作用域，对象也还没释放，因为内存管理权在我们
    
    CFRelease(cfMArray); // 释放对象，否则内存泄漏
```

* **__bridge_retained**：用在`Foundation`对象转换成`Core Foundation`对象时，进行`ARC`内存管理权的剥夺。
本来由`ARC`管理的`Foundation`对象，转换成`Core Foundation`对象后，`ARC`不再继续管理该对象，需要由开发者自己手动释放该对象，否则会发生内存泄漏。
```objc
    // 本来由 ARC 管理
    NSMutableArray *mArray = [[NSMutableArray alloc] init];       
    // 转换后由开发者手动管理              
    CFMutableArrayRef cfMArray = (__bridge_retained CFMutableArrayRef)(mArray); 
//    CFMutableArrayRef cfMArray = (CFMutableArrayRef)CFBridgingRetain(mArray); // 另一种等效写法
    ...
    CFRelease(cfMArray);  // 在不需要该对象的时候记得手动释放
```
`__bridge_retained`顾名思义会对对象`retain`，使转换赋值的变量也持有该对象，对象的引用计数 +1。由于转换后由开发者进行手动管理，所以再不需要该对象的时候记得调用`CFRelease`释放对象，否则内存泄漏。
```objc
    id obj = [[NSObject alloc] init];
    void *p = (__bridge_retained void *)(obj);
```
以上代码如果在`MRC`下相当于：
```objc
    id obj = [[NSObject alloc] init];

    void *p = obj;
    [(id)p retain];
```
**查看引用计数**

在`ARC`下，我们可以使用`_objc_rootRetainCount`函数查看对象的引用计数（该函数有时候不准确）。对于`Core Foundation`对象，我们可以使用`CFGetRetainCount`函数查看引用计数。
```objc
uintptr_t _objc_rootRetainCount(id obj);
```
打印上面代码的`obj`对象的引用计数，发现其引用计数确实增加。
```objc
    id obj = [[NSObject alloc] init];
    NSLog(@"%ld", _objc_rootRetainCount(obj)); // 1
    void *p = (__bridge_retained void *)(obj);
    NSLog(@"%ld", _objc_rootRetainCount(obj)); // 2
    NSLog(@"%ld", CFGetRetainCount(p));        // 2
```
以下给出几个示例代码：
```objc
    CFMutableArrayRef cfMArray;
    {
        NSMutableArray *mArray = [[NSMutableArray alloc] init];
        cfMArray = (__bridge_retained CFMutableArrayRef)(mArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 2, 因为 mArray 是 __strong, 而且使用了 __bridge_retained
    }
    NSLog(@"%ld", CFGetRetainCount(cfMArray));     // 1, __strong 作用域结束
    CFRelease(cfMArray); // 在不需要使用的时候释放，防止内存泄漏
```
如果将上面的代码由`__bridge_retained`改为使用`__bridge`会怎样？
```objc
    CFMutableArrayRef cfMArray;
    {
        NSMutableArray *mArray = [[NSMutableArray alloc] init];
        cfMArray = (__bridge CFMutableArrayRef)(mArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 1, 因为 mArray 是 __strong, 且使用 __bridge 还是由 ARC 管理，不增加引用计数
    }
    NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 程序崩溃，因为对象已销毁
```


* **__bridge_transfer**：用在`Core Foundation`对象转换成`Foundation`对象时，进行内存管理权的移交。
本来由开发者手动管理的`Core Foundation`对象，转换成`Foundation`对象后，将内存管理权移交给`ARC`，开发者不用再关心对象的释放问题，不用担心内存泄漏。

```objc
    // 本来由开发者手动管理
    CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
    // 转换后由 ARC 管理
    NSMutableArray *mArray = (__bridge_transfer NSMutableArray *)(cfMArray);
//    NSMutableArray *mArray = CFBridgingRelease(cfMArray); // 另一种等效写法
```

`__bridge_transfer`作用如其名，移交内存管理权。它的实现跟`__bridge_retained`相反，会`release`被转换的变量持有的对象，但同时它在赋值给转换的变量时会对对象进行`retain`，所以引用计数不变。也就是说，对于`Core Foundation`引用计数语义而言，对象是释放的，但是`ARC`保留了对它的引用。
```objc
    id obj = (__bridge_transfer void *)(p);
```
以上代码如果在`MRC`下相当于：
```objc
    id obj = (id)p;
    [obj retain];
    [(id)p release];
```
下面也给出一个示例代码：
```objc
    CFMutableArrayRef cfMArray;
    {
        cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
        NSMutableArray *mArray = (__bridge_transfer NSMutableArray *)(cfMArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray));    // 1, 因为 cfMArray 指针指向的对象存在，所以仍然可以通过该指针访问
        NSLog(@"%ld", _objc_rootRetainCount(mArray)); // 1, mArray 为 __strong
    }
    // __strong 作用域结束，ARC 对该对象进行了释放
    NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 再次访问就会崩溃
```
如果将上面的代码由`__bridge_transfer`改为使用`__bridge`会怎样？
其实在`__bridge`讲解中已经给出了示例代码，如果不释放就会造成内存泄漏。



以上提到了可以替代`__bridge_retained`和`__bridge_transfer`的两个函数：`CFBridgingRetain`和`CFBridgingRelease`，下面我们来看一下函数实现：

```objc
/* Foundation - NSObject.h */
#if __has_feature(objc_arc)  // ARC

// After using a CFBridgingRetain on an NSObject, the caller must take responsibility for calling CFRelease at an appropriate time.
NS_INLINE CF_RETURNS_RETAINED CFTypeRef _Nullable CFBridgingRetain(id _Nullable X) {
    return (__bridge_retained CFTypeRef)X;
}

NS_INLINE id _Nullable CFBridgingRelease(CFTypeRef CF_CONSUMED _Nullable X) {
    return (__bridge_transfer id)X;
}

#else // MRC

// This function is intended for use while converting to ARC mode only.
NS_INLINE CF_RETURNS_RETAINED CFTypeRef _Nullable CFBridgingRetain(id _Nullable X) {
    return X ? CFRetain((CFTypeRef)X) : NULL;
}

// Casts a CoreFoundation object to an Objective-C object, transferring ownership to ARC (ie. no need to CFRelease to balance a prior +1 CFRetain count). NS_RETURNS_RETAINED is used to indicate that the Objective-C object returned has +1 retain count.  So the object is 'released' as far as CoreFoundation reference counting semantics are concerned, but retained (and in need of releasing) in the view of ARC. This function is intended for use while converting to ARC mode only.
NS_INLINE id _Nullable CFBridgingRelease(CFTypeRef CF_CONSUMED _Nullable X) NS_RETURNS_RETAINED {
    return [(id)CFMakeCollectable(X) autorelease];
}

#endif
```
可以看到在`ARC`下，这两个函数就是使用了`__bridge_retained`和`__bridge_transfer`。


>**小结：** 在`ARC`下，必须恰当使用`Toll-Free Bridging`（`桥接`）在`Foundation`对象和`Core Foundation`对象之间进行类型转换，否则可能会导致内存泄漏。<br><br>
**建议：**
>* 将`Foundation`对象转为`Core Foundation`对象时，如果我们立即使用该`Core Foundation`对象，使用`__bridge`；如果我们想保存着以后使用，使用`__bridge_retained`，但是要记得在使用完调用`CFRelease`释放对象。
>* 将`Core Foundation`对象转为`Foundation`对象时，使用`__bridge_transfer`。


### 编译器处理从 Cocoa 方法返回的 CF 对象
编译器知道返回`Core Foundation`对象的`Objective-C`方法遵循历史 Cocoa 命名约定。例如，编译器知道，在`iOS`中，`UIColor`的`CGColor`方法返回的`CGColor`并不持有（因为方法名不是以`alloc/new/copy/mutableCopy`开头）。所以你仍然必须使用适当的类型转换，如以下示例所示：
```objc
    NSMutableArray *colors = [NSMutableArray arrayWithObject:(id)[[UIColor darkGrayColor] CGColor]];
    [colors addObject:(id)[[UIColor lightGrayColor] CGColor]];
```
否则编译警告：
```objc
    NSMutableArray *colors = [NSMutableArray arrayWithObject:[[UIColor darkGrayColor] CGColor]];
    // Incompatible pointer types sending 'CGColorRef _Nonnull' (aka 'struct CGColor *') to parameter of type 'id _Nonnull'
    // 不兼容的指针类型，将 CGColorRef（又称 struct CGColor *）作为 id 类型参数传入
```

### 使用桥接转换函数参数

当在函数调用中在`Objective-C`和`Core Foundation`对象之间进行转换时，需要告诉编译器参数的所有权语义。`Core Foundation`对象的所有权规则请参阅[《Memory Management Programming Guide for Core Foundation》](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i)。`Objective-C`的所有权规则请参阅`《从 MRC 说起 —— 内存管理策略》`章节。

如下实例所示，`NSArray`对象作为`CGGradientCreateWithColors`函数的参数传入，它的所有权不需要传递给该函数，因此需要使用`__bridge`进行强制转换。
```objc
    NSArray *colors = <#An array of colors#>;
    CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, (__bridge CFArrayRef)colors, locations);
```
以下实例中使用了`Core Foundation`对象以及`Objective-C`和`Core Foundation`对象之间进行转换，同时需要注意`Core Foundation`对象的内存管理。
```objc
- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceGray();
    CGFloat locations[2] = {0.0, 1.0};
    NSMutableArray *colors = [NSMutableArray arrayWithObject:(id)[[UIColor darkGrayColor] CGColor]];
    [colors addObject:(id)[[UIColor lightGrayColor] CGColor]];
    CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, (__bridge CFArrayRef)colors, locations);
    CGColorSpaceRelease(colorSpace);  // Release owned Core Foundation object.
    CGPoint startPoint = CGPointMake(0.0, 0.0);
    CGPoint endPoint = CGPointMake(CGRectGetMaxX(self.bounds), CGRectGetMaxY(self.bounds));
    CGContextDrawLinearGradient(ctx, gradient, startPoint, endPoint,
                                kCGGradientDrawsBeforeStartLocation | kCGGradientDrawsAfterEndLocation);
    CGGradientRelease(gradient);  // Release owned Core Foundation object.
}
```



## ARC 的实现
`ARC`仅仅依靠`LLVM`编译器是无法完成内存管理工作的，它还需要`Runtime`的支持。就比如`__weak`修饰符，如果没有`Runtime`，那么在对象`dealloc`时就不会将`__weak`变量置为`nil`。
`ARC`由以下工具、库来实现：
* clang（LLVM 编译器）3.0 以上
* objc4 Objective-C 运行时库 493.9 以上





## 转换项目时的常见问题
除了以上说明的几点`ARC`新规则以外，`ARC`下还要注意以下几个问题，也是`MRC`转换到`ARC`项目的常见问题：

* `ARC`要求你在`init`方法中将`[super init]`的结果分配给`self`。
```objc
    self = [super init];
    if (self) {
    ...
```
* 你无法实现自定义`retain`或`release`方法。
	<br>实现自定义`retain`或`release`方法会破坏`weak`弱指针。你想要这么做的原因可能如下：
   * ① 性能
		<br>请不要再这样做了，`NSObject`的`retain`和`release`方法的实现现在已经足够快了。如果你仍然发现有问题，请提交错误给苹果。
   * ② 实现自定义`weak`弱指针系统
		<br>请改用`__weak`。
   * ③ 实现单例类
		<br>请改用`shared instance`模式。或者，使用类方法替代实例方法，这样可以避免创建对象。
* “直接赋值” 的实例变量变成强引用了。

在`ARC`之前，实例变量是弱引用（非持有引用） —— 直接将对象分配给实例变量并不延长对象的生命周期。为了使属性变`strong`，你通常会实现或使用`@synthesize`合成 “调用适当内存管理方法” 的访问器方法。相反，有时你为了维持一个弱引用，你可能会像以下实例这样实现访问器方法。
```objc
@interface MyClass : Superclass {
    id thing; // Weak reference.
}
// ...
@end

@implementation MyClass
- (id)thing {
    return thing;
}
- (void)setThing:(id)newThing {
    thing = newThing;
}
// ...
@end
```
对于`ARC`，实例变量默认是`strong`强引用 —— 直接将对象分配给实例变量会延长对象的生命周期。迁移工具在将`MRC`代码转换为`ARC`代码时，无法确定它该使用`strong`还是`weak`，所以默认使用`strong`。 若要保持与`MRC`下一致，必须将实例变量使用`__weak`修饰，或使用`weak`关键字的属性。
```objc
@interface MyClass : Superclass {
    id __weak thing;
}
// ...
@end

@implementation MyClass
- (id)thing {
    return thing;
}
- (void)setThing:(id)newThing {
    thing = newThing;
}
// ...
@end
```
或者：
```objc
@interface MyClass : Superclass
@property (weak) id thing;
// ...
@end

@implementation MyClass
@synthesize thing;
// ...
@end
```


## ARC 补充

### __weak 黑科技 

在所有权修饰符中我们简单介绍了`__weak`修饰符。实际上，除了在`MRC`下无法使用`__weak`修饰符以外，还有其他无法使用`__weak`修饰符的情况。

例如，有一些类是不支持`__weak`修饰符的，比如`NSMachPort`。这些类重写了`retain / release`并实现该类独自的引用计数机制。但是赋值以及使用附有`__weak`修饰符的变量都必须恰当地使用 objc4 运行时库中的函数，因此独自实现引用计数机制的类大多不支持`__weak`修饰符。
```objc
    NSMachPort __weak *port = [NSMachPort new]; 
    // 编译错误：Class is incompatible with __weak references 类与弱引用不兼容
```
不支持`__weak`修饰符的类，其类的声明中添加了`NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE`宏，该宏的定义如下。
```objc
// Marks classes which cannot participate in the ARC weak reference feature.
#if __has_attribute(objc_arc_weak_reference_unavailable)
#define NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE __attribute__((objc_arc_weak_reference_unavailable))
#else
#define NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE
#endif
```
如果将不支持`__weak`的类的对象赋值给`__weak`修饰符的变量，一旦编译器检测出来就会报告编译错误。但是在 Cocoa 框架中，不支持`__weak`修饰符的类极为罕见，因此没有必要太过担心。

`__weak`黑科技来了！！！！！

还有一种情况也不能使用`__weak`修饰符。就是当对象的`allowsWeakReference`/`retainWeakReference`实例方法返回`NO`时，这两个方法的声明如下：
```objc
- (BOOL)allowsWeakReference;
- (BOOL)retainWeakReference;
```
这两个方法的默认实现是返回`YES`。

如果我们在类中重写了`allowsWeakReference`方法并返回`NO`，那么如果我们将该类的实例对象赋值给`__weak`修饰符的变量，那么程序就会`Crash`。

例如我们在`HTPerson`类中做了此操作，则以下代码运行就会`Crash`。
```objc
    HTPerson __weak *p = [[HTPerson alloc] init];
```
```objc
// 无法对 HTPerson 类的实例持有弱引用。可能是此对象被过度释放，或者正在销毁。
objc[18094]: Cannot form weak reference to instance (0x600001d7c2a0) of class HTPerson. It is possible that this object was over-released, or is in the process of deallocation.
(lldb) 
```
所以，对于所有`allowsWeakReference`方法返回`NO`的类的实例都绝对不能使用`__weak`修饰符。

另外，如果实例对象的`retainWeakReference`方法返回`NO`，那么赋值该对象`__weak`修饰符的变量将为`nil`，代表无法通过`__weak`变量访问该对象。

比如以下示例代码：
```objc
    HTPerson *p1 = [[HTPerson alloc] init];    
    HTPerson __weak *p2 = p1;
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);

/* 打印如下：
   <HTPerson: 0x600002e0dd20>
   <HTPerson: 0x600002e0dd20>
   <HTPerson: 0x600002e0dd20>
   <HTPerson: 0x600002e0dd20>
   <HTPerson: 0x600002e0dd20> */
```



由于`p1`为`__strong`持有对象的强引用，所以在`p1`作用域结束前，该对象都存在，使用`__weak`修饰的`p2`访问该对象没问题。

下面在`HTPerson`类中重写`retainWeakReference`方法：
```objc
@interface HTPerson ()
{
    NSUInteger count;
}

@implementation HTPerson
- (BOOL)retainWeakReference
{
    if (++count > 3) {
        return NO;
    }
    return [super retainWeakReference];    
}
@end
```

再次运行以上代码，发现从第 4 次开始，通过`__weak`变量就无法访问到对象，因为这时候`retainWeakReference`方法返回值为`NO`。
```objc
    HTPerson *p1 = [[HTPerson alloc] init];    
    HTPerson __weak *p2 = p1;
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);

/* 打印如下：
   <HTPerson: 0x600003e23ba0>
   <HTPerson: 0x600003e23ba0>
   <HTPerson: 0x600003e23ba0>
   (null)
   (null) */
```

### 查看引用计数

在`ARC`下，我们可以使用`_objc_rootRetainCount`函数查看对象的引用计数。
```objc
uintptr_t _objc_rootRetainCount(id obj);
```
但实际上并不能完全信任该函数取得的数值。对于已释放的对象以及不正确的对象地址，有时也返回 “1”。另外，在多线程中使用对象的引用计数数值，因为存在竞争条件的问题，所以取得的数值不一定完全可信。
虽然在调试中`_objc_rootRetainCount`函数很有用，但最好在了解其所具有的问题的基础上来使用。


## 苹果对 ARC 一些问题的回答

>**Q：** 我应该如何看待 ARC ？它将 retains/releases 调用的代码放在哪了？

尝试不要去思考`ARC`将`retains/releases`调用的代码放在哪里，而是思考应用程序算法，思考对象的`strong`和`weak`指针、所有权、以及可能产生的循环引用。

>**Q：** 我还需要为我的对象编写 dealloc 方法吗？

有时候需要。
因为`ARC`不会自动处理`malloc/free`、`Core Foundation`对象的生命周期管理、文件描述符等等，所以你仍然可以通过编写`dealloc`方法来释放这些资源。
你不必（实际上不能）释放实例变量，但可能需要对系统类和其他未使用`ARC`编写的代码调用`[self setDelegate:nil]`。
`ARC`下的`dealloc`方法中不需要且不允许调用`[super dealloc]`，`Runtime`会自动处理。

>**Q：** ARC 中仍然可能存在循环引用吗？

是的，`ARC`自动`retain/release`，也继承了循环引用问题。幸运的是，迁移到`ARC`的代码很少开始泄漏，因为属性已经声明是否`retain`。

>**Q：** block 是如何在 ARC 中工作的？

在`ARC`下，编译器会根据情况自动将栈上的`block`复制到堆上，比如`block`作为函数返回值时，这样你就不必再调用`Block Copy`。

需要注意的一件事是，在`ARC`下，`NSString * __block myString`这样写的话，`block`会对`NSString`对象强引用，而不是造成悬垂指针问题。如果你要和`MRC`保持一致，请使用`__block NSString * __unsafe_unretained myString`或（更好的是）使用`__block NSString * __weak myString`。

>**Q：** 我可以在 ARC 下创建一个 retained 指针的 C 数组吗？

可以，如下示例所示：
```objc
// Note calloc() to get zero-filled memory.
__strong SomeClass **dynamicArray = (__strong SomeClass **)calloc(entries, sizeof(SomeClass *));
for (int i = 0; i < entries; i++) {
     dynamicArray[i] = [[SomeClass alloc] init];
}
 
// When you're done, set each entry to nil to tell ARC to release the object.
for (int i = 0; i < entries; i++) {
     dynamicArray[i] = nil;
}
free(dynamicArray);
```
这里有一些注意点：

* 在某些情况下，你需要编写`__strong SomeClass **`，因为默认是`__autoreleasing SomeClass **`。
* 分配的内存区域必须初始化为 0（`zero-filled`）。
* 在`free`数组之前，必须将每个元素赋值为`nil`（`memset`或`bzero`将不起作用）。
* 你应该避免使用`memcpy`或`realloc`。

>**Q：** ARC 速度上慢吗？

不。编译器有效地消除了许多无关的`retain/release`调用，并且已经投入了大量精力来加速 Objective-C 运行时。特别的是，当方法的调用者是`ARC`代码时，常见的 “`return a retain/autoreleased object`” 模式要快很多，并且实际上并不将对象放入自动释放池中。

需要注意的一个问题是，优化器不是在常见的调试配置中运行的，所以预计在`-O0`模式下将会比`-Os`模式下看到更多的`retain/release`调用。


>**Q：** ARC 在 ObjC++ 模式下工作吗？

是。你甚至可以在类和容器中放置`strong/weak`的`id`对象。`ARC`编译器会在复制构造函数和析构函数等中合成`retain/release`逻辑以使其运行。


>**Q：** 哪些类不支持 weak 弱引用？

你当前无法创建对以下类的实例的`weak`弱引用：NSATSTypesetter、NSColorSpace、NSFont、NSMenuView、NSParagraphStyle、NSSimpleHorizontalTypesetter 和 NSTextView。

**注意：** 此外，在 OS X v10.7 中，你无法创建对 NSFontManager，NSFontPanel、NSImage、NSTableCellView、NSViewController、NSWindow 和 NSWindowController 实例的`weak`弱引用。此外，在 OS X v10.7 中，AV Foundation 框架中的任何类都不支持`weak`弱引用。

此外，你无法在`ARC`下创建 NSHashTable、NSMapTable 和 NSPointerArray 类的实例的`weak`弱引用。

>**Q：** 当我继承一个使用了 NSCopyObject 的类，如 NSCell 时，我需要做些什么？

没什么特别的。`ARC`会关注以前必须显式添加额外`retain`的情况。使用`ARC`，所有的复制方法只需要复制实例变量就可以了。


>**Q：** 我可以对指定文件选择退出`ARC`而使用`MRC`吗？

可以。当你迁移项目到`ARC`或创建一个`ARC`项目时，所以`Objective-C`源文件的默认编译器标志将设置为`-fobjc-arc`，你可以使用`-fno-objc-arc`编译器标志为指定的类禁用`ARC`。操作如下图所示：

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dd790dabea04b83a407c6feafb7c246~tplv-k3u1fbpfcp-zoom-1.image)

>**Q：** 在 Mac 上是否弃用了 GC (Garbage Collection) 机制？

OS X Mountain Lion v10.8 中不推荐使用`GC`机制，并且将在 OS X 的未来版本中删除`GC`机制。`ARC`是推荐的替代技术。为了帮助现有应用程序迁移，Xcode 4.3 及更高版本中的`ARC`迁移工具支持将使用`GC`的 OS X 应用程序迁移到`ARC`。

**注意：** 对于面向 Mac App Store 的应用，Apple 强烈建议你尽快使用`ARC`替换`GC`，因为 Mac App Store Guidelines 禁止使用已弃用的技术，否则不会通过审核，详情请参阅 [Mac App Store Review Guidelines](http://developer.apple.com/appstore/mac/resources/approval/guidelines.html)。


