![网络配图](https://user-gold-cdn.xitu.io/2020/3/7/170b0ef52e70577d?w=1240&h=698&f=jpeg&s=106102)

## 1. load
### 1.1 load 方法的调用

* ① **调用时刻：**
    * `+load`方法会在`Runtime`加载类、分类时调用（不管有没有用到这些类，在程序运行起来的时候都会加载进内存，并调用`+load`方法）；
    * 每个类、分类的`+load `，在程序运行过程中只调用一次（除非开发者手动调用）。
* ② **调用方式：** 系统自动调用`+load `方式为直接通过函数地址调用，开发者手动调用`+load `方式为消息机制`objc_msgSend`函数调用。
* ③ **调用顺序：**
    * 先调用类的`+load `，按照编译先后顺序调用（先编译，先调用），调用子类的`+load `之前会先调用父类的`+load `；
    * 再调用分类的`+load `，按照编译先后顺序调用（先编译，先调用）（注意：分类的其它方法是：后编译，优先调用）。

### 1.2 场景分析

Person 以及它的两个分类 Person (Test)、Person (Test2) 都实现了`+test`和`+load`两个方法，且 Person (Test2)  最后编译。调用 Person 的`+test`，并打印 Person 元类对象中的类方法列表，查看打印结果。

```objc
// Person
#import <Foundation/Foundation.h>
@interface Person : NSObject
+ (void)test;
+ (void)load;
@end

#import "Person.h"
@implementation Person
- (void)test {
    NSLog(@"person test");
}
+ (void)load {
    NSLog(@"person +load");
}
@end

// Person (Test)
...
// Person (Test2)
...

#import "ViewController.h"
#import "Person.h"
#import <objc/runtime.h>
@implementation ViewController

void printAllMethodNamesOfClass(Class cls)
{
    u_int count;
    Method *methods = class_copyMethodList(cls, &count);
    NSMutableString *methodNames = [NSMutableString string];
    for (int i = 0; i < count ; i++)
    {
        Method method = methods[i];
        NSString *methodName = NSStringFromSelector(method_getName(method));
        [methodNames appendString:methodName];
        [methodNames appendString:@", "];
    }
    free(methods);    
    NSLog(@"%@: %@",cls,methodNames);
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [Person test];    
    // 打印 Person 元类对象中的类方法列表
    printAllMethodNamesOfClass(object_getClass([Person class]));
}
@end

/*
2020-02-18 22:05:33.114202+0800 Category [32631:7237868] person +load
2020-02-18 22:05:33.114929+0800 Category [32631:7237868] Person (Test) +load
2020-02-18 22:05:33.115009+0800 Category [32631:7237868] Person (Test2) +load
2020-02-18 22:05:33.209976+0800 Category [32631:7237868] Person (Test2) test
2020-02-18 22:05:33.210124+0800 Category [32631:7237868] Person: load, test, load, test, load, test,
*/
```
以上打印结果可以验证 Category 的几个原理：

* ① 分类方法会“覆盖”同名的宿主类方法，不是真正的覆盖，只是顺序超前了，通过`Runtime API`打印方法可以得知。
* ② 同名分类方法谁能生效取决于编译顺序，最后参与编译的分类中的同名方法会最终生效。

那么，为什么分类也实现了`+load`方法，却不是只调用 Person (Test2) 的`+load`，而是全部调用呢？下面我们进入源码分析。

### 1.3 源码分析
**函数调用栈：**
* objc-os.mm
    <br>① _objc_init：`Runtime`的入口函数，进行一些初始化操作
* objc-runtime-new.mm
    <br>② load_images
    <br>③ prepare_load_methods
    <br>&emsp;schedule_class_load
    <br>&emsp;add_class_to_loadable_list
    <br>&emsp;add_category_to_loadable_list
    <br>④ call_load_methods
    <br>&emsp;call_class_loads
    <br>&emsp;call_category_loads
    <br>&emsp; (*load_method)(cls, SEL_load) **核心函数**

**load_images**


![](https://user-gold-cdn.xitu.io/2020/3/21/170f8e58f1b30360?f=png&s=163027)

**call_load_methods**

先调用类的`+load`方法，再调用分类的`+load`方法。

![](https://user-gold-cdn.xitu.io/2020/3/21/170f91b3dccdc8c9?w=1598&h=1490&f=png&s=285584)

**call_class_loads & call_category_loads**

类和分类的`+load`方法是直接通过函数地址调用，所以都会调用。而其他方法如`+test`则是通过消息机制`objc_msgSend`函数调用，会优先查找宿主类中靠前的元素，找到同名方法就进行调用，所以优先调用最后编译的分类的方法。


![](https://user-gold-cdn.xitu.io/2020/3/21/170f909c98450fca?w=1480&h=1850&f=png&s=368924)

![](https://user-gold-cdn.xitu.io/2020/3/21/170f91882d4338ed?w=1528&h=1670&f=png&s=352411)



从`call_load_methods`函数中我们可以得知：先调用类的`+load`方法，再调用分类的`+load`方法。

那么在类或者分类中`+load`方法的调用顺序是怎么样的呢？

从`call_class_loads`和`call_category_loads`函数中可以得知：可加载的类和分类分别保存在`loadable_classes`和`loadable_categories`数组中，因此我们只需要搞明白这两个数组中的类和分类的存放顺序，就可以知道调用顺序。

**prepare_load_methods**

按编译顺序将可加载的类添加进`loadable_classes`数组，先添加父类，再添加子类；<br>按编译顺序将可加载的分类添加进`loadable_categories`数组。所以：
* 按编译顺序调用类的`+load`方法，先编译先调用，调用子类的`+load `之前会先调用父类的`+load `；
* 按编译顺序调用分类的`+load`方法，先编译先调用。


![](https://user-gold-cdn.xitu.io/2020/3/21/170f93d83428686a?w=1564&h=2138&f=png&s=446947)

## 2. initialize
### 2.1 initialize 方法的调用
* ① **调用时刻：**
    * `+initialize`方法会在`类`第一次接收到消息时调用。
    * 如果子类没有实现`+initialize`方法，会调用父类的`+initialize`，所以父类的`+initialize`方法可能会被调用多次，但不代表父类初始化多次，每个类只会初始化一次。
* ② **调用方式：** 消息机制`objc_msgSend`函数调用。
* ③ **调用顺序：** 先调用父类的`+initialize`，再调用子类的`+initialize`（先初识化父类，再初始化子类）


### 2.2 场景分析
Person 、Person 的两个分类 Person (Test)、Person (Test2) 以及它的子类 Student 都实现了`+initialize`方法，且 Person (Test2)  最后编译。给 Student 类发送一条消息，查看打印结果。
```objc
// Person
#import "Person.h"
@implementation Person
+ (void)initialize {
    NSLog(@"person +initialize");
}
@end

// Person (Test)
...
// Person (Test2)
...
// Student
...

#import "ViewController.h"
#import "Student.h"
- (void)viewDidLoad {
    [super viewDidLoad];

    [Student alloc];
}

/*
2020-02-19 20:48:59.230852+0800 Category [34163:7694237] Person (Test2) +initialize
2020-02-19 20:48:59.231052+0800 Category [34163:7694237] Student +initialize
*/
```
从以上打印结果可以得出结论：
* ① `+initialize`方法会在`类`第一次接收到消息时调用；
* ② 先调用父类的`+initialize`，再调用子类的`+initialize`；
* ③ 调用了分类的`+initialize`，没有调用宿主类的`+initialize`。说明`+initialize`方法的调用方式为消息机制，而非像`+load`那样直接通过函数地址调用。

### 2.3 源码分析

OC 中的方法调用（也称消息发送），其实都是转换为`objc_msgSend()`函数的调用。`+initialize`方法会在`类`第一次接收到消息时调用，说明在`objc_msgSend()`函数内部会判断是不是第一次发送消息，是的话就调用`+initialize`方法。

关于`objc_msgSend()`函数的具体调用流程可以查看我的文章 [深入浅出 Runtime（三）：objc_msgSend方法调用流程](https://www.jianshu.com/p/a5d818d90a6e)。

下面我们通过`Runtime`源码`objc4-750`版本的来分析`+initialize`方法的调用流程。（我查看了最新版本的`+initialize`的函数调用栈与旧版本有差异，不过原理相同）

**函数调用栈：**
* objc-msg-arm64.s
    <br>① _objc_msgSend
* objc-runtime-new.mm
    <br>② class_getInstanceMethod：调用方法之前需要先获取方法
    <br>③ lookUpImpOrNil
    <br>④ lookUpImpOrForward
    <br>⑤ _class_initialize：初始化类的函数
    <br>⑥ callInitialize
    <br>⑦ objc_msgSend(cls, SEL_initialize)：给 cls 对象发送 initialize 消息

**lookUpImpOrForward**

![](https://user-gold-cdn.xitu.io/2020/3/21/170f8fdf9954b052?w=2048&h=1274&f=png&s=345885)

**_class_initialize**

![](https://user-gold-cdn.xitu.io/2020/3/21/170f8ff60b5ca1a0?w=1632&h=1346&f=png&s=237353)

**callInitialize**

调用`objc_msgSend`函数，给 cls 对象发送 initialize 消息。

![](https://user-gold-cdn.xitu.io/2020/3/21/170f900375435587?w=1328&h=554&f=png&s=111140)


## 3. load 和 initialize 的区别

区别|load|initialize
--|--|--
调用时刻|在`Runtime`加载类、分类时调用<br>（不管有没有用到这些类，在程序运行起来的时候都会加载进内存，并调用`+load`方法）。<br><br>每个类、分类的`+load`，在程序运行过程中只调用一次（除非开发者手动调用）。|在`类`第一次接收到消息时调用。<br><br>如果子类没有实现`+initialize`方法，会调用父类的`+initialize`，所以父类的`+initialize`方法可能会被调用多次，但不代表父类初始化多次，每个类只会初始化一次。
调用方式|① 系统自动调用`+load `方式为直接通过函数地址调用；<br>② 开发者手动调用`+load `方式为消息机制`objc_msgSend`函数调用。|消息机制`objc_msgSend`函数调用。
调用顺序|① 先调用类的`+load `，按照编译先后顺序调用（先编译，先调用），调用子类的`+load `之前会先调用父类的`+load `；<br>② 再调用分类的`+load `，按照编译先后顺序调用（先编译，先调用）（注意：分类的其它方法是：后编译，优先调用）。|① 先调用父类的`+initialize`，<br>② 再调用子类的`+initialize`<br>（先初识化父类，再初始化子类）。


## 4. 相关面试题
#### Q：Category 中有 +load 方法吗？+load 方法是什么时候调用的？+load 方法能继承吗？
1. 分类中有`+load`方法；
2. `+load`方法在`Runtime`加载类、分类的时候调用；
3. `+load`方法可以继承，但是一般情况下不会手动去调用`+load`方法，都是让系统自动调用。


#### Q：手动调用 Student 类的 +load 方法，但是 Student 类没有实现该方法，为什么会去调用父类的 +load 方法，且是调用父类的分类的 +load 方法呢？
因为`+load`方法可以继承，`[Student load]`手动调用方式为是消息机制`objc_msgSend`函数的调用，会去类方法列表里找对应的方法，由于 Student 类没有实现，就会去父类的方法列表中查找，且优先调用分类的`+load`方法。而系统自动调用的`+load`方法是直接通过函数地址调用的。
