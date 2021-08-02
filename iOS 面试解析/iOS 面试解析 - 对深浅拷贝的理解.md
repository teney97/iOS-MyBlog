
本期 [「iOS 摸鱼周报 第二十期」](https://mp.weixin.qq.com/s/PjiZzx3VSAfAGHRJs160aQ) 面试解析模块讲解的知识点是 `对深浅拷贝的理解`。文章将从深拷贝和浅拷贝的区别开始讲起，然后讲解在 iOS 中对 mutable 对象与 immutable 对象进行 copy 与 mutableCopy 的结果，以及如何对集合对象进行真正意义上的深拷贝，最后带你实现对自定义对象的深浅拷贝。

## 对深浅拷贝的理解

我们先要理解拷贝的目的：产生一个副本对象，跟源对象互不影响。

#### 深拷贝和浅拷贝的区别

拷贝类型|拷贝方式|特点
--|--|--
深拷贝|内存拷贝，让副本对象指针和源对象指针指向 `两片` 内容相同的内存空间。|1. 不会增加被拷贝对象的引用计数；<br>2. 产生了一个内存分配，出现了两块内存。
浅拷贝|指针拷贝，对内存地址的复制，让副本对象指针和源对象指针指向 `同一片` 内存空间。|1. 会增加被拷贝对象的引用计数；<br>2. 没有进行新的内存分配。<br>注意：如果是小对象如 NSString，可能通过 Tagged Pointer 来存储，没有引用计数。

简而言之：

* 深拷贝：内存拷贝，产生新对象，不增加被拷贝对象引用计数
* 浅拷贝：指针拷贝，不产生新对象，增加被拷贝对象引用计数，相当于执行了 retain
* 区别：1. 是否影响了引用计数；2. 是否开辟了新的内存空间

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fee5bbb3c81d45b5b9d2e5c024d7948c~tplv-k3u1fbpfcp-zoom-1.image)

#### 在 iOS 中对 mutable 对象与 immutable 对象进行 copy 与 mutableCopy 的结果

iOS 提供了 2 个拷贝方法：

* copy：不可变拷贝，产生不可变副本
* mutableCopy：可变拷贝，产生可变副本

对 mutable 对象与 immutable 对象进行 copy 与 mutableCopy 的结果：

源对象类型|拷贝方式|副本对象类型|拷贝类型（深/浅）
:--:|:--:|:--:|:--:
mutable 对象|copy|不可变|深拷贝
mutable 对象|mutableCopy|可变|深拷贝
immutable 对象|copy|不可变|`浅拷贝`
immutable 对象|mutableCopy|可变|深拷贝

>注：这里的 immutable 对象与 mutable 对象指的是系统类 NSArray、NSDictionary、NSSet、NSString、NSData 与它们的可变版本如 NSMutableArray 等。

一个记忆技巧就是：对 immutable 对象进行 copy 操作是 `浅拷贝`，其它情况都是 `深拷贝`。

我们还可以根据拷贝的目的加深理解：

* 对 immutable 对象进行 copy 操作，产生 immutable 对象，因为源对象和副本对象都不可变，所以进行指针拷贝即可，节省内存
* 对 immutable 对象进行 mutableCopy 操作，产生 mutable 对象，对象类型不同，所以需要深拷贝
* 对 mutable 对象进行 copy 操作，产生 immutable 对象，对象类型不同，所以需要深拷贝
* 对 mutable 对象进行 mutableCopy 操作，产生 mutable 对象，为达到修改源对象或副本对象互不影响的目的，需要深拷贝

#### 使用 copy、mutableCopy 对集合对象进行的深浅拷贝是针对集合对象本身的

使用 copy、mutableCopy 对集合对象（Array、Dictionary、Set）进行的深浅拷贝是针对集合对象本身的，对集合中的对象执行的默认都是浅拷贝。也就是说只拷贝集合对象本身，而不复制其中的数据。主要原因是，集合内的对象未必都能拷贝，而且调用者也未必想在拷贝集合时一并拷贝其中的每个对象。

