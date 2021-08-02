
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65ba901acf26484194b8adce70382114~tplv-k3u1fbpfcp-zoom-1.image)

## 1.Block 的使用
#### Block 是什么？
块，封装了函数调用以及调用环境的 OC 对象，

#### Block 的声明
```objc
// 1.
@property (nonatomic, copy) void(^myBlock1)(void);
// 2.BlockType:类型别名
typedef void(^BlockType)(void);
@property (nonatomic, copy) BlockType myBlock2;
// 3.
    // 返回值类型(^block变量名)(参数1类型,参数2类型,...)
    void(^block)(void);
```

#### Block 的定义
```objc
    // ^返回值类型(参数1,参数2,...){};
    // 1.无返回值，无参数
    void(^block1)(void) = ^{
        
    };
    // 2.无返回值，有参数
    void(^block2)(int) = ^(int a){
        
    };
    // 3.有返回值，无参数（不管有没有返回值，定义的返回值类型都可以省略）
    int(^block3)(void) = ^int{
        return 3;
    };
    // 以上Block的定义也可以这样写：
    int(^block4)(void) = ^{
        return 3;
    };
    // 4.有返回值，有参数
    int(^block5)(int) = ^int(int a){
        return 3 * a;
    };
```



#### Block 的调用
```objc
    // 1.无返回值，无参数
    block1();
    // 2.有返回值，有参数
    int a = block5(2);
```

#### 使用示例
```objc
    int multiplier = 7;
    int (^myBlock)(int) = ^(int num) {
        return num * multiplier;
    };
     
    printf("%d", myBlock(3));
    // prints "21"
```
#### Block 的 Code Snippets 快捷方式
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48e63456535148c2be72f997c04bc5c0~tplv-k3u1fbpfcp-zoom-1.image)
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7634d5a0cfb442e998b516f75888ceb3~tplv-k3u1fbpfcp-zoom-1.image)

## 2.Block 的底层数据结构
* Block 本质上也是一个 OC 对象，它内部也有个`isa`指针；
* Block 是封装了函数调用以及调用环境的 OC 对象；
* Block 的底层数据结构如下图所示：

![Block 的底层数据结构.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84ec5854a0ea4776be7c8f8f04cd06ac~tplv-k3u1fbpfcp-zoom-1.image)

通过 Clang 将以下 Block 代码转换为 C++ 代码，来分析 Block 的底层实现。
```objc
// Clang
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m
```
```objc
// main.m
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        void(^block)(void) = ^{
            NSLog(@"调用了block");
        };
        block();
    }
    return 0;
}
```

* Block 底层数据结构就是一个`__main_block_impl_0`结构体对象，其中有`__block_impl`和`__main_block_desc_0`两个结构体对象成员。
    * main：表示 block 所在的函数
    * block：表示这个一个 block
    * impl：表示实现（implementation）
    * 0：表示这是该函数中的第一个 block
* `__main_block_func_0`结构体封装了 block 里的代码；
* `__block_impl`结构体才是真正定义 block 的结构，其中的`FuncPtr`指针指向`__main_block_func_0`；
* `__main_block_desc_0`是 block 的描述对象，存储着 block 的内存大小等；
*  定义 block 的本质：
    * 调用`__main_block_impl_0()`构造函数，并且给它传了两个参数`__main_block_func_0`和`&__main_block_desc_0_DATA`。拿到函数的返回值，再取返回值的地址`&__main_block_impl_0`，把这个地址赋值给 block 变量。
* 调用 block 的本质：
    * 通过`__main_block_impl_0`中的`__block_impl`中的`FuncPtr`拿到函数地址，直接调用。
