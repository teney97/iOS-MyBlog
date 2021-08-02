
本期 [「 iOS 摸鱼周报 第十八期」](https://mp.weixin.qq.com/s/JsGmu7pzYLI3Svrmk5i2cA) 面试解析模块讲解的知识点是 `block 的变量捕获机制`。掌握了 block 的变量捕获机制，我们就能更好的应对内存管理，避免因使用不当造成内存泄漏。


## block 的变量捕获机制

block 的变量捕获机制，是为了保证 block 内部能够正常访问外部的变量。

|变量类型|是否捕获到 block 内部|访问方式
|:--|:--|:--|
|全局变量|否|直接访问|
|局部变量（auto 类型）|是|值传递|
|局部变量（static 类型）|是|指针传递|

对于全局变量，不会捕获到 block 内部，访问方式为`直接访问`。作用域的原因，全局变量哪里都可以直接访问，所以不用捕获。而对于局部变量，外部不能直接访问，所以需要捕获。下面我们来看一下 block 对于局部变量的具体捕获机制。

### auto 类型的局部变量

auto 类型的局部变量（我们定义出来的变量，默认都是 auto 类型，只是省略了），block 内部会自动生成一个同类型成员变量，用来存储这个变量的值，访问方式为`值传递`。**auto 类型的局部变量可能会销毁，其内存会消失，block 将来执行代码的时候不可能再去访问那块内存，所以捕获其值**。由于是值传递，我们修改 block 外部被捕获变量的值，不会影响到 block 内部捕获的变量值。

### static 类型的局部变量

static 类型的局部变量，block 内部会自动生成一个同类型成员变量，用来存储这个变量的地址，访问方式为`指针传递`。static 变量会一直保存在内存中， 所以捕获其地址即可。相反，由于是指针传递，我们修改 block 外部被捕获变量的值，会影响到 block 内部捕获的变量值。    

### 对象类型的局部变量

对于对象类型的局部变量，block 会连同它的所有权修饰符一起捕获。

* 如果 block 是在栈上，将不会对对象产生强引用
* 如果 block 被拷贝到堆上，将会调用 block 内部的 `copy(__funcName_block_copy_num)`函数，copy 函数内部又会调用 `assign(_Block_object_assign)`函数，assign 函数将会根据变量的所有权修饰符做出相应的操作，形成强引用（retain）或者弱引用。
* 如果 block 从堆上移除，也就是被释放的时候，会调用 block 内部的 `dispose(_Block_object_dispose)`函数，dispose 函数会自动释放引用的变量（release）。
   
### 对于 __block 修饰的变量
    
对于 `__block`（可用于解决 block 内部无法修改 auto 变量值的问题） 修饰的变量，编译器会将 `__block` 变量包装成一个 `__Block_byref_varName_num` 对象。它的内存管理几乎等同于访问对象类型的 auto 变量，但还是有差异。

* 如果 block 是在栈上，将不会对 `__block` 变量产生强引用
* 如果 block 被拷贝到堆上，将会调用 block 内部的 copy 函数，copy 函数内部又会调用 assign 函数，assign 函数将会直接对 `__block` 变量形成强引用（retain）。
* 如果 block 从堆上移除，也就是被释放的时候，会调用 block 内部的 dispose 函数，dispose 函数会自动释放引用的 `__block` 变量（release）。


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8436077c33d46088d58f3cb3b272ac2~tplv-k3u1fbpfcp-watermark.image)
 
被 `__block `修饰的对象类型的内存管理：

* 如果 `__block` 变量是在栈上，将不会对指向的对象产生强引用
* 如果 `__block` 变量被拷贝到堆上，将会调用 `__block` 变量内部的 `copy(__Block_byref_id_object_copy)`函数，copy 函数内部会调用 assign 函数，assign 函数又会根据变量的所有权修饰符做出相应的操作，形成强引用（retain）或者弱引用。（注意：这里仅限于 ARC 下会 retain，MRC 下不会 retain，所以在 MRC 下还可以通过 `__block` 解决循环引用的问题）
* 如果 `__block` 变量从堆上移除，会调用 `__block` 变量内部的 dispose 函数，dispose 函数会自动释放指向的对象（release）。
    
## 使用 block 时避免造成内存泄漏    
    
掌握了 block 的变量捕获机制，我们就能更好的应对内存管理，避免因使用不当造成内存泄漏。

