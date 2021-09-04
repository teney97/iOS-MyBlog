## iOS 面试解析｜KVO 的实现原理

Apple 使用了 isa-swizzling 方案来实现 KVO。

**注册：**

当我们调用 `addObserver:forKeyPath:options:context:` 方法，为 **被观察对象** a 添加 KVO 监听时，系统会在运行时动态创建 a 对象所属类 A 的子类 `NSKVONotifying_A`，并且让 a 对象的 isa 指向这个子类，同时重写父类 A 的 **被观察属性** 的 setter 方法来达到可以通知所有 **观察者对象** 的目的。

这个子类的 isa 指针指向它自己的 meta-class 对象，而不是原类的 meta-class 对象。

重写的 setter 方法的 SEL 对应的 IMP 为 Foundation 中的 `_NSSetXXXValueAndNotify` 函数（XXX 为 Key 的数据类型）。因此，当 **被观察对象** 的属性发生改变时，会调用 _NSSetXXXValueAndNotify 函数，这个函数中会调用：

* `willChangeValueForKey:` 方法
* 父类 A 的 setter 方法
* `didChangeValueForKey:` 方法

**监听：**

而 willChangeValueForKey: 和 didChangeValueForKey: 方法内部会触发 **观察者对象** 的监听方法：`observeValueForKeyPath:ofObject:change:context:`，以此完成 KVO 的监听。

willChangeValueForKey: 和 didChangeValueForKey: 触发监听方法的时机：

* didChangeValueForKey: 方法会直接触发监听方法
* `NSKeyValueObservingOptionPrior` 是分别在值改变前后触发监听方法，即一次修改有两次触发。而这两次触发分别在 willChangeValueForKey: 和 didChangeValueForKey: 的时候进行的。如果注册方法中 options 传入 NSKeyValueObservingOptionPrior，那么可以通过只调用 willChangeValueForKey: 来触发改变前的那次 KVO，可以用于在属性值即将更改前做一些操作。

**移除：**

在移除 KVO 监听后，被观察对象的 isa 会指回原类 A，但是 NSKVONotifying_A 类并没有销毁，还保存在内存中。

**重写方法：**

NSKVONotifying_A 除了重写 setter 方法外，还重写了 class、dealloc、_isKVOA 这三个方法（可以通过 class_copyMethodList 获得），其中：

* class：返回父类的 class 对象，目的是为了不让外界知道 KVO 动态生成类的存在，隐藏 KVO 实现
* dealloc：释放 KVO 使用过程中产生的东西
* _isKVOA：用来标志它是一个 KVO 的类

可以看看我总结的 KVO 文章：[iOS - 关于 KVO 的一些总结](https://juejin.cn/post/6844903972528979976 "iOS - 关于 KVO 的一些总结")