```objc
// main.cpp
struct __main_block_impl_0 {
    struct __block_impl impl;         // block的结构体
    struct __main_block_desc_0* Desc; // block的描述对象，描述block的大小等
    /*  构造函数
     ** 返回值：__main_block_impl_0 结构体
     ** 参数一：__main_block_func_0 结构体
     ** 参数二：__main_block_desc_0 结构体的地址
     ** 参数三：flags 标识位
     */
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock; //_NSConcreteStackBlock 表示block存在栈上
        impl.Flags = flags; 
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

// __main_block_func_0 封装了block里的代码
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_58a448_mi_0);
}

struct __block_impl {
    void *isa;     // block的类型
    int Flags;     // 标识位
    int Reserved;  // 
    void *FuncPtr; // block的执行函数指针，指向__main_block_func_0
};

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size; // block本质结构体所占内存空间
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)}; 
 
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        /*
         ** void(^block)(void) = ^{
                NSLog(@"调用了block");
            };
         ** 定义block的本质：
         ** 调用__main_block_impl_0()构造函数
         ** 并且给它传了两个参数 __main_block_func_0 和 &__main_block_desc_0_DATA
         ** __main_block_func_0 封装了block里的代码
         ** 拿到函数的返回值，再取返回值的地址 &__main_block_impl_0，
         ** 把这个地址赋值给 block
         */
        void(*block)(void) = ((void (*)())&__main_block_impl_0(
                                                               (void *)__main_block_func_0,
                                                               &__main_block_desc_0_DATA
                                                              ));
        /*
         ** block();
         ** 调用block的本质：
         ** 通过 __main_block_impl_0 中的 __block_impl 中的 FuncPtr 拿到函数地址，直接调用
         */      
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    }
    return 0;
}
```

## 3.Block 的变量捕获机制
为了保证 block 内部能够正常访问外部的变量，block 有个变量捕获机制。
* 对于全局变量，`不会捕获`到 block 内部，访问方式为`直接访问`；
* 对于 auto 类型的局部变量，`会捕获`到 block 内部，block 内部会自动生成一个成员变量，用来存储这个变量的值，访问方式为`值传递`；
* 对于 static 类型的局部变量，`会捕获`到 block 内部，block 内部会自动生成一个成员变量，用来存储这个变量的地址，访问方式为`指针传递`；
* 对于对象类型的局部变量，block 会`连同它的所有权修饰符一起捕获`。
![block 变量捕获机制.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/797e6dd11f2c4a4d8a530aaf04ffe680~tplv-k3u1fbpfcp-zoom-1.image)

### 3.1 auto 类型的局部变量
auto 自动变量：我们定义出来的变量，默认都是 auto 类型，只是省略了。
```objc
    auto int age = 10；
```
auto 类型的局部变量`会捕获`到 block 内部，访问方式为`值传递`。

通过 Clang 将以下代码转换为 C++ 代码：
```objc
    int age = 10;
    void(^block)(void) = ^{
        NSLog(@"%d",age);
    };
    block();
```
* `__main_block_impl_0`对象内部会生成一个相同的`age`变量；
* `__main_block_impl_0()`构造函数多了个参数，用来捕获访问的外面的`age`变量的`值`，将它赋值给`__main_block_impl_0`对象内部的`age`变量。

```objc
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int age;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int age = __cself->age; // bound by copy
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_5ed490_mi_0,age);
}

......

    int age = 10;
    void(*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
```

由于是`值传递`，我们修改外部的`age`变量的值，不会影响到 block 内部的`age`变量。
```objc
    int age = 10;
    void(^block)(void) = ^{
        NSLog(@"%d",age);
    };
    age = 20;
    block();
    // 10
```

### 3.2 static 类型的局部变量
static 类型的局部变量`会捕获`到 block 内部，访问方式为`指针传递`。

通过 Clang 将以下代码转换为 C++ 代码：
```objc
    static int age = 10;
    void(^block)(void) = ^{
        NSLog(@"%d",age);
    };
    block();
```
* `__main_block_impl_0`对象内部会生成一个相同类型的`age`指针；
* `__main_block_impl_0()`构造函数多了个参数，用来捕获访问的外面的`age`变量的`地址`，将它赋值给`__main_block_impl_0`对象内部的`age`指针。
```objc
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int *age;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_age, int flags=0) : age(_age) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int *age = __cself->age; // bound by copy
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_a4bc7d_mi_0,(*age));
}

......

    static int age = 10;
    void(*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &age));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
```

由于是`指针传递`，我们修改外部的`age`变量的值，会影响到 block 内部的`age`变量。
```objc
    static int age = 10;
    void(^block)(void) = ^{
        NSLog(@"%d",age);
    };
    age = 20;
    block();
    // 20
```

### 3.3 全局变量
全局变量`不会捕获`到 block 内部，访问方式为`直接访问`。

