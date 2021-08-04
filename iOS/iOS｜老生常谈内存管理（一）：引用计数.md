
## 简单聊聊 GC 与 RC 
随着各个平台的发展，现在被广泛采用的内存管理机制主要有 GC 和 RC 两种。

* GC (Garbage Collection)：垃圾回收机制，定期查找不再使用的对象，释放对象占用的内存。
* RC (Reference Counting)：引用计数机制。采用引用计数来管理对象的内存，当需要持有一个对象时，使它的引用计数 +1；当不需要持有一个对象的时候，使它的引用计数 -1；当一个对象的引用计数为 0，该对象就会被销毁。

Objective-C 程序现基本都已使用 `ARC` 内存管理机制。之前 Mac OS X 平台的 Objective-C 程序可以启用 `GC`，但从 Mac OS X 10.8 开始，`GC` 机制就苹果被废弃了，改用 `ARC` 机制。而 iOS 平台的 Objective-C 程序从未支持过 `GC`，以前使用 `MRC`，iOS 5、OS X Lion 之后 `ARC` 诞生。


## Reference Counting

作为一名 iOS 开发者，引用计数机制是我们必须掌握的知识。那么，引用计数机制下是怎样工作的呢？它存在什么优势？

### 办公室里的照明问题

在《Objective-C 高级编程：iOS 与 OS X 多线程和内存管理》这本书中举了一个 “办公室里的照明问题” 的例子，很好地说明了引用计数机制。

假设办公室里的照明设备只有一个。上班进入办公室的人需要照明，所以要把灯打开。而对于下班离开办公室的人来说，已经不需要照明了，所以要把灯关掉。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11801201a3644e1287182f7aefddc50c~tplv-k3u1fbpfcp-zoom-1.image)

若是很多人上下班，每个人都开灯或者关灯，那么办公室的情况又将如何呢？最早下班的人如果关了灯，那就会像下图那样，办公室里还没走的所有人都将处于一片黑暗之中。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea05a65a217c4ccf83040054e4510c77~tplv-k3u1fbpfcp-zoom-1.image)

解决这一问题的办法就是使办公室在还有至少一人的情况下保持开灯状态，而在无人时保持关灯状态。

（1）最早进入办公室的人开灯。<br>
（2）之后进入办公室的人，需要照明。<br>
（3）下班离开办公室的人，不需要照明。<br>
（4）最后离开办公室的人关灯（此时已无人需要照明）。

为判断是否还有人在办公室里，这里导入计数功能来计算 “需要照明的人数”。下面让我们来看看这一功能是如何运作的吧。

（1）第一个人进入办公室，“需要照明的人数” 加 1。计数值从 0 变成了 1，因此要开灯。<br>
（2）之后每当有人进入办公室，“需要照明的人数” 就加 1。如计数值从 1 变成 2。<br>
（3）每当有人下班离开办公室，“需要照明的人数” 就减 1。如计数值从 2 变成 1。<br>
（4）最后一个人下班离开办公室，“需要照明的人数” 减 1。计数值从 1 变成了 0，因此要关灯。

这样就能在不需要照明的时候保持关灯状态。办公室中仅有的照明设备得到了很好的管理，如下图所示：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9dbf39a1c77a404ca88eecf079ec36ba~tplv-k3u1fbpfcp-zoom-1.image)

在 Objective-C 中，“对象” 相当于办公室里的照明设备。在现实世界中办公室里的照明设备只有一个，但在 Objective-C 的世界里，虽然计算机的资源有限，但一台计算机可以同时处理好几个对象。

此外，“对象的使用环境” 相当于上班进入办公室的人。虽然这里的 “环境” 有时也指在运行中的程序代码、变量、变量作用域、对象等，但在概念上就是使用对象的环境。上班进入办公室的人对办公室照明设备发出的动作，与 Objective-C 中的对应关系则如下表所示：

| 对照明设备所做的动作 | 对 Objective-C 对象所做的动作 |
|:------------|:------------------------|
| 开灯         | 生成对象                   |
| 需要照明       | 持有对象                   |
| 不需要照明      | 释放对象                   |
| 关灯         | 废弃对象                   |

使用计数功能计算需要照明的人数，使办公室的照明得到了很好的管理。同样，使用引用计数功能，对象也就能够得到很好的管理，这就是 Objective-C 的内存管理。如下图所示：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed4162de66a944ce928960cea635970e~tplv-k3u1fbpfcp-zoom-1.image)



## 引用计数的存储

以上我们对 “引用计数” 这一概念做了初步了解，Objective-C 中的 “对象” 通过引用计数功能来管理它的内存生命周期。那么，对象的引用计数是如何存储的呢？它存储在哪个数据结构里？

首先，不得不提一下`isa`。

### isa 
`isa`指针用来维护 “对象” 和 “类” 之间的关系，并确保对象和类能够通过`isa`指针找到对应的方法、实例变量、属性、协议等。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d95cc86303334597ac6ea73f1331dcdc~tplv-k3u1fbpfcp-watermark.image)