如果想要深拷贝集合对象本身的同时，也对集合内容进行 copy 操作，可使用类似以下的方法，copyItems 传 YES。集合中的每个对象都会收到 copyWithZone: 消息，所以需要注意的是集合中的对象必须都符合 NSCopying 协议，否则会导致 Crash。

```objectivec
NSArray *deepCopyArray = [[NSArray alloc]initWithArray:someArray copyItems:YES];
```

>注：`initWithArray:copyItems:` 方法不是所有情况下都深拷贝集合对象本身的。如果执行 `[[NSArray alloc]initWithArray:@[] copyItems:aBoolValue];`，也就是源对象为不可变的空数组的话，对源对象本身执行的是浅拷贝，苹果对 `@[]` 使用了享元。

但是，如果集合中的对象的 copy 操作是浅拷贝，那么对于集合来说还不是真正意义上的深拷贝。比如，你需要对一个 `NSArray<NSArray *>` 对象进行真正的深拷贝，那么内层数组及其内容也应该执行深拷贝，可以对该集合对象进行 `归档` 然后 `解档`，只要集合中的对象都符合 NSCoding 协议。而且，使用这种方式，无论集合中存储的模型对象嵌套多少层，都可以实现深拷贝，但前提是嵌套的子模型也需要符合 NSCoding 协议才行，否则会导致 Crash。

```objectivec
NSArray *trueDeepCopyArray = [NSKeyedUnarchiver unarchiveObjectWithData:[NSKeyedArchiver archivedDataWithRootObject:oldArray]];
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af7287980a194872ac170de124c9f1b8~tplv-k3u1fbpfcp-zoom-1.image)


>需要注意的是，使用 [initWithArray:copyItems:](https://developer.apple.com/documentation/foundation/nsarray/1408557-initwitharray/) 并将 copyItems 传 YES 时，生成的副本集合对象中的对象（下一个级别）是不可变的，所有更深的级别都具有它们以前的可变性。比如以下代码将 Crash。
>
>```objectivec
>NSArray *oldArray = @[@[].mutableCopy];
>NSArray *deepCopyArray = [[NSArray alloc] initWithArray:oldArray copyItems:YES];
>NSMutableArray *mArray = deepCopyArray[0]; // deepCopyArray[0] 已经被深拷贝为 NSArray 对象
>[mArray addObject:@""]; // Crash
>```
>而 `归档解档集合` 的方式会保留所有级别的可变性，就像以前一样。

#### 实现对自定义对象的拷贝

如果想要实现对自定义对象的拷贝，需要遵守 `NSCopying` 协议，并实现 `copyWithZone:` 方法。

* 如果要浅拷贝，`copyWithZone:` 方法就返回当前对象：return self；
* 如果要深拷贝，`copyWithZone:` 方法中就创建新对象，并给希望拷贝的属性赋值，然后将其返回。如果有嵌套的子模型也需要深拷贝，那么子模型也需符合 NSCopying 协议，且在属性赋值时调用子模型的 copy 方法，以此类推。

如果自定义对象支持可变拷贝和不可变拷贝，那么还需要遵守 `NSMutableCopying` 协议，并实现 `mutableCopyWithZone:` 方法，返回可变副本。而 `copyWithZone:` 方法返回不可变副本。使用方可根据需要调用该对象的 copy 或 mutableCopy 方法来进行不可变拷贝或可变拷贝。

#### 以下代码会出现什么问题？

```objectivec
@interface Model : NSObject
@property (nonatomic, copy) NSMutableArray *array;
@end
```

不论赋值过来的是 NSMutableArray 还是 NSArray 对象，进行 copy 操作后都是 NSArray 对象（深拷贝）。由于属性被声明为 NSMutableArray 类型，就不可避免的会有调用方去调用它的添加对象、移除对象等一些方法，此时由于 copy 的结果是 NSArray 对象，所以就会导致 Crash。


## 相关资料

* [Apple Documentation Archive - Collections Programming Topics - Copying Collections](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Collections/Articles/Copying.html#//apple_ref/doc/uid/TP40010162-SW1)
* [Apple Documentation - initWithArray:copyItems:](https://developer.apple.com/documentation/foundation/nsarray/1408557-initwitharray/)