通过 Clang 将以下代码转换为 C++ 代码：
```objc
int _age = 10;
static int _height = 20;
......
        void(^block)(void) = ^{
            NSLog(@"%d,%d",_age,_height);
        };
        block();
```
* `__main_block_impl_0`对象内并没有生成对应的变量，也就是说全局变量没有捕获到 block 内部，而是直接访问。
```objc
int _age = 10;
static int _height = 20;

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_12efa5_mi_0,_age,_height);
}

......

    void(*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
```

#### 为什么局部变量需要捕获，全局变量不用捕获呢？
* 作用域的原因，全局变量哪里都可以直接访问，所以不用捕获；
* 局部变量，外部不能直接访问，所以需要捕获；
* auto 类型的局部变量可能会销毁，其内存会消失，block 将来执行代码的时候不可能再去访问那块内存，所以捕获其值；
* static 变量会一直保存在内存中， 所以捕获其地址即可。


### 3.4 对象类型的 auto 变量
当 block 内部访问了对象类型的 auto 变量时：
* 如果 block 是在栈上，将不会对 auto 变量产生强引用
* 如果 block 被拷贝到堆上
    <br>① block 内部的 desc 结构体会新增两个函数：
    <br>&emsp;copy（`__main_block_copy_0`，函数名命名规范同`__main_block_impl_0`）
    <br>&emsp;dispose（`__main_block_dispose_0`）
    <br>② 会调用 block 内部的 copy 函数
    <br>③ copy 函数内部会调用`_Block_object_assign`函数
    <br>④ `_Block_object_assign`函数会根据 auto 变量的修饰符（`__strong`、`__weak`、`__unsafe_unretained`）做出相应的操作，形成强引用（retain）或者弱引用
* 如果 block 从堆上移除
    <br>① 会调用 block 内部的 dispose 函数
    <br>② dispose 函数内部会调用`_Block_object_dispose`函数
    <br>③ `_Block_object_dispose`函数会自动释放引用的 auto 变量（release）

函数|调用时机
:--|:--
copy 函数|栈上的 block 复制到堆时
dispose 函数|堆上的 block 被废弃时

如下代码，block 保存在堆中，当执行完作用域2的时候，Person 对象并没有被释放，而是在执行完作用域1的时候释放，说明 block 内部对 Person 对象产生了强引用。
```objc
typedef void(^MyBlock)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool { //作用域1
        MyBlock block;
        { //作用域2
            Person *p = [Person new];
            p.name = @"zhangsan";      
            block = ^{
                NSLog(@"%@",p.name);
            };
        }
        NSLog(@"-----");
    }
    return 0;
}
// -----
// Person-dealloc
```
通过 Clang 将以上代码转换为 C++ 代码：
```objc
// 弱引用需要运行时的支持，所以需要加上 -fobjc-arc -fobjc-runtime=ios-8.0.0
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-8.0.0 main.m
```
`__main_block_impl_0`中生成了一个`Person *__strong p`指针，指向外面的 person 对象，且是强引用。
```objc
typedef void(*MyBlock)(void);

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    Person *__strong p; // 强引用
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, Person *_p, int flags=0) : p(_p) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    Person *__strong p = __cself->p; // bound by copy
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_9e5699_mi_1,((NSString *(*)(id, SEL))(void *)objc_msgSend)((id)p, sel_registerName("name")));
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->p, (void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/);}
static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        MyBlock block;
        {
            Person *p = ((Person *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Person"), sel_registerName("new"));
            ((void (*)(id, SEL, NSString * _Nonnull))(void *)objc_msgSend)((id)p, sel_registerName("setName:"), (NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_9e5699_mi_0);

            block = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, p, 570425344));
        }
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_9e5699_mi_2);
    }
    return 0;
}
```
添加了`__weak`修饰后，当执行完作用域2的时候，Person 对象就被被释放了。
```objc
typedef void(^MyBlock)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool { //作用域1
        MyBlock block;
        { //作用域2
            __weak Person *p = [Person new];
            p.name = @"zhangsan";      
            block = ^{
                NSLog(@"%@",p.name);
            };
        }
        NSLog(@"-----");
    }
    return 0;
}
// Person-dealloc
// -----
```
同样的，通过 Clang 将以上代码转换为 C++ 代码。

