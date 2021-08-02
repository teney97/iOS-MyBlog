## Runtime 系列文章
[深入浅出 Runtime（一）：初识](https://juejin.im/post/6844904071480999949)<br>
[深入浅出 Runtime（二）：数据结构](https://juejin.im/post/6844904072215003143)<br>
[深入浅出 Runtime（三）：消息机制](https://juejin.im/post/6844904072235974663)<br>
[深入浅出 Runtime（四）：super 的本质](https://juejin.im/post/6844904072252751880)<br>
[深入浅出 Runtime（五）：相关面试题](https://juejin.im/post/6844904072428912653)

![](https://user-gold-cdn.xitu.io/2020/2/28/17087c4d8e1b438a?w=1600&h=900&f=jpeg&s=357120)


## 1. objc_super 与 objc_msgSendSuper
我们先来看两个数据结构`objc_super`和`objc_super2`。

它们的区别在于第二个成员：
* `objc_super`：super_class  // receiverClass 的父类
* `objc_super2`：current_class  // receiverClass（消息接收者的class对象）

```objc
// message.h（objc4）
struct objc_super {
    __unsafe_unretained _Nonnull id receiver;  // 消息接收者
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;  // receiverClass 的父类
#endif
    /* super_class is the first class to search */
};

// objc_runtime_new.h（objc4）
struct objc_super2 {
    id receiver;  // 消息接收者
    Class current_class;  // receiverClass（消息接收者的class对象）
};
```
再来看两个函数`objc_msgSendSuper()`和`objc_msgSendSuper2()`。

从源码来看，两个函数所接收的参数没有区别。

但是从官方注释我们可以推测，`objc_msgSendSuper2()` 函数所接收的第一个参数应该为`objc_super2`而非`objc_super`。
```objc
// message.h（objc4）
void objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )

// objc-abi.h（objc4）
// objc_msgSendSuper2() takes the current search class, not its superclass.
id _Nullable objc_msgSendSuper2(struct objc_super * _Nonnull super, SEL _Nonnull op, ...)
```

## 2. self 和 super

### self
* OC 方法都带有两个隐式参数：`(id)self`和`(SEL)_cmd`；
* self 是一个对象指针，指向当前方法的调用者/消息接收者；
    * 如果是实例方法，它就是指向当前类的实例对象；
    * 如果是类方法，它就是指向当前类的类对象。
* 当使用 self 调用方法的时候，底层会转换为`objc_msgSend()`函数的调用，通过上一篇文章可以知道，该函数会从`当前消息接收者类`中开始查找方法的实现。

### super
* super 是一个编译器指令；
* 当使用 super 调用方法的时候，底层会转换为`objc_msgSendSuper2()`函数的调用，该函数会从`当前消息接受者类的父类`中开始查找方法的实现。

## 3. super 本质
我们通过 clang 将以下 OC 代码 转换为 C++ 代码：
```objc
    [super viewDidLoad];
```
```objc
    // 转换为 C++
    ((void (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("ViewController"))}, sel_registerName("viewDidLoad"));
    // 简化
    struct objc_super arg = {
        self,
        class_getSuperclass(objc_getClass("ViewController"))
    };
    objc_msgSendSuper(arg, sel_registerName("viewDidLoad"));
```
可以看到，Runtime 将`super`转换为`objc_msgSendSuper()`函数的调用，参数为`objc_super`和`SEL`。

### LLVM & 中间代码
那么为什么前面说`super`会转换为`objc_msgSendSuper2()`函数的调用呢？

因为转成的 C++ 的实现和真正的底层实现是有差异的，

`LLVM`编译器会将 `“ OC 代码”` 先转成 `“中间代码(.ll)”` 再转成 `“汇编、机器代码”`，该中间代码非 C/C++。

可以使用以下命令行指令生成中间代码：`clang -emit-llvm -S main.m`
<br>具体可以查看官方文档 [LLVM](https://llvm.org/docs/LangRef.html)，这里不做过多介绍。

### 通过汇编验证
将 ViewController.m 文件转换成汇编代码进行验证：

![](https://user-gold-cdn.xitu.io/2020/2/25/1707c482ab2e2f52?w=1118&h=952&f=png&s=799410)

查看第 18 行代码即`[super viewDidLoad]`转换成的汇编代码

![](https://user-gold-cdn.xitu.io/2020/2/25/1707c482abe8bae2?w=1240&h=847&f=png&s=291562)

以上可以看到，`[super viewDidLoad]`底层实际上是转换成了`objc_msgSendSuper2()`函数的调用而非`objc_msgSendSuper()`。

### super 本质

当使用 super 调用方法的时候，底层会转换为`objc_msgSendSuper2()`函数的调用，该函数接收两个参数`struct objc_super2 `和`SEL`。

```objc
struct objc_super2 {
    id receiver;  // 消息接收者
    Class current_class;  // receiverClass
};
id _Nullable objc_msgSendSuper2(struct objc_super * _Nonnull super, SEL _Nonnull op, ...)
```

`objc_msgSendSuper2()`函数内部会通过`current_class`的`superclass`指针拿到它的父类，从父类开始查找方法的实现。忽略“从 receiverClass 中查找方法的过程”，对应下图就是直接从第 5 步开始。

![objc_msgSendSuper2()执行流程](https://user-gold-cdn.xitu.io/2020/2/25/1707c482abe50bdf?w=1240&h=683&f=png&s=457596)

要注意`receiver`消息接收者还是子类对象，而不是父类对象，只是查找方法实现的范围变了。

## 4. 相关面试题
#### Q：调用以下 init 方法的打印结果是什么？
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
