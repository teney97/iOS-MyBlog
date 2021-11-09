![网络配图](https://user-gold-cdn.xitu.io/2020/3/7/170b0ef52e70577d?w=1240&h=698&f=jpeg&s=106102)


## 1. 属性关键字有哪些？
分类|属性关键字
--|--
原子性|atomic、nonatomic
读写权限|readwrite、readonly、setter、getter
内存管理|assign、weak、unsafe_unretained、retain、strong、copy
可空性|(nullable、_Nullable 、__nullable)、<br>(nonnull、_Nonnull、__nonnull)、<br>(null_unspecified、_Null_unspecified 、__null_unspecified)、<br>null_resettable

### 1.1 原子性

属性关键字|用法
-- |--
atomic|原子性（默认），编译器会自动生成互斥锁，对 setter 和 getter 方法进行加锁，可以保证属性的赋值和取值的原子性操作是线程安全的，但不包括操作和访问。<br>比如说 atomic 修饰的是一个数组的话，那么我们对数组进行赋值和取值是可以保证线程安全的。但是如果我们对数组进行操作，比如说给数组添加对象或者移除对象，是不在 atomic 的负责范围之内的，所以给被 atomic 修饰的数组添加对象或者移除对象是没办法保证线程安全的。
nonatomic|非原子性，一般属性都用 nonatomic 进行修饰，因为 atomic 非常耗时。

### 1.2 读写权限
属性关键字|用法
--|--
readwrite|可读可写（默认），同时生成 setter 方法和 getter 方法的声明和实现。
readonly|只读，只生成 getter 方法的声明和实现。
setter|可以指定生成的 setter 方法名，如 setter = setName。
getter|可以指定生成的 getter 方法名，如 getter = getName。

### 1.3 内存管理
属性关键字|用法
--|--
assign|1. 既可以修饰基本数据类型，也可以修饰对象类型；<br>2. setter 方法的实现是直接赋值，一般用于基本数据类型 ；<br>3. 修饰基本数据类型，如 NSInteger、BOOL、int、float 等；<br>4. 修饰对象类型时，不增加其引用计数；<br>5. 会产生悬垂指针（悬垂指针：assign 修饰的对象在被释放之后，指针仍然指向原对象地址，该指针变为悬垂指针。这时候如果继续通过该指针访问原对象的话，就可能导致程序崩溃）。
weak|1. 只能修饰对象类型；<br>2. ARC 下才能使用；<br>3. 修饰弱引用，不增加对象引用计数，主要可以用于避免循环引用；<br>4. weak 修饰的对象在被释放之后，会自动将指针置为 nil，不会产生悬垂指针。
unsafe_unretained|1. 既可以修饰基本数据类型，也可以修饰对象类型；<br>2. MRC 下经常使用，ARC 下基本不用；<br>3. 同 weak，区别就在于 unsafe_unretained 会产生悬垂指针。
retain|1. MRC 下使用，ARC 下基本使用 strong；<br>2. 修饰强引用，将指针原来指向的旧对象释放掉，然后指向新对象，同时将新对象的引用计数加1；<br>3. setter 方法的实现是 release 旧值，retain 新值，用于OC对象类型。
strong|1. ARC 下才能使用；<br>2. 原理同 retain；<br>3. 但是在修饰 block 时，strong 相当于 copy，而 retain 相当于 assign。
copy|setter 方法的实现是 release 旧值，copy 新值，用于 NSString、block 等类型。

### 1.4 可空性
见 [iOS 混编｜为 Objective-C API 指定可空性](https://juejin.cn/post/7028567690964893703/)。


## 2.所有权修饰符
所有权修饰符|用法
--|--
__strong|1. 强引用持有对象，可以对应 strong、retain、copy 关键字。<br>2. 编译器将为 strong、retain、copy 修饰的属性生成带 __strong 所有权修饰符的实例变量。
__weak|1. 弱引用持有对象，对应 weak 关键字，ARC下用来防止循环引用。<br>2. 编译器将为 weak 修饰的属性生成带 __weak 所有权修饰符的实例变量。
__unsafe_unretained|1. 弱引用持有对象，对应 unsafe_unretained、assign 关键字，MRC下用来防止循环引用。<br>2. 编译器将为 unsafe_unretained 修饰的属性生成带 __unsafe_unretained 所有权修饰符的实例变量。<br>3. 与 __weak 相比，它不需要遍历 weak 表来检查对象是否 nil，性能上要更好一些。但是它会产生悬垂指针。
__autoreleasing|在 MRC 中我们可以给对象发送 autorelease 消息来将它注册到 autoreleasepool 中，而在 ARC 中我们可以使用 __autoreleasing 修饰符修饰对象将对象注册到 autoreleasepool 中。

关于所有权修饰符的详细解释，可以参阅[《iOS - 老生常谈内存管理（三）：ARC 面世》](https://juejin.im/post/6844904130431942670)。

## 3.相关面试题

#### Q：atomic 修饰的属性是怎么样保存线程安全的？
**答：** 编译器会自动生成互斥锁，对 setter 和 getter 方法进行加锁，可以保证属性的赋值和取值原子性操作是线程安全的，但不包括操作和访问。<br>比如说`atomic`修饰的是一个数组的话，那么我们对数组进行赋值和取值是可以保证线程安全的。但是如果我们对数组进行操作，比如说给数组添加对象或者移除对象，是不在`atomic`的负责范围之内的，所以给被`atomic`修饰的数组添加对象或者移除对象是没办法保证线程安全的。


#### Q：什么时候使用 weak/__weak 关键字？
* ① ARC 中为了避免循环引用而使用，可以让相互引用的对象中的一个使用`weak/__weak`弱引用修饰，常用于对`delegate`和`block`的修饰；
* ② Interface Builder 中 IBOutlet 修饰的控件一般也是用`weak`。

#### Q：assign 和 weak 关键字的区别有哪些？
* ① `weak`只能修饰对象，而`assign`既可以修饰对象也可以修饰基本数据类型；
* ② `assign`修饰的对象在被释放后，指针仍然指向原对象地址；而`weak`修饰的对象在被释放之后会自动置指针为 nil；
* ③ 相同点：在修饰对象的时候，`assign`和`weak`都不改变对象的引用计数。


#### Q：以下代码会出现什么问题？（深浅拷贝）
```objc
@property (copy) NSMutableArray *array;
```
**答：** 不论赋值过来的是`NSMutableArray`还是`NSArray`对象，进行`copy`操作后都是`NSArray`对象（深拷贝）。由于属性被声明为`NSMutableArray`类型，就不可避免的会有调用方去调用它的添加对象、移除对象等一些方法，此时由于`copy`的结果是`NSArray`不可变对象，对`NSArray`对象调用添加对象、移除对象等方法，就会产生程序异常。


## 参考
[Nullability and Objective-C（Apple Blog）](https://developer.apple.com/swift/blog/?id=25)