`__main_block_impl_0`中生成了一个`Person *__weak p`指针，指向外面的 person 对象，且是弱引用。
说明当 block 内部 访问了对象类型的 auto 变量时，如果 block 被拷贝到堆上，会连同对象的所有权修饰符一起捕获。
```objc
......
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    Person *__weak p; //弱引用
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, Person *__weak _p, int flags=0) : p(_p) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    Person *__weak p = __cself->p; // bound by copy
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_c61841_mi_1,((NSString *(*)(id, SEL))(void *)objc_msgSend)((id)p, sel_registerName("name")));
}
......
```


### 3.5 __block 修饰的变量

#### 3.5.1 __block 作用
默认情况下 block 是不能修改外面的 auto 变量的，解决办法？
* 变量用 static 修饰（原因：捕获 static 类型的局部变量是指针传递，可以访问到该变量的内存地址）
* 全局变量
* __block（我们只希望临时用一下这个变量临时改一下而已，而改为 static 变量和全局变量会一直在内存中）

#### 3.5.2 __block 修饰符
* __block 可以用于解决 block 内部无法修改 auto 变量值的问题；
* __block 不能修饰全局变量、静态变量；
* 编译器会将 __block 变量包装成一个对象（`struct __Block_byref_age_0`（byref：按地址传递））；
* 加 __block 修饰不会修改变量的性质，它还是 auto 变量；
* 一般情况下，对被捕获变量进行赋值(赋值!=使用)操作需要添加 __block 修饰符。比如给数组添加或者删除对象，就不用加 __block 修饰；
* 在 MRC 下使用 __block 修饰对象类型，在 block 内部不会对该对象进行 retain 操作，所以在 MRC 环境下可以通过 __block 解决循环引用的问题。


**使用示例**
```objc
    __block int age = 10;
    void(^block)(void) = ^{
        age = 20;
        NSLog(@"block-%d",age);
    };
    block();
    NSLog(@"%d",age);
    // block-20
    // 20
```
通过 Clang 将以上代码转换为 C++ 代码。
* 编译器会将 __block 修饰的变量包装成一个`__Block_byref_age_0`对象；
* 以上`age = 20;`的赋值过程为：通过 block 结构体里的（`__Block_byref_age_0`）类型的 age 指针，找到`__Block_byref_age_0`结构体的内存（即被 __block 包装成对象的内存），把`__Block_byref_age_0`结构体里的 age 变量的值改为20。
* 由于编译器将 __block 变量包装成了一个对象，所以它的内存管理几乎等同于访问对象类型的 auto 变量，但还是有差异，下面会讲到。
```objc
struct __Block_byref_age_0 {
    void *__isa;
    __Block_byref_age_0 *__forwarding;
    int __flags;
    int __size;
    int age; 
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_age_0 *age; // by ref
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_age_0 *_age, int flags=0) : age(_age->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_age_0 *age = __cself->age; // bound by ref
    (age->__forwarding->age) = 20;
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_9578d0_mi_0,(age->__forwarding->age));
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->age, (void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);}
static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        __attribute__((__blocks__(byref))) __Block_byref_age_0 age = {(void*)0,(__Block_byref_age_0 *)&age, 0, sizeof(__Block_byref_age_0), 10};
        void(*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_age_0 *)&age, 570425344));
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_77_f_d18dtx6277bxbcd8s72my80000gn_T_main_9578d0_mi_1,(age.__forwarding->age));
    }
    return 0;
}
```
#### 3.5.3 __block 的内存管理
* 当 block 在栈上时，并不会对 __block 变量产生强引用
* 当 block 被 copy 到堆时
    <br>① block 内部的 desc 结构体会新增两个函数：
    <br>&emsp;copy（`__main_block_copy_0`，函数名命名规范同`__main_block_impl_0`）
    <br>&emsp;dispose（`__main_block_dispose_0`）
    <br>② 会调用 block 内部的 copy 函数
    <br>③ copy 函数内部会调用`_Block_object_assign`函数
    <br>④ `_Block_object_assign`函数会对 __block 变量形成强引用（retain）
* 当 block 从堆中移除时
    <br>① 会调用 block 内部的 dispose 函数
    <br>② dispose 函数内部会调用`_Block_object_dispose`函数
    <br>③ `_Block_object_dispose`函数会自动释放引用的 __block 变量（release）
