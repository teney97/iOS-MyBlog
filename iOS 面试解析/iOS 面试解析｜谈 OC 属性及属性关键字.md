
本期 [「 iOS 摸鱼周报 第十九期」](https://mp.weixin.qq.com/s/dtyozlqCO7PcpyGhx2qB5g) 面试解析模块讲解的知识点是 `OC 属性及属性关键字`，于是笔者又对 OC 属性及属性关键字做了一次总结，大部分内容都摘自以往博客，加了一些新的理解和经验总结。


## @property、@synthesize 和 @dynamic

### @property

属性用于封装对象中数据，属性的本质是 ivar + setter + getter。

可以用 @property 语法来声明属性。@property 会帮我们自动生成属性的 setter 和 getter 方法的声明。

### @synthesize

帮我们自动生成 setter 和 getter 方法的实现以及 _ivar。

你还可以通过 @synthesize 来指定实例变量名字，如果你不喜欢默认的以下划线开头来命名实例变量的话。但最好还是用默认的，否则影响可读性。

如果不想令编译器合成存取方法，则可以自己实现。如果你只实现了其中一个存取方法 setter or getter，那么另一个还是会由编译器来合成。但是需要注意的是，如果你实现了属性所需的全部方法（如果属性是 readwrite 则需实现 setter and getter，如果是 readonly 则只需实现 getter 方法），那么编译器就不会自动进行 @synthesize，这时候就不会生成该属性的实例变量，需要根据实际情况自己手动 @synthesize 一下。

```objectivec
@synthesize ivar = _ivar;
```

### @dynamic

告诉编译器不用自动进行 @synthesize，你会在运行时再提供这些方法的实现，无需产生警告，但是它不会影响 @property 生成的 setter 和 getter 方法的声明。@dynamic 是 OC 为动态运行时语言的体现。动态运行时语言与编译时语言的区别：动态运行时语言将函数决议推迟到运行时，编译时语言在编译器进行函数决议。

```objectivec
@dynamic ivar;
```

以前我们需要手动对每个 @property 添加 @synthesize，而在 iOS 6 之后 LLVM 编译器引入了 `property autosynthesis`，即属性自动合成。换句话说，就是编译器会自动为每个 @property 添加 @synthesize。

那你可能就会问了，@synthesize 现在有什么用呢？

1. 如果我们同时重写了 setter 和 getter 方法，则编译器就不会自动为这个 @property 添加 @synthesize，这时候就不存在 _ivar，所以我们需要手动添加 @synthesize；
2. 如果该属性是 readonly，那么只要你重写了 getter 方法，`property autosynthesis` 就不会执行，同样的你需要手动添加 @synthesize 如果你需要的话，看你这个属性是要定义为存储属性还是计算属性吧；
3. 实现协议中要求的属性。

此外需要注意的是，分类当中添加的属性，也不会 `property autosynthesis` 哦。因为类的内存布局在编译的时候会确定，但是分类是在运行时才加载并将数据合并到宿主类中的，所以分类当中不能添加成员变量，只能通过关联对象间接实现分类有成员变量的效果。如果你给分类添加了一个属性，但没有手动给它实现 getter、setter（如果属性是 readonly 则不需要实现）的话，编译器就会给你警告啦 `Property 'ivar' requires method 'ivar'、'setIvar:' to be defined - use @dynamic or provide a method implementation in this category`，编译器已经告诉我们了有两种解决方式来消除警告：

1. 在这个分类当中提供该属性 getter、setter 方法的实现
2. 使用 @dynamic 告诉编译器 getter、setter 方法的实现在运行时自然会有，您就不用操心了。当然在这里 @dynamic 只是消除了警告而已，如果你没有在运行时动态添加方法实现的话，那么调用该属性的存取方法还是会 Crash。


## 属性修饰符分类


分类|属性关键字
--|--
原子性|atomic、nonatomic
读写权限|readwrite、readonly
方法名|setter、getter
内存管理|assign、weak、unsafe_unretained、retain、strong、copy
可空性|(nullable、_Nullable 、__nullable)、<br>(nonnull、_Nonnull、__nonnull)、<br>(null_unspecified、_Null_unspecified 、__null_unspecified)、<br>null_resettable
类属性|class


### 原子性

属性关键字|用法
-- |--
atomic|原子性（默认），编译器会自动生成互斥锁（以前是自旋锁，后面改为了互斥锁），对 setter 和 getter 方法进行加锁，可以保证属性的赋值和取值的原子性操作是线程安全的，但不包括操作和访问。<br>比如说 atomic 修饰的是一个数组的话，那么我们对数组进行赋值和取值是可以保证线程安全的。但是如果我们对数组进行操作，比如说给数组添加对象或者移除对象，是不在 atomic 的负责范围之内的，所以给被 atomic 修饰的数组添加对象或者移除对象是没办法保证线程安全的。
nonatomic|非原子性，一般属性都用 nonatomic 进行修饰，因为 atomic 耗时。

### 读写权限

属性关键字|用法
--|--
readwrite|可读可写（默认），同时生成 setter 方法和 getter 方法的声明和实现。
readonly|只读，只生成 getter 方法的声明和实现。为了达到封装的目的，我们应该只在确有必要时才将属性对外暴露，并且尽量把对外暴露的属性设为 readonly。如果这时候想在对象内部通过 setter 修改属性，可以在类扩展中将属性重新声明为 readwrite；如果仅在对象内部通过 _ivar 修改，则不需要重新声明为 readwrite。


### 方法名

属性关键字|用法
--|--
setter|可以指定生成的 setter 方法名，如 setter = setName。这个关键字笔者在给分类添加属性的时候会用得比较多，为了避免分类方法“覆盖”同名的宿主类（或者其它分类）方法的问题，一般我们都会加前缀，比如 bb_ivar，但是这样生成的 setter 方法名就不美观了（为 setBb_ivar），于是就使用到了 setter 关键字 `@property (nonatomic, strong, setter = bb_setIvar:) NSObject *bb_ivar;`
getter|可以指定生成的 getter 方法名，如 getter = getName。使用示例：`@property (nonatomic, assign, getter = isEnabled) BOOL enabled;`

### 内存管理

属性关键字|用法
--|--
assign|1. 既可以修饰基本数据类型，也可以修饰对象类型；<br>2. setter 方法的实现是直接赋值，一般用于基本数据类型 ；<br>3. 修饰基本数据类型，如 NSInteger、BOOL、int、float 等；<br>4. 修饰对象类型时，不增加其引用计数；<br>5. 会产生悬垂指针（悬垂指针：assign 修饰的对象在被释放之后，指针仍然指向原对象地址，该指针变为悬垂指针。这时候如果继续通过该指针访问原对象的话，就可能导致程序崩溃）。
weak|1. 只能修饰对象类型；<br>2. ARC 下才能使用；<br>3. 修饰弱引用，不增加对象引用计数，主要可以用于避免循环引用；<br>4. weak 修饰的对象在被释放之后，会自动将指针置为 nil，不会产生悬垂指针；<br>5. 对于视图，通常还是用在 xib 和 storyboard 上；代码中对于有必要进行 remove 的视图也可以使用 weak，这样 remove 之后会自动置为 nil。
unsafe_unretained|1. 既可以修饰基本数据类型，也可以修饰对象类型；<br>2. MRC 下经常使用，ARC 下基本不用；<br>3. 同 weak，区别就在于 unsafe_unretained 会产生悬垂指针；<br>4. weak 对性能会有一定的消耗，当一个对象 dealloc 时，需要遍历对象的 weak 表，把表里的所有 weak 指针变量值置为 nil，指向对象的 weak 指针越多，性能消耗就越多。所以 unsafe_unretained 比 weak 快。当明确知道对象的生命周期时，选择 unsafe_unretained 会有一些性能提升。比如 A 持有 B 对象，当 A 销毁时 B 也销毁。这样当 B 存在，A 就一定会存在。而 B 又要调用 A 的接口时，B 就可以存储 A 的 unsafe_unretained 指针。虽然这种性能上的提升是很微小的。但当你很清楚这种情况下，unsafe_unretained 也是安全的，自然可以快一点就是一点。而当情况不确定的时候，应该优先选用 weak。「比如笔者封装了一个  DisplayLink 类，在内部使用 NSProxy 中间变量来更好地避免循环引用，将 displayLink 的 target 设置为 proxy，而 proxy 当中需要调用 displayLink 的 API，为避免循环引用 proxy 就需要弱引用 displayLink。因为这里当 displayLink 存在 proxy 就存在，displayLink 销毁 proxy 就销毁，所以 proxy 就可以存储 displayLink 的 unsafe_unretained 指针。」
retain|1. MRC 下使用，ARC 下基本使用 strong；<br>2. 修饰强引用，将指针原来指向的旧对象释放掉，然后指向新对象，同时将新对象的引用计数加 1；<br>3. setter 方法的实现是 release 旧值，retain 新值，用于 OC 对象类型。
strong|1. ARC 下才能使用；<br>2. 原理同 retain；<br>3. 但是在修饰 block 时，strong 相当于 copy，而 retain 相当于 assign。
copy|setter 方法的实现是 release 旧值，copy 新值，一般用于 block、NSString、NSArray、NSDictionary 等类型。使用 copy 或 strong 修饰 block 其实都一样，用 copy 是为了和 MRC 下保持一致的写法；用于 NSString、NSArray、NSDictionary 是为了保证赋值后是一个不可变对象，以免遭外部修改而导致不可预期的结果。

### 可空性

见 [iOS 混编｜为 Objective-C API 指定可空性](https://juejin.cn/post/7028567690964893703/)。

### 类属性 class

> Objective-C 类属性是在 [Xcode 8](https://developer.apple.com/library/archive/releasenotes/DeveloperTools/RN-Xcode/Chapters/Introduction.html#//apple_ref/doc/uid/TP40001051-CH1-SW78) 支持的，为了与 Swift 类型属性互操作，它不会进行 `property autosynthesis`。可以说自从 Swift 发布了后，Objective-C 的更新几乎都是为了更好地与 Swift 混编。

属性可以分为实例属性和类属性：

* 实例属性：每个实例都有一套属于自己的属性值，它们之前是相互独立的；
* 类属性：可以为类本身定义属性，无论创建了多少个该类型的实例，这些属性都只有唯一一份，因为类是单例。

说白了就是实例属性与 instance 关联，类属性与 class 关联。

用处：类属性用于定义某个类型所有实例共享的数据，比如所有实例都能用的一个常量/变量（就像 C 语言中的静态常量/静态变量）。

通过给属性添加 class 关键字来定义`类属性`。

```
@property (class, nonatimoc, strong) NSObject *object;
```

类属性是不会进行 `property autosynthesis` 的，那怎么关联值呢？

* 如果是存储属性
    1. 在 .m 中定义一个 static 全局变量，然后在 setter 和 getter 方法中对此变量进行操作。
    2. 在 setter 和 getter 方法中使用关联对象来存储值。笔者之前遇到的一个使用场景就是，类是通过 Runtime 动态创建的，这样就没办法使用 static 全局变量存储值。于是笔者在父类中定义了一个类属性并使用关联对象来存储值，这样动态创建的子类就可以给它的类属性关联值了。
* 如果是计算属性，就直接实现 setter 和 getter 方法就好。

## 其它补充

在设置属性所对应的实例变量时，一定要遵从该属性所声明的语义：

```objectivec
@property (nonatomic, copy) NSString *name;

— (instancetype)initWithName:(NSString *)name {
    if (self = [super init]) {
        _name = [name copy];
    }
    return self;
}
```

若是自己来实现存取方法，也应该保证其具备相关属性所声明的性质。