常见的 block 循环引用为：`self(obj) -> block -> self(obj)`。这里 block 强引用了 self 是因为对于对象类型的局部变量，block 会连同它的所有权修饰符一起捕获，而对象的默认所有权修饰符为 __strong。

```objectivec
self.block = ^{
    NSLog(@"%@", self);
};
```

> 为什么这里说 self 是局部变量？因为 self 是 OC 方法的一个隐式参数。

为了避免循环引用，我们可以使用 `__weak` 解决，这里 block 将不再持有 self。

```objectivec
__weak typeof(self) weakSelf = self;
self.block = ^{
    NSLog(@"%@", weakSelf);
};
```

为了避免在 block 调用过程中 self 提前释放，我们可以使用 `__strong` 在 block 执行过程中持有 self，这就是所谓的 Weak-Strong-Dance。

```objectivec
__weak typeof(self) weakSelf = self;
self.block = ^{
    __strong typeof(self) strongSelf = weakSelf;
    NSLog(@"%@", strongSelf);
};
```

当然，我们平常用的比较多的还是 `@weakify(self)` 和 `@strongify(self)` 啦。

```objectivec
@weakify(self);
self.block = ^{
    @strongify(self);
    NSLog(@"%@", self);
};
```

如果你使用的是 RAC 的 Weak-Strong-Dance，你还可以这样：

```objectivec
@weakify(self, obj1, obj2);
self.block = ^{
    @strongify(self, obj1, obj2);
    NSLog(@"%@", self);
};
```

如果是嵌套的 block：

```objectivec
@weakify(self);
self.block = ^{
    @strongify(self);
    self.block2 = ^{
        @strongify(self);
        NSLog(@"%@", self);
    }
};
```

你是否会疑问，为什么内部不需要再写 @weakify(self) ？这个问题就留给你自己去思考和解决吧！

相比于简单的相互循环引用，block 造成的大环引用更需要你足够细心以及敏锐的洞察力，比如：

```objectivec
TYAlertView *alertView = [TYAlertView alertViewWithTitle:@"TYAlertView" message:@"This is a message, the alert view containt text and textfiled. "];
[alertView addAction:[TYAlertAction actionWithTitle:@"取消" style:TYAlertActionStyleCancle handler:^(TYAlertAction *action) {
    NSLog(@"%@-%@", self, alertView);
}]];
self.alertController = [TYAlertController alertControllerWithAlertView:alertView preferredStyle:TYAlertControllerStyleAlert];
[self presentViewController:alertController animated:YES completion:nil];
```

这里循环引用有两处：

1. `self -> alertController -> alertView -> handlerBlock -> self`
2. `alertView -> handlerBlock -> alertView`

避免循环引用：

```objectivec
TYAlertView *alertView = [TYAlertView alertViewWithTitle:@"TYAlertView" message:@"This is a message, the alert view containt text and textfiled. "];
@weakify(self, alertView);
[alertView addAction:[TYAlertAction actionWithTitle:@"取消" style:TYAlertActionStyleCancle handler:^(TYAlertAction *action) {
    @strongify(self, alertView);
    NSLog(@"%@-%@", self, alertView);
}]];
self.alertController = [TYAlertController alertControllerWithAlertView:alertView preferredStyle:TYAlertControllerStyleAlert];
[self presentViewController:alertController animated:YES completion:nil];
```

> 另外再和你提一个小知识点，当我们在 block 内部直接使用 _variable 时，编译器会给我们警告：`Block implicitly retains self; explicitly mention 'self' to indicate this is intended behavior`。
>
> 原因是 block 中直接使用 `_variable` 会导致 block 隐式的强引用 self。Xcode 认为这可能会隐式的导致循环引用，从而给开发者带来困扰，而且如果不仔细看的话真的不太好排查，笔者之前就因为这个循环引用找了半天，还拉上了我导师一起查找原因。所以警告我们要显式的在 block 中使用 self，以达到 block 显式 retain 住 self 的目的。改用 `self->_variable` 或者 `self.variable`。
> 
> 你可能会觉得这种困扰没什么，如果你使用 `@weakify` 和 `@strongify` 那确实不会造成循环引用，因为 `@strongify` 声明的变量名就是 self。那如果你使用 `weak typeof(self) weak_self = self;` 和 `strong typeof(weak_self) strong_self = weak_self` 呢？
