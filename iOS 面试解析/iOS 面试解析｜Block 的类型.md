
本期 [「iOS 摸鱼周报 第二十二期」](https://mp.weixin.qq.com/s/JI5mlzX9cYhXJS81k1WE6A) 面试解析模块讲解的知识点是 `Block 类型`。你是否遇到过这样的面试题：
* Block 都有什么类型？
* 栈 Block 存在什么问题？
* Block 每种类型调用 copy 的结果分别是怎样的？

希望以下的总结能帮助到你。如果你对内容有任何疑问，或者有更好的解答，都可以联系我们。

## Block 类型

Block 有 3 种类型：栈块、堆块、全局块。

Block 类型|描述|环境
:--:|:--:|:--:
`__NSGlobalBlock__`<br>（ _NSConcreteGlobalBlock ）|全局 Block，保存在数据段区（.data 区）|定义在全局区，或者没有访问自动局部变量
`__NSStackBlock__`<br>（ _NSConcreteStackBlock ）|栈 Block，保存在栈区|访问了自动局部变量
`__NSMallocBlock__`<br>（ _NSConcreteMallocBlock ）|堆 Block，保存在堆区|`__NSStackBlock__` 调用了 copy

它们最终都是继承自 NSBlock 类型，NSBlock 又继承自 NSObject。


### 1. 栈块

定义块的时候，其所占的内存区域是分配在栈中的。块只在定义它的那个范围内有效。

```objectivec
void (^block)(void);
if ( /* some condition */ ) {
    block = ^{
        NSLog(@"Block A");
    };
} else {
    block = ^{
        NSLog(@"Block B");
    };
}
block();
```

上面的代码有危险，定义在 if 及 else 中的两个块都分配在栈内存中，当出了 if 及 else 的范围，栈块可能就会被销毁。如果编译器覆写了该块的内存，那么调用该块就会导致程序崩溃。或者数据可能会变成垃圾数据，尽管将来该块还能正常调用，但是它捕获的变量的值已经错乱了。

>若是在 ARC 下，上面 block 会被自动 copy 到堆，所以不会有问题。但在 MRC 下我们要避免这样写。

### 2. 堆块

为了解决以上问题，可以给块对象发送 copy 消息将其从栈拷贝到堆区，堆块可以在定义它的那个范围之外使用。堆块是带引用计数的对象，所以在 MRC 下如果不再使用堆块需要调用 release 进行释放。

```objectivec
void (^block)(void);
if ( /* some condition */ ) {
    block = [^{
        NSLog(@"Block A");
    } copy];
} else {
    block = [^{
        NSLog(@"Block B");
    } copy];
}
block();
[block release];
```

你可能会问，我平常在代码中也没有对 Block 进行 copy 呀？

其实，在 ARC 下，编译器会根据情况自动生成将栈上的 Block 复制到堆上的代码，比如以下几种情况：

* 手动调用 Block 的 copy 方法时；
* Block 作为函数返回值时；
* 将 Block 赋值给 __strong 指针时；
* Block 作为 Cocoa API 中方法名含有 usingBlock 的方法参数时；
* Block 作为 GCD API 的方法参数时。


### 3. 全局块

如果运行块所需的全部信息都能在编译期确定，包括没有访问自动局部变量等，那么该块就是全局块。全局块可以声明在全局内存中，而不需要在每次用到的时候于栈中创建。全局块的 copy 操作是空操作，因为全局块决不可能被系统所回收，其实际上相当于单例。

因为没有访问自动局部变量，所以 Block 不依赖于执行时的状态（这里指的是不依赖自动局部变量的变化），所以整个程序只需一个实例即可，因此它保存在数据段区即可。

```objectivec
void (^block_0)(void) = ^{ NSLog(@"This is a block"); };

int main(int argc, const char * argv[]) {
    @autoreleasepool {

        void (^block_1)(void) = ^{ NSLog(@"This is a block"); };

        NSLog(@"%@", [block_0 class]);
        NSLog(@"%@", [block_1 class]);
    }
    return 0;
}

// __NSGlobalBlock__
// __NSGlobalBlock__
```

> 上面的 block_1 你通过 clang 转换成的 C++ 代码是 `_NSConcreteStackBlock` 类型，但打印 class 类型却是 `__NSGlobalBlock__`，以后者为准，这个问题《Objective-C 高级编程：iOS 与 OS X 多线程和内存管理》书中也提到了。

## 每一种类型的 Block 调用 copy 后的结果如下所示：

Block 类型|副本源的配置存储区|复制效果
:--|:--|:--
_NSConcreteGlobalBlock|程序的数据段区（.data 区）|什么也不做
_NSConcreteStackBlock|栈|从栈复制到堆
_NSConcreteMallocBlock|堆|引用计数增加
