## Runtime 系列文章
[深入浅出 Runtime（一）：初识](https://juejin.im/post/6844904071480999949)<br>
[深入浅出 Runtime（二）：数据结构](https://juejin.im/post/6844904072215003143)<br>
[深入浅出 Runtime（三）：消息机制](https://juejin.im/post/6844904072235974663)<br>
[深入浅出 Runtime（四）：super 的本质](https://juejin.im/post/6844904072252751880)<br>
[深入浅出 Runtime（五）：相关面试题](https://juejin.im/post/6844904072428912653)

![](https://user-gold-cdn.xitu.io/2020/2/28/17087c4d8e1b438a?w=1600&h=900&f=jpeg&s=357120)

#### Q：你了解 isa 指针吗？
* `isa`指针用来维护对象和类之间的关系，并确保对象和类能够通过`isa`指针找到对应的方法、实例变量、属性、协议等；
* 在 arm64 架构之前，`isa`就是一个普通的指针，直接指向`objc_class`，存储着`Class`、`Meta-Class`对象的内存地址。`instance`对象的`isa`指向`class`对象，`class`对象的`isa`指向`meta-class`对象；
* 从 arm64 架构开始，对`isa`进行了优化，变成了一个共用体（`union`）结构，还使用位域来存储更多的信息。将 64 位的内存数据分开来存储着很多的东西，其中的 33 位才是拿来存储`class`、`meta-class`对象的内存地址信息。要通过位运算将`isa`的值`& ISA_MASK`掩码，才能得到`class`、`meta-class`对象的内存地址；
* `isa`指针存储的信息；
* `isa`指针的指向。<br>
[传送门：深入浅出 Runtime（二）：数据结构](https://juejin.im/post/6844904072215003143)

#### Q：类对象与元类对象的区别和联系。
* `class`、`meta-class`底层结构都是`objc_class`结构体，`objc_class`继承自`objc_object`，所以它也有`isa`指针，它也是对象；
* `class`中存储着实例方法、成员变量、属性、协议等信息，
* `meta-class`中存储着类方法等信息；
* `isa`指针和`superclass`指针的指向；
* 基类的`meta-class`的`superclass`指向基类的`class`，决定了一个性质：<br>当我们调用一个类方法，会通过`class`的`isa`指针找到`meta-class`，在`meta-class`中查找有无该类方法，如果没有，再通过`meta-class`的`superclass`指针逐级查找父`meta-class`，一直找到基类的`meta-class`如果还没找到该类方法的话，就会去找基类的`class`中同名的实例方法的实现。

![isa 与 superclass 指针指向](https://user-gold-cdn.xitu.io/2020/2/25/1707be7d4709e537?w=1041&h=664&f=png&s=282120)

#### Q：为什么要设计 meta-class ？

目的是将实例和类的相关方法列表以及构建信息区分开来，方便各司其职，符合单一职责设计原则。

#### Q：Runtime 的消息机制，objc_msgSend 方法调用流程。

[传送门：深入浅出 Runtime（三）：消息机制](https://juejin.im/post/6844904072235974663)

`OC`中的方法调用，其实都是转换为`objc_msgSend()`函数的调用（不包括`[super message]`）。`objc_msgSend()`的执行流程可以分为 3 大阶段：消息发送、动态方法解析、消息转发。

![](https://user-gold-cdn.xitu.io/2020/2/26/1707d62ff681d0e5?w=1240&h=1059&f=png&s=304345)

#### Q：调用以下 init 方法的打印结果是什么？（super）
```objc
@interface HTPerson : NSObject
@end

@interface HTStudent : HTPerson
@end

@implementation HTStudent
- (instancetype)init
{
    if (self = [super init]) {
        
        NSLog(@"[self class] = %@",[self class]);
        NSLog(@"[super class] = %@",[super class]);
        NSLog(@"[self superclass] = %@",[self superclass]);
        NSLog(@"[super superclass] = %@",[super superclass]);
        
    }
    return self;
}
@end
```
> [self class] = HTStudent<br>
> [super class] = HTStudent<br>
> [self superclass] = HTPerson<br>
> [super superclass] = HTPerson

`class`和`superclass`方法的实现在 NSObject 类中，可以看到它们的返回值取决于`receiver`。

```objc
+ (Class)class {
    return self;
}
- (Class)class {
    return object_getClass(self);
}
+ (Class)superclass {
    return self->superclass;
}
- (Class)superclass {
    return [self class]->superclass;
}
```
`[self class]`是从`receiverClass`开始查找方法的实现，如果没有重写的情况，则会一直找到基类 NSObject，然后调用。

`[super class]`是从`receiverClass->superclass`开始查找方法的实现，如果没有重写的情况，则会一直找到基类 NSObject，然后调用。

由于`receiver`相同，所以它们的返回值是一样的。

#### Q：如何防止“调用无法识别的方法导致应用程序崩溃”？
重写`doseNotRecognizeSelector`方法。

#### Q：@synthesize 和 @dynamic

* `@synthesize` ：为属性生成下划线成员变量，并且自动生成`setter`和`getter`方法的实现。以前 Xcode 还没这么智能的时候就要这么做。而现在默认我们写的属性，会自动进行`@synthesize`。有时候我们不希望它自动生成，而是在程序运行过程中再去决定该方法的实现，就可以使用`@dynamic`。
* `@dynamic` ：是告诉编译器不用自动生成`setter`和`getter`的实现，不用自动生成成员变量，等到运行时再添加方法实现，但是它不会影响`setter`和`getter`方法的声明。
* 动态运行时语言与编译时语言的区别：动态运行时语言将函数决议推迟到运行时，编译时语言在编译器进行函数决议。OC 是动态运行时语言。

#### Q：能否向编译后的类增加实例变量？能否向运行时动态创建的类增加实例变量？

* 不能向编译后的类增加实例变量。类的内存布局在编译时就已经确定，类的实例变量列表存储在`class_ro_t`结构体里，编译时就确定了内存大小无法修改，所以不能向编译后的类增加实例变量。
* 能向运行时动态创建的类增加实例变量。运行时动态创建的类只是通过`alloc`分配了类的内存空间，没有对类进行内存布局，内存布局是在类初始化过程中完成的，所以能向运行时动态创建的类增加实例变量。<br>

需要注意的是，要在调用`注册类`的方法之前去完成实例变量的添加，因为注册类的时候，类的结构就生成了。说白了就是`class_addIvar()`函数不能给已经存在的类动态添加成员变量。

```objc
    // 动态创建一对类和元类（参数：父类，类名，额外的内存空间）
    Class newClass = objc_allocateClassPair([NSObject class], "Person", 0);
    // 动态添加成员变量
    class_addIvar(newClass, "_age", 4, 1, @encode(int));
    class_addIvar(newClass, "_name", sizeof(NSString *), log2(sizeof(NSString *)), @encode(NSString *));
    // 注册一对类和元类（要在类注册之前添加成员变量）
    objc_registerClassPair(newClass);
    // 创建实例
    id person = [[newClass alloc] init];
    [person setValue:@"Lucy" forKey:@"name"];
    [person setValue:@"20" forKey:@"age"];  
    NSLog(@"name:%@, age:%@", [person valueForKey:@"name"], [person valueForKey:@"age"]);    
    // 当类和它的子类的实例存在时，不能调用 objc_disposeClassPair()，否则会 Crash：Attempt to use unknown class 0x1005af5c0.
    person = nil;    
    // 销毁一对类和元类
    objc_disposeClassPair(newClass);

    // name:Lucy, age:20
```

#### Q：你是否有使用过 performSelector: 方法？
使用场景：一个类在编译时没有这个方法，在运行的时候才产生了这个方法，这个时候要调用这个方法就要用到`performSelector:`方法。<br>
关于动态添加方法的实现可以查看：[传送门：深入浅出 Runtime（三）：消息机制](https://juejin.im/post/6844904072235974663)


#### Q：以下打印结果是什么？（isKindOfClass & isMemberOfClass）
```objc
@interface Person : NSObject
@end
......
    BOOL res1 = [[NSObject class] isKindOfClass:[NSObject class]];
    BOOL res2 = [[NSObject class] isMemberOfClass:[NSObject class]];
    BOOL res3 = [[Person class] isKindOfClass:[Person class]];
    BOOL res4 = [[Person class] isMemberOfClass:[Person class]];

    NSLog(@"%d,%d,%d,%d", res1, res2, res3, res4);
......
```
打印结果：1,0,0,0<br>
以下是`isMemberOfClass`和`isKindOfClass`方法以及`object_getClass()`函数的实现。

```objc
+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
```

* `isMemberOfClass`方法是判断当前`instance/class`对象的`isa`指向是不是`class/meta-class`对象类型；
* `isKindOfClass`方法是判断当前`instance/class`对象的`isa`指向是不是`class/meta-class`对象或者它的子类类型。

显然`isKindOfClass`的范围更大。如果方法调用着是`instance`对象，传参就应该是`class`对象。如果方法调用着是`class`对象，传参就应该是`meta-class`对象。所以`res2`-`res4`都为 0。那为什么`res1`为 1 呢？

因为 NSObject 的`class`的对象的`isa`指向它的`meta-class`对象，而它的`meta-class`的`superclass`指向它的`class`对象，所以它满足`isKindOfClass`方法的判断条件。

总之，`[instance/class isKindOfClass:[NSObject class]];`都返回 1。