在 arm64 架构之前，`isa`就是一个普通的指针，直接指向`objc_class`，存储着`Class`、`Meta-Class`对象的内存地址。`instance`对象的`isa`指向`class`对象，`class`对象的`isa`指向`meta-class`对象；

```objc
// objc.h
struct objc_object {
    Class isa;  // 在 arm64 架构之前
};
```
    
从 arm64 架构开始，对`isa`进行了优化，用`nonpointer`表示，变成了一个共用体（`union`）结构，还使用位域来存储更多的信息。将 64 位的内存数据分开来存储着很多的东西，其中的 33 位才是拿来存储`class`、`meta-class`对象的内存地址信息。要通过位运算将`isa`的值`& ISA_MASK`掩码，才能得到`class`、`meta-class`对象的内存地址。
```objc
// objc-private.h
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

# if __arm64__  // 在 __arm64__ 架构下
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
        uintptr_t has_sidetable_rc  : 1;  // 如果为1，代表引用计数过大无法存储在 isa 中，那么超出的引用计数会存储在一个叫 SideTable 结构体的 RefCountMap（引用计数表）散列表中
        uintptr_t extra_rc          : 19; // 里面存储的值是对象本身之外的引用计数的数量，retainCount - 1
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };
......  // 在 __x86_64__ 架构下
};
```
如果`isa`非`nonpointer`，即 arm64 架构之前的`isa`指针。由于它只是一个普通的指针，存储着`Class`、`Meta-Class`对象的内存地址，所以它本身不能存储引用计数，所以以前对象的引用计数都存储在一个叫`SideTable`结构体的`RefCountMap`（引用计数表）散列表中。

如果`isa`是`nonpointer`，则它本身可以存储一些引用计数。从以上`union isa_t`的定义中我们可以得知，`isa_t`中存储了两个引用计数相关的东西：`extra_rc`和`has_sidetable_rc`。
* extra_rc：里面存储的值是对象本身之外的引用计数的数量，这 19 位如果不够存储，`has_sidetable_rc`的值就会变为 1；
* has_sidetable_rc：如果为 1，代表引用计数过大无法存储在`isa`中，那么超出的引用计数会存储`SideTable`的`RefCountMap`中。

所以，如果`isa`是`nonpointer`，则对象的引用计数存储在它的`isa_t`的`extra_rc`中以及`SideTable`的`RefCountMap`中。

>**备注**：
>* 以上`isa_t`结构来自老版本的`objc4`源码，从`objc4-750`版本开始，`isa_t`中的`struct`的内容定义成了宏并写在`isa.h`文件里，不过其数据结构不变，这里不影响。
>* 更多关于`isa`的知识，以及以上提到的一些细节，可以查看[《深入浅出 Runtime（二）：数据结构》](https://juejin.im/post/6844904072215003143)。


### SideTable
以上提到了一个数据结构`SideTable`，我们进入`objc4`源码查看它的定义。
```objc
// NSObject.mm
struct SideTable {
    spinlock_t slock;        // 自旋锁
    RefcountMap refcnts;     // 引用计数表（散列表）
    weak_table_t weak_table; // 弱引用表（散列表）
    ......
}
```
`SideTable`存储在`SideTables()`中，`SideTables()`本质也是一个散列表，可以通过对象指针来获取它对应的（引用计数表或者弱引用表）在哪一个`SideTable`中。在非嵌入式系统下，`SideTables()`中有 64 个`SideTable`。以下是`SideTables()`的定义：
```objc
// NSObject.mm
static objc::ExplicitInit<StripedMap<SideTable>> SideTablesMap;

static StripedMap<SideTable>& SideTables() {
    return SideTablesMap.get();
}
```
所以，查找对象的引用计数表需要经过两次哈希查找：

* ① 第一次根据当前对象的内存地址，经过哈希查找从`SideTables()`中取出它所在的`SideTable`；
* ② 第二次根据当前对象的内存地址，经过哈希查找从`SideTable`中的`refcnts`中取出它的引用计数表。


>**Q：为什么不是一个`SideTable`，而是使用多个`SideTable`组成`SideTables()`结构？**

如果只有一个`SideTable`，那我们在内存中分配的所有对象的引用计数或者弱引用都放在这个`SideTable`中，那我们对对象的引用计数进行操作时，为了多线程安全就要加锁，就存在效率问题。<br>
系统为了解决这个问题，就引入 “分离锁” 技术方案，提高访问效率。把对象的引用计数表分拆多个部分，对每个部分分别加锁，那么当所属不同部分的对象进行引用操作的时候，在多线程下就可以并发操作。所以，使用多个`SideTable`组成`SideTables()`结构。

>**备注：** 关于引用计数具体是怎么管理的，可以参阅[《iOS - 老生常谈内存管理（四）：内存管理方法源码分析》](https://juejin.cn/post/6844904131719593998)。

  