![__block 的内存管理.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7475013d6bd1421b8b9f5e8840608b94~tplv-k3u1fbpfcp-zoom-1.image)

#### 3.5.4 __block 的 __forwarding 指针
##### __block 的`__forwarding`指针存在的意义？
##### 为什么要通过 age 结构体里的`__forwarding`指针拿到 age 变量的值，而不直接 age 结构体拿到 age 变量的值呢？
__block 的`__forwarding`是指向自己本身的指针，
为了不论在任何内存位置，都可以顺利的访问同一个 __block 变量。
* block 对象 copy 到堆上时，内部的 __block 变量也会 copy 到堆上去。为了防止 age 的值赋值给栈上的 __block 变量，就使用了`__forwarding`；
* 当 __block 变量在栈上的时候，__block 变量的结构体中的`__forwarding`指针指向自己，这样通过`__forwarding`取到结构体中的 age 给它赋值没有问题；
* 当 __block 变量 copy 到堆上后，栈上的`__forwarding`指针会指向 copy 到堆上的 _block 变量结构体，而堆上的`__forwarding`指向自己；

这样不管我们访问的是栈上还是堆上的 __block 变量结构体，只要是通过`__forwarding`指针访问，都是访问到堆上的 __block 变量结构体；给 age 赋值，就肯定会赋值给堆上的那个 __block 变量中的 age。
![__forwarding.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50191bfdcaaa474397af23e2c14ba0ac~tplv-k3u1fbpfcp-zoom-1.image)


#### 3.5.5 对象类型的 auto 变量、__block 变量内存管理区别
* 当 block 在栈上时，对它们都不会产生强引用
* 当 block 拷贝到堆上时，都会通过 copy 函数来处理它们

对象类型的 auto 变量<br>（假设变量名叫做p）| __block 变量<br>（假设变量名叫做a）
:--|:--
`_Block_object_assign((void*)&dst->p, (void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/); `|`_Block_object_assign((void*)&dst->a, (void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);`
`_Block_object_assign`函数会根据 auto 变量的修饰符（`__strong`、`__weak`、`__unsafe_unretained`）做出相应的操作，形成强引用（retain）或者弱引用|`_Block_object_assign`函数会对 __block 变量形成强引用（retain）

* 当 block 从堆上移除时，都会通过 dispose 函数来释放它们

对象类型的 auto 变量<br>（假设变量名叫做p）|__block 变量<br>（假设变量名叫做a）
:--|:--
`_Block_object_dispose((void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/);`|`_Block_object_dispose((void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);`


#### 3.5.6 被 __block 修饰的对象类型
* 当 __block 变量在栈上时，不会对指向的对象产生强引用
* 当 __block 变量被 copy 到堆时
    <br>①`__Block_byref_object_0`即 __block 变量内部会新增两个函数：
    <br>&emsp;copy（`__Block_byref_id_object_copy`）
    <br>&emsp;dispose（`__Block_byref_id_object_dispose`）
    <br>② 会调用 __block 变量内部的 copy 函数
    <br>③ copy 函数内部会调用`_Block_object_assign`函数
    <br>④ `_Block_object_assign`函数会根据所指向对象的修饰符（`__strong`、`__weak`、`__unsafe_unretained`）做出相应的操作，形成强引用（retain）或者弱引用（注意：这里仅限于 ARC 时会 retain，MRC 时不会 retain）
* 如果 __block 变量从堆上移除
    <br>① 会调用 __block 变量内部的 dispose 函数
    <br>② dispose 函数内部会调用`_Block_object_dispose`函数
    <br>③ `_Block_object_dispose`函数会自动释放指向的对象（release）



```objc
int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    @autoreleasepool {
        // Setup code that might create autoreleased objects goes here.
        appDelegateClassName = NSStringFromClass([AppDelegate class]);
        
        __block NSObject *object = [[NSObject alloc] init];
        void(^block)(void) = ^{
            object = [[NSObject alloc] init];
        };
    }
    return UIApplicationMain(argc, argv, nil, appDelegateClassName);
}
```

