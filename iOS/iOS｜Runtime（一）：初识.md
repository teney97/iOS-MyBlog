## Runtime 系列文章
[深入浅出 Runtime（一）：初识](https://juejin.im/post/6844904071480999949)<br>
[深入浅出 Runtime（二）：数据结构](https://juejin.im/post/6844904072215003143)<br>
[深入浅出 Runtime（三）：消息机制](https://juejin.im/post/6844904072235974663)<br>
[深入浅出 Runtime（四）：super 的本质](https://juejin.im/post/6844904072252751880)<br>
[深入浅出 Runtime（五）：相关面试题](https://juejin.im/post/6844904072428912653)


![](https://user-gold-cdn.xitu.io/2020/2/28/17087c4d8e1b438a?w=1600&h=900&f=jpeg&s=357120)

## 大纲


![](https://user-gold-cdn.xitu.io/2020/4/19/1718e0978b130480?w=6057&h=4395&f=png&s=2564546)


## Runtime 简介
* Runtime 是一个用`C、C++、汇编`编写的运行时库，包含了很多 C 语言的 API，封装了很多动态性相关的函数；
* Objective-C 是一门动态运行时语言，允许很多操作推迟到程序运行时再进行。`OC`的动态性就是由 `Runtime` 来支撑和实现的，`Rumtime` 就是它的核心；
* 我们平时编写的`OC`代码，底层都是转换成了 `Runtime API` 进行调用。


## Objective-C 是一门动态运行时语言
#### 什么是编译时与运行时？
* 编译时：编译器将程序代码编译成计算机能够识别的语言，只进行一些简单的语法检查；
* 运行时：代码跑起来，被装载到内存中去，此时如果出错会导致程序崩溃。如经典的 crash：`unrecognized selector send to instance/class`。
#### 编译时语言与动态运行时语言的区别？
* 编译时语言：在编译期进行函数决议；
* 动态运行时语言：将函数决议推迟到运行时。

##### 举例
对于 `NSString *string = [[NSMutableArray alloc]init]`;
* 编译时：编译器进行类型检查的时候，由于给一个`NSString`类型的指针赋值的是一个`NSMutableArray`对象，所以编译器会给出类型不匹配的警告。但是编译器会将 `string`当作`NSString`的实例，所以`string`对象调用`NSString`的方法，编译没有任何问题，而调用`NSMutableArray`的方法，编译会直接报错。
* 运行时：由于`string`实际上是指向一个`NSMutableArray`对象，`NSMutableArray`对象没有`stringByAppendingString:`方法，所以导致crash：`unrecognized selector send to instance`。
```objc
    NSString *string = [[NSMutableArray alloc]init];  //⚠️Incompatible pointer types initializing 'NSString *' with an expression of type 'NSMutableArray *'
    [string stringByAppendingString:@"abc"];
    [string addObject:@"abc"];  //❌No visible @interface For 'NSString' declares the selector 'addObject:'
```


## Runtime 有两个版本
* Legacy (早期版本) ，对应的编程接口：Objective-C 1.0，应用于`32-bit programs on OS X desktop`；
* Modern (现代版本)，对应的编程接口：[Objective-C 2.0](https://developer.apple.com/documentation/objectivec/objective-c_runtime)，应用于`iPhone applications and 64-bit programs on OS X v10.5 and later`。



## Objective-C 程序在三个不同的级别上与 Runtime 系统进行交互
* 通过 Objective-C 源代码；
* 通过 Foundation 框架中 NSObject 类定义的方法，如：
```objc
// 根据 instance 对象或者类名获得一个 class 对象
- (Class)class
+ (Class)class
// 判断当前 instance/class 对象的 isa 指向是不是 class/meta-class 对象或者它的子类类型
- (BOOL)isKindOfClass:(Class)cls
+ (BOOL)isKindOfClass:(Class)cls
// 判断当前 instance/class 对象的 isa 指向是不是 class/meta-class 对象类型
- (BOOL)isMemberOfClass:(Class)cls
+ (BOOL)isMemberOfClass:(Class)cls
// 判断对象是否可以接收特定消息
- (BOOL)respondsToSelector:(SEL)sel
+ (BOOL)respondsToSelector:(SEL)sel
// 判断对象是否实现了特定协议中定义的方法
- (BOOL)conformsToProtocol:(Protocol *)protocol
+ (BOOL)conformsToProtocol:(Protocol *)protocol
// 可以根据一个 SEL，得到该方法的 IMP
- (IMP)methodForSelector:(SEL)sel
+ (IMP)methodForSelector:(SEL)sel
```
* 通过直接调用 [Runtime 函数](https://developer.apple.com/documentation/objectivec/objective-c_runtime)，如：

类相关
```objc
// 动态创建一对类和元类（参数：父类，类名，额外的内存空间）
Class objc_allocateClassPair(Class superclass, const char *name, size_t extraBytes)
// 注册一对类和元类（要在类注册之前添加成员变量）
void objc_registerClassPair(Class cls) 
// 销毁一对类和元类
void objc_disposeClassPair(Class cls)
// 获取 isa 指向的 Class
Class object_getClass(id obj)
// 设置 isa 指向的 Class
Class object_setClass(id obj, Class cls)
// 判断一个 OC 对象是否为 Class
BOOL object_isClass(id obj)
// 判断一个 Class 是否为元类
BOOL class_isMetaClass(Class cls)
// 获取父类
Class class_getSuperclass(Class cls)
```

成员变量相关
```objc
// 获取一个实例变量信息
Ivar class_getInstanceVariable(Class cls, const char *name)
// 拷贝实例变量列表（最后需要调用 free 释放）
Ivar *class_copyIvarList(Class cls, unsigned int *outCount)
// 设置和获取成员变量的值
void object_setIvar(id obj, Ivar ivar, id value)
id object_getIvar(id obj, Ivar ivar)
// 动态添加成员变量（已经注册的类是不能动态添加成员变量的）
BOOL class_addIvar(Class cls, const char * name, size_t size, uint8_t alignment, const char * types)
// 获取成员变量的相关信息
const char *ivar_getName(Ivar v)
const char *ivar_getTypeEncoding(Ivar v)
```
  属性相关
```objc
// 获取一个属性
objc_property_t class_getProperty(Class cls, const char *name)
// 拷贝属性列表（最后需要调用 free 释放）
objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount)
// 动态添加属性
BOOL class_addProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount)
// 动态替换属性
void class_replaceProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount)
// 获取属性的一些信息
const char *property_getName(objc_property_t property)
const char *property_getAttributes(objc_property_t property)
```
 方法相关
```objc
// 获得一个实例方法、类方法
Method class_getInstanceMethod(Class cls, SEL name)
Method class_getClassMethod(Class cls, SEL name)
// 方法实现相关操作
IMP class_getMethodImplementation(Class cls, SEL name) 
IMP method_setImplementation(Method m, IMP imp)
void method_exchangeImplementations(Method m1, Method m2) 
// 拷贝方法列表（最后需要调用 free 释放）
Method *class_copyMethodList(Class cls, unsigned int *outCount)
// 动态添加方法
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
// 动态替换方法
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
// 获取方法的相关信息（带有 copy 的需要调用 free 去释放）
SEL method_getName(Method m)
IMP method_getImplementation(Method m)
const char *method_getTypeEncoding(Method m)
unsigned int method_getNumberOfArguments(Method m)
char *method_copyReturnType(Method m)
char *method_copyArgumentType(Method m, unsigned int index)
// 选择器相关
const char *sel_getName(SEL sel)
SEL sel_registerName(const char *str)
// 用 block 作为方法实现
IMP imp_implementationWithBlock(id block)
id imp_getBlock(IMP anImp)
BOOL imp_removeBlock(IMP anImp)
```
  关联对象相关
  
[传送门：OC - Association 关联对象](https://www.jianshu.com/p/0de99fa1409c)
```objc
// 添加关联对象
void objc_setAssociatedObject(id object, const void * key, id value, objc_AssociationPolicy policy)
// 获得关联对象
id objc_getAssociatedObject(id object, const void * key)
// 移除指定 object 的所有关联对象
void objc_removeAssociatedObjects(id object)
```


## Runtime 都有哪些应用？
* 利用关联对象（AssociatedObject）给分类添加属性
* 遍历类的所有成员变量（修改 textfield 的占位文字颜色、字典转模型、自动归档解档）
* 交换方法实现（拦截交换系统的方法）
* 利用消息转发机制解决方法找不到的异常问题
* ......




## 相关链接

[苹果维护的 Runtime 开源代码](https://opensource.apple.com/tarballs/objc4/)<br>
[GNU 维护的 Runtime 开源代码](https://github.com/RetVal/objc-runtime)<br>
[苹果官方文档 Runtime](https://developer.apple.com/documentation/objectivec/objective-c_runtime#//apple_ref/doc/uid/TP40001418-CH1g-126286)