```objc
struct __Block_byref_object_0 {
    void *__isa;
    __Block_byref_object_0 *__forwarding;
    int __flags;
    int __size;
    void (*__Block_byref_id_object_copy)(void*, void*); // copy
    void (*__Block_byref_id_object_dispose)(void*);     // dispose
    NSObject *__strong object;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_object_0 *object; // by ref
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_object_0 *_object, int flags=0) : object(_object->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_object_0 *object = __cself->object; // bound by ref
    (object->__forwarding->object) = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->object, (void*)src->object, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->object, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        appDelegateClassName = NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class")));

        __attribute__((__blocks__(byref))) __Block_byref_object_0 object = {
            (void*)0,
            (__Block_byref_object_0 *)&object, 
            33554432, 
            sizeof(__Block_byref_object_0), 
            __Block_byref_id_object_copy_131,
            __Block_byref_id_object_dispose_131, 
            ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"))
        };
        void(*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_object_0 *)&object, 570425344));
    }
    return UIApplicationMain(argc, argv, __null, appDelegateClassName);
}

static void __Block_byref_id_object_copy_131(void *dst, void *src) {
    // __block 对象结构体的地址+40个字节，即为结构体中 object 对象的地址
    _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
static void __Block_byref_id_object_dispose_131(void *src) {
    _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
```
>**注意**：在 MRC 下使用 __block 修饰对象类型，在 block 内部不会对该对象进行 retain 操作，所以在 MRC 环境下可以通过 __block 解决循环引用的问题。

**示例（MRC）**
```objc
// 对象类型的捕获，连同所有权修饰符一起捕获，所以 block 对 person 强引用
    HTPerson *person = [[HTPerson alloc] init];

    void(^block)(void)  = [^{
        NSLog(@"%p", person);
    } copy];
        
    [person release];
        
    block();
        
    [block release];

// 0x10053da30
// -[HTPerson dealloc]
```

```objc
// __block 修饰的对象类型的捕获，MRC 下在 block 内部不会对该对象进行 retain 操作
    __block HTPerson *person = [[HTPerson alloc] init];

    void(^block)(void)  = [^{
        NSLog(@"%p", person);
    } copy];
        
    [person release];
        
    block();
        
    [block release];

// -[HTPerson dealloc]
// 0x1007337d0
```






## 4.Block 的类型
block 有 3 种类型，可以通过调用 class 方法或者 isa 指针 查看具体类型，最终都是继承自 NSBlock 类型。

block类型|描述|环境
:--:|:--:|:--:
__ NSGlobalBlock __<br>（ _NSConcreteGlobalBlock ）|全局block，保存在数据段|没有访问auto变量
__ NSStackBlock __<br>（ _NSConcreteStackBlock ）|栈block，保存在栈区|访问了auto变量
__ NSMallocBlock __<br>（ _NSConcreteMallocBlock ）|堆block，保存在堆区|__ NSStackBlock __调用了copy

打印各种 block 的类型，以及遍历 block 的父类类型，如下：
```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        void(^block1)(void) = ^{
            NSLog(@"hello");
        };
        /*  block2
         ** 在ARC下会自动copy，从栈复制到堆，所以为__NSMallocBlock__类型
         ** 在MRC下为__NSStackBlock__类型，需要手动调用copy方法才会变为__NSMallocBlock__类型
         **     同时，在不需要该block的时候需要手动调用release方法
         */ 
        int age = 10;
        void(^block2)(void) = ^{
            NSLog(@"%d",age);
        };
        
        NSLog(@"%@,%@,%@", [block1 class], [block2 class], [^{
            NSLog(@"%d",age);
        } class]);
        
        Class class = [block1 class];
        while (class) {
            NSLog(@"%@",class);
            class = [class superclass];
        }
    }
    return 0;
}
// __NSGlobalBlock__,__NSMallocBlock__,__NSStackBlock__
// __NSGlobalBlock__
// __NSGlobalBlock
// NSBlock
// NSObject
```
每一种类型的 block 调用 copy 后的结果如下所示：
block类型|副本源的配置存储区|复制效果
:--|:--|:--
_NSConcreteGlobalBlock|程序的数据段区|什么也不做
_NSConcreteStackBlock|栈|从栈复制到堆
_NSConcreteMallocBlock|堆|引用计数增加

#### __ NSStackBlock __ 存在的问题：
以下是在 MRC 环境下，block 类型为`__NSStackBlock__`。
当 test() 函数执行完毕，栈上的东西可能会被销毁，数据就会变成垃圾数据。尽管 block 还能正常调用，但是输出的 age 的值发生了错乱。
```objc
void (^block)(void);
void test()
{    
    // __NSStackBlock__
    int age = 10;
    block = ^{
        NSLog(@"%d", age);
    };
}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        test();
        block();
    }
    return 0;
}
// 272632936
```
解决办法：调用`copy`方法，将栈 block 复制到堆。
```objc
void (^block)(void);
void test()
{    
    // __NSMallocBlock__
    int age = 10;
    block = [^{
        NSLog(@"%d", age);
    } copy];
}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        test();
        block();
        [block release];
    }
    return 0;
}
// 10
```

## 5.Block 的 copy
在 `ARC` 环境下，编译器会根据情况自动将栈上的 block 复制到堆上，比如以下几种情况：
* 手动调用 block 的 copy 方法时；
* block 作为函数返回值时（Masonry 框架中用很多）；
* 将 block 赋值给`__strong`指针时；
* block 作为 Cocoa API 中方法名含有`usingBlock`的方法参数时；
* block 作为 GCD API 的方法参数时。

#### block 作为属性的写法：
* ARC 下写`strong`或者`copy`都会对 block 进行强引用，都会自动将 block 从栈 copy 到堆上；
* 建议都写成`copy`，这样 MRC 和 ARC 下一致。
```objc
// MRC
@property (nonatomic, copy) void(^block)(void);
// ARC
@property (nonatomic, copy) void(^block)(void);
@property (nonatomic, strong) void(^block)(void);
```

## 6.Block 的循环引用问题

#### 为什么 block 会产生循环引用？
* ① **相互循环引用：** 如果当前 block 对当前对象的某一成员变量进行捕获的话，可能会对它产生强引用。而当前 block 又由于当前对象对其有一个强引用，就产生了相互循环引用的问题；
* ② **大环引用：** 我们如果使用`__block`的话，在 ARC 下可能会产生循环引用（MRC 则不会），在 ARC 下可以通过断环的方式去解除循环引用。但是有一个弊端，如果该 block 一直得不到调用，循环引用就一直存在。

### 6.1 ARC
* 用`__weak`或者`__unsafe_unretained`解决：
```objc
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        NSLog(@"%p",weakSelf);
    };
```
```objc
    __unsafe_unretained id weakSelf = self;
    self.block = ^{
        NSLog(@"%p",weakSelf);
    };
```
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6dfbad0c08b4595b1b1a68732e0928c~tplv-k3u1fbpfcp-zoom-1.image)

>**注意**：`__unsafe_unretained`会产生悬垂指针。

* 用`__block`解决（必须要调用 block）：
缺点：必须要调用 block，而且 block 里要将指针置为 nil。如果一直不调用 block，对象就会一直保存在内存中，造成内存泄漏。
```objc
    __block id weakSelf = self;
    self.block = ^{
        NSLog(@"%p",weakSelf);
        weakSelf = nil;
    };
    self.block();
```
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/891d90f43f7e48e590f311079ef19d48~tplv-k3u1fbpfcp-zoom-1.image)


### 6.2 MRC
* 用`__unsafe_unretained`解决：同 ARC
* 用`__block`解决（在 MRC 下使用 __block 修饰对象类型，在 block 内部不会对该对象进行 retain 操作，所以在 MRC 环境下可以通过 __block 解决循环引用的问题）
```objc
    __block id weakSelf = self;
    self.block = ^{
        NSLog(@"%p",weakSelf);
    };
```


## 7.相关面试题
#### Q：block 的本质是什么？
封装了函数调用以及调用环境的 OC 对象。
#### Q：block 的属性修饰词为什么是 copy？使用 block 有哪些使用注意？
block 一旦没有进行 copy 操作，就不会在堆上。

使用注意：循环引用问题。
#### Q：block在给 NSMutableArray 添加或移除对象，需不需要添加 __block？
不需要。
#### Q：block 的变量捕获机制
block 的变量捕获机制，是为了保证 block 内部能够正常访问外部的变量。
* 对于全局变量，`不会捕获`到 block 内部，访问方式为`直接访问`；
* 对于 auto 类型的局部变量，`会捕获`到 block 内部，block 内部会自动生成一个成员变量，用来存储这个变量的值，访问方式为`值传递`；
* 对于 static 类型的局部变量，会捕获到 block 内部，block 内部会自动生成一个成员变量，用来存储这个变量的地址，访问方式为`指针传递`；
* 对于对象类型的局部变量，block 会`连同它的所有权修饰符一起捕获`。

#### Q：为什么局部变量需要捕获，全局变量不用捕获呢？
* 作用域的原因，全局变量哪里都可以直接访问，所以不用捕获；
* 局部变量，外部不能直接访问，所以需要捕获；
* auto 类型的局部变量可能会销毁，其内存会消失，block 将来执行代码的时候不可能再去访问那块内存，所以捕获其值；
* static 变量会一直保存在内存中， 所以捕获其地址即可。
#### Q：self 会不会捕获到 block 内部？
会捕获。
OC 方法都有两个隐式参数，方法调用者`self`和方法名`_cmd`。
参数也是一种局部变量。
#### Q：_name 会不会捕获到 block 内部？
会捕获。
不是将`_name`变量进行捕获，而是直接将`self`捕获到 block 内部，因为`_name`是 Person 类的成员变量，`_name`来自当前的对象/方法调用者`self（self->_name）`。

如果使用`self.name`即调用`self`的`getter`方法，即给`self`对象发送一条消息，那还是要访问到`self`。`self`是局部变量，不是全局变量，所以`self`会捕获到 block 内部。

#### Q：__ NSStackBlock __ 存在的问题：
如果没有将 block 从栈上 copy 到堆上，那我们访问栈上的 block 的话，可能会由于变量作用域结束导致栈上的 block 以及 __block 变量被销毁，而造成内存崩溃。或者数据可能会变成垃圾数据，尽管将来 block 还能正常调用，但是它捕获的变量的值已经错乱了。

解决办法：将 block 的内存放堆里，意味着它就不会自动销毁，而是由我们程序员来决定什么时候销毁它。

#### Q：默认情况下 block 是不能修改外面的 auto 变量的，解决办法？
* 变量用 static 修饰（原因：捕获 static 类型的局部变量是指针传递，可以访问到该变量的内存地址）
* 全局变量
* __block（我们只希望临时用一下这个变量临时改一下而已，而改为 static 变量和全局变量会一直在内存中）

#### Q：__block 修饰符使用注意点：
在 MRC 下使用 __block 修饰对象类型，在 block 内部不会对该对象进行 retain 操作，所以在 MRC 环境下可以通过 __block 解决循环引用的问题。


#### Q：__block 的 __forwarding 指针存在的意义？为什么要通过 age 结构体里的 __forwarding 指针拿到 age 变量的值，而不直接 age 结构体拿到 age 变量的值呢？
见 3.5.4 __block 的 __forwarding 指针。

#### Q：为什么通过 __weak 去修饰成员变量或对象就可以达到规避循环引用的目的呢？
block 对于对象类型的局部变量连同所有权修饰符一起截获，所以如果我们在外部定义的对象是 __weak 所有权修饰符的，那么在 block 中所产生的结构体里所持有的变量也是 __weak 类型的。

#### Q：解决在 block 内部通过弱指针访问对象成员时编译器报错的问题：
```objc
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        __strong typeof(weakSelf) strongSelf = weakSelf; //避免执行过程中提前释放
        NSLog(@"%d",strongSelf->age);
    };
```
#### Q：以下代码的打印结果是？
```objc
    __block int multiplier = 6;
    int (^block)(int) = ^(int num) {
        return num * multiplier;
    };
    multiplier = 4;
    NSLog(@"%d",block(2));
    // 8
```
#### Q：以下代码有问题吗？
```objc
    __block id weakSelf = self;
    self.block = ^{
        NSLog(@"%p",weakSelf);
    };
```
* 在 MRC 下，不会产生循环引用；
* 在 ARC 下，会产生循环引用，导致内存泄漏，解决方案如下。

```objc
    __block id weakSelf = self;
    self.block = ^{
        NSLog(@"%p",weakSelf);
        weakSelf = nil;
    };
    self.block();
```
缺点：必须要调用 block，而且 block 里要将指针置为 nil。如果一直不调用 block，对象就会一直保存在内存中，造成内存泄漏。
