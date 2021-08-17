

# 系列文章
* [肝了两个月的《Effective Objective-C 2.0》52 个知识点总结](https://juejin.cn/post/6904439922490441742)
* [《Effective Objective-C 2.0》52 个知识点总结（上）](https://juejin.cn/post/6904440708006936590/)
* [《Effective Objective-C 2.0》52 个知识点总结（下）](https://juejin.cn/post/6904440732287762439/)

# 第一章：熟悉 Objective-C 
通论该语言的核心概念。

## 1. 了解 Objective-C 语言的起源
**要点**

* Objective-C 为 C 语言添加了面向对象特性，是其超集。Objective-C 使用动态绑定的消息结构，也就是说，在运行时才会检查对象类型。接收一条消息之后，究竟应执行何种代码，由运行期环境而非编译器来决定。
* 理解 C 语言的核心概念有助于写好 Objective-C 程序。尤其要掌握内存管理模型和指针。

**笔者**

* Objective-C 顾名思义，为面向对象的 C 语言，它是 C 的超集。OC 由 Smalltalk 演化而来，后者是消息型语言的鼻祖。OC 使用 “消息结构” 而非 “函数调用” ，它是一门动态运行时语言。
* 动态运行时语言与编译时语言的区别：
    * 使用消息结构的语言（动态运行时语言），其运行时所执行的代码由运行环境来决定；在运行时才会检查对象的类型，在运行时才会去查找所要执行的方法。【🚩 11】
    * 使用函数调用的语言（编译时语言），其运行时所执行的代码由编译器决定。
        <br>我们可以通过一个 🌰 例子来理解动态运行时语言：
        >对于 `NSString *string = [[NSMutableArray alloc]init]`; 
        >* 编译时：编译器进行类型检查的时候，由于给一个 `NSString` 类型的指针赋值的是一个 `NSMutableArray` 对象，所以编译器会给出类型不匹配的警告。但是编译器会将 `string` 当作 `NSString` 的实例，所以 `string` 对象调用 `NSString` 的方法，编译没有任何问题，而调用 `NSMutableArray` 的方法，编译会直接报错。
        >* 运行时：由于 `string` 实际上是指向一个 `NSMutableArray` 对象，`NSMutableArray` 对象没有 `stringByAppendingString:` 方法，所以导致 Crash：`unrecognized selector send to instance`。
        >
        >```objc
        > NSString *string = [[NSMutableArray alloc]init];  //⚠️  Incompatible pointer types initializing 'NSString *' with an expression of >type 'NSMutableArray *'
        >[string stringByAppendingString:@"abc"];
        >[string addObject:@"abc"];  //❌  No visible @interface For 'NSString' declares the selector 'addObject:'
        >```
* OC 的动态性都是由 “运行期组件” ，也就是 Runtime 库（[ObjC4](https://opensource.apple.com/tarballs/objc4/)）来实现的，使用 OC 的面向对象特性所需的全部数据结构以及函数都在 ObjC4 里面。
    <br>运行期组件本质上是一种与开发者所编写的代码相链接的动态库（dynamic library），其代码能把开发者所编写的所有程序粘合起来，所以只要更新运行期组件，就可以提升应用程序性能。
* 理解 C 语言的核心概念有助于写好 OC 程序，尤其要掌握内存模型和指针。
* 对于 `NSString *string = @"the string"` ，NSString 实例分配在堆中，string 指针变量分配在栈上并指向该实例。
    <br>分配在堆中的内存必须直接管理，而分配在栈上用于保存变量的内存则会在其栈帧弹出时自动清理。
* OC 运行期环境将 “用 malloc 及 free 来分配或释放对象所占内存” 等工作抽象为一套名叫 “引用计数” 的内存管理架构，但是需要注意的是真正实现分配和释放的操作是 malloc 和 free，而非 alloc 和 dealloc。<br>[Link：《iOS - 老生常谈内存管理（四）：内存管理方法源码分析》](https://juejin.im/post/6844904131719593998) 
* 与创建结构体相比，创建对象还需要额外开销，例如分配和释放对内存等。
    
## 2. 在类的头文件中尽量少引入其他头文件
**要点**
* 除非确有必要，否则不要引入头文件。一般来说，应在某个类的头文件中使用前向声明来提及别的类，并在实现文件中引入那些类的头文件。这样做可以尽量降低类之间的耦合（coupling）。
* 有时无法使用向前声明，比如要声明的某个类遵循一项协议。这种情况下，尽量把 “该类遵循某协议” 的这条声明移至 “class-continuation 分类” 中。如果不行的话，就把协议单独放在一个头文件中，然后将其引入。
**笔者**
* 在类的头文件中尽量少引入其他头文件，将引入头文件的时机尽量延后，只在确有需要时才引入，这样可以：
    1. 缩减编译时间，减少类的使用者所需引入的头文件数量<br>假如我们在 A.h 中 `#import` B.h，那么如果类 C 中 `#import` A.h 的话，就会把 B 也给导入进来，而 C 根本就没用到 B，所以这样只会增加编译时间。
    2. 避免因两个类在头文件中互相引用而导致其中一个类无法编译的问题<br>如果两个类在 .h 文件中互相 `#import` 了对方，虽然它不像 `#include` 会导致死循环，但是会让其中一个类无法被编译 `Cannot find interface declaration for 'Class'`。
    3. 降低彼此依赖程度，降低类之间的耦合，避免因依赖关系太复杂而导致维护困难的问题
* 使用 `@class` 前向声明某个类<br>将引入头文件的时机延后，在非必要的情况下，在 .h 文件中使用 `@class` 来前向声明某个类，然后在需要知道该类细节的 .m  中使用 `#import`。
* 使用 `@protocol` 前向声明某个协议<br>同 `@class`，如果在头文件中不需要知道协议的全部细节，比如声明的属性或者方法参数、返回值等遵从某个协议，那么就可以使用 `@protocol` 前向声明该协议。但如果是该类遵从某个协议，则必须 `#import` 协议所在的文件。
* 有时候必须要在 .h 中引入其他文件
    1. 继承，子类的 .h 中必须 `#import` 父类
    2. 遵从某个协议，使用 `@protocol` 只能告诉编译器有某个协议，而此时编译器却要知道该协议中的具体方法和属性。
        * 这是难免的，但我们最好将该协议单独放在一个头文件里。如果将协议定义在某个比较大的类 A 的头文件里，那么我们如果要引入该协议，就要引入 A 的全部内容，这样就增加了编译时间，甚至可能产生相互引用问题。
        * 但是 “委托协议” 就不用单独写一个头文件了，因为 “委托协议” 只有与 “委托类” 放在一起定义才有意义。这种情况下，代理类应该将 “该类遵从某协议” 的这条声明转移至类扩展中，这样的话只要在 .m 中引入包含委托协议的头文件即可，而不需要放在 .h 中。【🚩 27】
    3. 分类
    4. 枚举，用到定义在其他类里的枚举
* 每次在头文件中 `#import` 其他头文件之前，都想想是否有必要。
    
## 3. 多用字面量语法，少用与之等价的方法
**要点**
* 应该使用字面量语法来创建字符串、数值、数组、字典。与创建此类对象的常规方法相比，这么做更加简洁扼要。
* 应该通过取下标操作来访问数组下标或字典中的键所对应的元素。
* 用字面量语法创建数组或字典时，若值中有 nil，则会抛出异常。因此，务必确保值里不含 nil。

**笔者**
* 使用字面量语法（语法糖）来创建字符串、数值、数组、字典，可以缩减代码长度，使代码更整洁，同时提高易读性。
    ```objc
    NSString *someString = @"Effective Objective-C 2.0";
        
    NSNumber *someNumber = @1; // 只包含数值，没有多余的语法成分
        
    NSArray *animals = @[@"cat", @"dog", @"mouse", @"badger"];
    NSString *dog = animals[1]; // “取下标” 操作更为简洁，更易理解，与其他语言语法类似
        
    NSDictionary *personData = @{@"firstName" : @"Matt", 
                     @"lastName" : @"Galloway",
                              @"age" : @28};   // <Key>:<Value>，比使用方法创建的写法 <Value>, <key> 更易读
    NSString *lastName = personData[@"lastName"];
        
    NSMutableArray *mutableArray = animals.mutableCopy;
    NSMutableDictionary *mutableDictionary = personData.mutableCopy;
    mutableArray[1] = @"dog"; // 对于可变数组或字典，可以通过下标修改其中的元素值
    mutableDictionary[@"lastName"] = @"Galloway";
    ```
* 对于数组和字典来说，使用字面量语法更安全。如果数组中存在值为 nil 的对象，则会抛出异常。而如果使用 `initWithObjects:` 的话，数组元素个数会在你不知道的情况下减少，且难以排查该错误。
    ```objc
       id obj1 = [[NSObject alloc]init];
       id obj2 = nil;
    id obj3 = [[NSObject alloc]init];
    NSArray *array2 = [[NSArray alloc]initWithObjects:obj1, obj2, obj3, nil];
    // array2 = @[obj1]，而非 @[obj1, obj3]
    // initWithObjects: 方法会依次处理各个参数，直到发现 nil 为止，由于 obj2 为 nil，所以该方法会提前结束
    NSArray *array1 = @[obj1, obj2, obj3];
    // 该字面量方式创建数组，效果等同于是先创建了一个数组，然后把方括号内的所以对象都加到这个数组中。如果数值元素对象中有 nil 则会抛出异常。
    // Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '*** -[__NSPlaceholderArray initWithObjects:count:]: attempt to insert nil object from objects[1]'
    // 同样的，你使用 addObject: 往数组中添加 nil 也会抛出异常
    NSMutableArray *mArr = [NSMutableArray arrayWithObjects:obj1, obj2, obj3, nil];
    [mArr addObject:nil]; // 编译器已经给出了警告 ⚠️  Null passed to a callee that requires a non-null argument
    // Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '*** -[__NSArrayM insertObject:atIndex:]: object cannot be nil'
    ```
* 局限性
    <br>如果是自定义的 NSNumber、NSArray、NSDictionary 这些类的子类，则无法使用字面量语法创建其实例。然而，由于 NSNumber、NSArray、NSDictionary 都是业已定型的 “子族”【🚩 9】 ，所以很少有人会去从中定义子类。自定义的 NSString 的子类支持字面量语法，但要修改编译器的选项才行。
* 使用字面量语法创建出来的都是不可变对象，若想要可变对象则需要调用 `mutableCopy` 方法。这样做会多调用一个方法，而且还多创建了一个对象（如果使用方法的话就可以直接创建一个可变对象），不过使用字面量语法所带来的好处是多于这个缺点的。
    ```objc
    NSMutableArray *mutableArray = [@[@1, @2, @3] mutableCopy];
    ```

## 4. 多用类型常量，少用 #define 预处理指令
**要点**
* 不要用预处理指令定义常量。这样定义出来的常量不含类型信息，编译器只是会在编译前据此执行查找与替换操作。即使有人重新定义了常量值，编译器也不会产生警告信息，这将导致应用程序中的常量值不一致。
* 在实现文件中使用 `static const` 来定义 “只在编译单元内可见的常量”（translation-unit-specific constant）。由于此类常量不在全局符号表中，所以无须为其名称加前缀。
* 在头文件中使用 `extern` 来声明全局常量，并在相关实现文件中定义其值。这种常量要出现在全局符号表中，所以其名称应加以区隔，通常用与之相关的类名做前缀。

**笔者**
* 定义常量使用类型常量，不建议使用预处理指令。
* 预处理指令和类型常量的区别：
    |   | 预处理指令  | 类型常量  |
    | ------------ | ------------ | ------------ |
    |   | 简单的文本替换  |   |
    |   | 不包括类型信息  | 包括类型信息（可以清楚描述常量的含义，有助于编写开发文档）  |
    |   | 可被任意修改  | 不可被修改  |
    |   |   | 可以设置其使用范围  |
    | 编译时刻  | 预编译  | 编译  |
    | 编译检查  | 没有 | 有  |
* 预处理指令
    ```objc
    #define ANIMATION_DURATION 0.3
    ```
    1. 预处理指令会把源代码中的 ANIMATION_DURATION 都替换为 0.3。
    2. 该常量没有类型信息，它应该为 NSTimeInterval 类型才合理。
    3. 假设此指令声明在某个头文件中，那么所有引入了这个头文件的代码，其 ANIMATION_DURATION 在编译时都会被替换为 0.3。
* 类型常量
    ```objc
    static const NSTimeInterval kAnimationDuration = 0.3;
    ```
      1. 该常量 kAnimationDuration 包含类型信息，其为 NSTimeInterval 类型（包括类型信息）。
    2. static 修饰符意味着该常量只在定义它的 .m 中可见（设置了其使用范围）。
    3. const 修饰符意味着该常量不可修改（不可修改）。
* 类型常量命名法
       1. 如果常量局限于某 “编译单元”（也就是 .m 中），则命名前面加字母 `k`，比如 `kAnimationDuration`。
    2. 如果常量在类之外可见，定义成了全局常量，则通常以 `类名` 作为前缀，比如 `EOCViewClassAnimationDuration`。
* 局部类型常量
    ```objc
    static const NSTimeInterval kAnimationDuration = 0.3;
    ```
    1. 一定要同时使用 static 和 const 来声明。这样编译器就不会创建符号，而是像预处理指令一样，进行值替换。
    2. 如果试图修改由 const 修饰的变量，编译器就会报错。
    3. 如果不加 static，则编译器就会它创建一个 “外部符号 symbol”。此时如果另一个编译单元中也声明了同名变量，那么编译器就会抛出 “重复定义符号” 的错误：
    ```objc
    duplicate symbol _kAnimationDuration in:
        EOCAnimatedView.o
        EOCOtherView.o
    ```
    4. 局部类型常量不放在 “全局符号表” 中，所以无须用类名作为前缀。
* 全局类型常量
    ```objc
    // In the header file
    extern NSString *const EOCStringConstant;
        
    // In the implementation file
    NSString *const EOCStringConstant = @"VALUE";
    ```
    1. 此类常量会被放在 “全局符号表” 中，这样才可以在定义该常量的编译单元之外使用。
    2. const 位置不同则常量类型不同，以上为，定义一个指针常量 EOCStringConstant，指向 NSString 对象。也就是说，EOCStringConstant 不会再指向另一个 NSString 对象。
    3. extern 是告诉编译器，在 “全局符号表” 中将会有一个名叫 EOCStringConstant 的符号。这样编译器就允许代码使用该常量。因为它知道，当链接成二进制文件后，肯定能找到这个常量。
    4. 必须要定义，而且只能定义一次，通常定义在声明该常量的 .h 的对应的 .m 中。
    5. 在实现文件生成目标文件时（编译器每收到一个 “编译单元” .m，就会输出一份 “目标文件” ），编译器会在 “数据段” 为字符串分配存储空间。链接器会把此目标文件与其他目标文件相链接，以生成最终的二进制文件。凡是用到 EOCStringConstant 这个全局符号的地方，链接器都能将其解析。
    6. 因为符号要放在全局符号表里，所以常量命名需谨慎，为避免名称冲突，一般以类名作为前缀。
    
## 5. 用枚举表示状态、选项、状态码
**要点**
* 应该用枚举来表示状态机的状态、传递给方法的选项以及状态码等值，给这些值起个易懂的名字。
* 如果把传递给某个方法的选项表示为枚举类型，而多个选项又可同时使用，那么就将各选项值定义为 2 的幂次，以便通过按位或操作将其组合起来。
* 用 `NS_ENUM` 与 `NS_OPTIONS` 宏来定义枚举类型，并指明其底层数据类型。这样做可以确保枚举是用开发者所选的底层数据类型实现出来的，而不会采用编译器所选的类型。
* 在处理枚举类型的 switch 语句中不要实现 default 分支。这样的话，加入新枚举之后，编译器就会提示开发者：switch 语句并未处理所有枚举。
**笔者**
* 应该用枚举表示状态、选项、状态码等值，并给这些值取个易懂的名字。
* 原始写法
    ```objc
    enum EOCConnectionState {
        EOCConnectionStateDisconnected,
        EOCConnectionStateConnecting,
        EOCConnectionStateConnected,
    };
    // 声明变量
    enum EOCConnectionState state = EOCConnectionStateDisconnected;
    ```
* 定义类型别名
    ```objc
    // typedef 定义一个类型别名
    typedef enum EOCConnectionState EOCConnectionState;
    // 声明变量
    EOCConnectionState state = EOCConnectionStateDisconnected;
    ```
* 可以不使用编译器所分配的序号，而是手动指定，接下去的枚举的值都会在你指定的值的基础上递增 1
    ```objc
    enum EOCConnectionState  {
        EOCConnectionStateDisconnected = 1,
        EOCConnectionStateConnecting,
        EOCConnectionStateConnected,
    };
    ```
* 指定枚举的底层数据类型
    * 在以前，实现枚举所用的数据类型取决于编译器，不过其二进制位的个数必须能完全表示下枚举编号才行。比如以上枚举的最大编号为 3，所以使用 1 个字节的 char 类型即可。（一个字节 8 位，可以表示 2^8=256 种枚举）<br>缺点：无法前向声明枚举变量，因为（声明的时候）编译器不清楚底层数据类型的大小（定义的时候才确定），所以在用到此枚举类型时，也就不知道究竟该给变量分配多少空间。
    * C++11 标准修订，可以指明用何种 “底层数据类型” 来保存枚举类型的变量。这样就可以前向声明枚举变量了，其写法为：
        ```objc
        enum EOCConnectionState ：NSInteger {
            EOCConnectionStateDisconnected,
            EOCConnectionStateConnecting,
            EOCConnectionStateConnected,
        };
        //前向声明
        enum EOCConnectionState ：NSInteger;
        ```
* 用枚举定义选项
    <br>各选项之间可以通过 “按位或” 运算来组合
    ```objc
    enum UIViewAutoresizing  {
        UIViewAutoresizingNone                  = 0,
        UIViewAutoresizingFlexibleLeftMargin,   = 1 << 0,
        UIViewAutoresizingFlexibleWidth,        = 1 << 1,
        UIViewAutoresizingFlexibleRightMargin,  = 1 << 2,
    };
    typedef enum UIViewAutoresizing UIViewAutoresizing;
    
    UIViewAutoresizing resizing = UIViewAutoresizingFlexibleLeftMargin | UIViewAutoresizingFlexibleWidth;
    if (resizing & UIViewAutoresizingFlexibleLeftMargin) {
         ......
    }
    ```
* Foundation 中定义了关于枚举的辅助宏
    1. 用这些宏可以指定用于保存枚举值的底层数据类型。
    2. 这些宏还具备向后兼容能力 —— 如果目标平台的编译器支持新标准，就用新式语法，否则用旧式。
    3. 这些宏是用 #define 预编译指令来定义的，其中一个用于定义 “状态” 枚举类型，另一个用于定义 “选项” 枚举类型。
    ```objc
    typedef NS_ENUM(NSUInteger, EOCConnectionState) {
        EOCConnectionStateDisconnected,
        EOCConnectionStateConnecting,
        EOCConnectionStateConnected,
    };
        
    typedef NS_OPTIONS(NSUInteger, EOCPermittedDirection) {
        EOCPermittedDirectionUp    = 1 << 0,
        EOCPermittedDirectionDown  = 1 << 1,
        EOCPermittedDirectionLeft  = 1 << 2,
        EOCPermittedDirectionRight = 1 << 3,
    };
    ```
* NS_ENUM、NS_OPTIONS 宏定义（这里就不贴出来了）
* 应该用 NS_OPTIONS 来定义 “选项” 类型枚举
    1. 提高通用性，免去类型强转操作
        >NS_OPTIONS 在 C++ 编译模式与非 C++ 模式下不同
        >* 在非 C++ 模式下，其（展开方式）和 NS_ENUM 相同
        >* 在 C++ 模式下，如果还按 NS_ENUM 来的话就会编译错误。原因是，比如以下代码，C++ 认为按位或运算结果的数据类型应该是枚举的底层数据类型即 NSUInteger，而且 C++ 不允许将这个底层类型 “隐式转换” 枚举类型本身 EOCPermittedDirection，所以以下代码在 C++ 模式或者 Objective-C++模式下就会编译错误（error：cannot initialize a variable of type ‘EOCPermittedDirection’ with an rvalue of type ‘int’），需要显示转换一下。
        >```objc
        >EOCPermittedDirection permittedDirections = EOCPermittedDirectionLeft｜EOCPermittedDirectionRight；
        >```
        >所以在 C++ 模式下，应该用 NS_OPTIONS 来定义 “选项” 类型枚举，因为它在 C++ 和非 C++ 模式下的 #define 是不一样的。
    2. 提高可读性
* 枚举在 switch 里的用法规范
    <br>在处理枚举类型的 switch 语句中不要实现 default 分支。这样的话，如果我们新加了一种枚举类型，那么编译器就会给我们警告，提示新加的枚举类型没在 switch 中处理。而如果加了 default 分支就不会给警告了。这就不能确保 switch 语句能正确处理所有的枚举类型。
* 用 `NS_ENUM` 和 `NS_OPTIONS` 来定义枚举类型，并指明其底层数据类型。这样可以确保枚举是用开发者所选的底层数据类型实现出来的，而不会采用编译器所选的类型。

# 第二章：对象、消息、运行期
对象之间能够关联与交互，这是面向对象语言的重要特征。本章讲述这些特征，并深入研究代码在运行期的行为。

## 6. 理解 “属性” 这一概念
**要点**
* 可以用 @property 语法来定义对象中所封装的数据。
* 通过 “特质” 来指定存储数据所需的正确语义。
* 在设置属性所对应的实例变量时，一定要遵从该属性所声明的语义。
* 开发 iOS 程序时应该使用 nonatomic 属性，因为 atomic 属性会严重影响性能。
**笔者**
* 属性用于封装对象中数据。可以用 `@property` 语法来定义属性。
* `@synthesize` 和 `@dynamic`
    1. 可以通过 `@synthesize` 来指定实例变量名字，如果你不喜欢默认的以下划线开头来命名实例变量的话。但最好还是用默认的，否则别人可能看不懂。
        <br>如果不想令编译器合成存取方法，则可以自己实现。如果你只实现了其中一个存取方法 setter or getter，那么另一个还是会由编译器来合成。但是需要注意的是，如果你实现了属性所需的全部方法（如果属性是 readwrite 则需实现 setter and getter，如果是 readonly 则只需实现 getter 方法），那么编译器就不会自动进行 `@synthesize`，这时候就不会生成该属性的实例变量，需要根据实际情况自己手动 `@synthesize` 一下。
        ```objc
        @synthesize name = _myName;
        ```
    2. `@dynamic` 会告诉编译器：不要自动创建实现属性所用的实例变量，也不要为其创建存取方法，即告诉编译器你要自己做这些事。当使用了 `@dynamic`，即使你没有为其实现存取方法，编译器也不会报错，因为你已经告诉它你要自己来做。
* 通过 “特质” 来指定存储数据所需的正确语义
    <br>属性关键字的类型有：原子性、读写权限、内存管理语义、方法名以及 Xcode 6.3 引入的可空性。
    <br>[Link：《OC - 属性关键字和所有权修饰符》](https://juejin.im/post/6844904067425124366)
* 在设置属性所对应的实例变量时，一定要遵从该属性所声明的语义
    >注意：尽量不要在初始化方法和 dealloc 方法中调用存取方法。【🚩 7】
    ```objc
    @property (nonatomic, copy) NSString *name;
    
    — (instancetype) initWithName:(NSString *)name {
        if(self = [super init]) {
            _name = [name copy];
        }
           return self;
    }
    ```
    若是自己来实现存取方法，也应该保证其具备相关属性所声明的性质。
* 开发 iOS 程序时，应使用 `nonatomic` 而非 `atomic`，因为 `atomic` 会严重影响性能。
    
## 7. 在对象内部尽量直接访问实例变量
**要点**
* 在对象内部读取数据时，应该直接通过实例变量来读，而写入数据时，则应通过属性来写。
* 在初始化方法及 dealloc 方法中，总是应该直接通过实例变量来读写数据。
* 有时会使用惰性初始化技术配置某份数据，这种情况下，需要通过属性来读取数据。
**笔者**
* 直接访问实例变量和通过属性访问的区别：
       1. 直接访问实例变量的速度比较快，编译器所生成的代码会直接访问保存对象实例变量的那块内存，而不是通过调用存取方法来访问。
    2. 直接访问实例变量，不会调用 setter 方法，这就绕过了为相关属性所定义的 “内存管理语义”。比如一个声明为 copy 的属性，访问属性进行赋值的话，会 copy 新对象并 release 旧对象。而直接访问实例变量则是 retain 新对象并 release 旧对象。
    3. 直接访问实例变量，就不会触发 KVO，因为 KVO 是通过在运行时生成派生类并重写 setter 方法以达到通知所有观察者的目的。这样做是否会产生问题，取决于具体的对象行为。
    4. 通过属性方法有助于排查与之相关的错误，因为可以在 setter 和 getter 方法中设置断点来调试。
* 在初始化方法和 dealloc 方法中不建议使用存取方法，应该直接访问实例变量。
    * 为什么在初始化方法中不建议使用存取方法？<br>因为子类可能会重写 setter 方法从而可能导致一些问题的发生。
        >假设 EOCPerson 类有一个子类叫做 EOCSmithPerson，这个子类专门表示那些姓 “Smith” 的人。该子类可能会覆写 lastName 属性所对应的设置方法：
        >```objc
        >- (void)setLastName:(NSString *)lastName {
        >     if (![lastName isEqualToString:@"Smith"]) {
        >         [NSException raise:NSInvalidArgumentException format:@"Last name must be Smith"];
        >     }
        >     self.lastName = lastName;
        >}
        >```
        >在父类 EOCPerson 的默认初始化方法中，可能会将姓氏设为空字符串 `self.lastName = @""`。此时若是通过 setter 方法来做，那么调用的将会是子类的 setter 方法，导致抛出异常。<br><br>
        >为什么会是调用子类的 setter 方法？因为在子类中调用 [super init] 先初始化父类的东西，根据 super 的原理，是从父类开始查找方法的实现，而消息接收者还是子类。也就是说在父类的 init 方法中调用 [self setLastName] 中 self 是子类对象，而又由于子类重写了该方法，故调用子类的。<br>
        >[Link：《深入浅出 Runtime（四）：super 的本质》](https://juejin.cn/post/6844904072252751880)
    * 但是有些情况下必须在初始化方法中调用存取方法：
        1. 待初始化的实例变量声明在父类，此时我们就无法在子类中直接访问此实例变量，所有就只能调用 setter 方法。
            >那么问题来了，既然子类不能直接访问声明在父类中的实例变量，那上述问题怎么避免呢？<br>
            >父类的初始化方法中直接访问实例变量，子类的初始化方法中通过 setter 方法访问就好了呀。
        2. 属性是懒加载的，就必须通过 getter 方法来访问，否则实例变量就永远不会初始化。
* 两种访问方案各有优劣，折中的方案为：<br>在对象内部读取数据时，应该直接通过实例变量来读，而写入数据时，则应该通过属性来写。这样既能提高读取操作的速度，又能控制对属性的写入操作。因为写入数据使用实例变量的话，就会绕过为相关属性所定义的 “内存管理语义”。如果我们直接通过访问实例变量来写入数据，则应该根据它的内存管理语义来设置。比如声明为 copy 的 name 属性：`_name = [name copy]`。

## 8. 理解 “对象等同性” 这一概念

**要点**
* 若想检测对象的等同性，请提供 “`isEqual:`” 与 “`hash`” 方法。
* 相同的对象必须具有相同的哈希码，但是两个哈希码相同的对象却未必相同。
* 不要盲目地检查每条属性，而是应该依照具体需求来制定检测方案。
* 编写 hash 方法时，应该使用计算速度快而且哈希码碰撞几率低的算法。
**笔者**
* `==` 操作符比较的是两个指针本身，而不是其所指的对象。所以有时候这结果不是我们想要的。<br>应该使用 NSObject 协议中声明的 `isEqual:` 方法来判断两个对象的等同性。某些对象提供了特殊的“等同性判定方法”，比如 NSString 的 `isEqualToString:`、NSArray 的 `isEqualToArray:`、NSDictionary 的 `isEqualToDictionary:`。
    * `isEqualToString:` 参数必须是 NSString 类型，否则结果未定义，但不会抛出异常。
    * `isEqualToArray:` 和 `isEqualToDictionary:` 的参数也必须是 NSArray 和 NSDictionary 类型，否则会抛出异常 `reason: '*** -[NSArray isEqualToArray:]: array argument is not an NSArray'`。
    我们也可以根据自己类的需求，编写自己的 “等同性判定方法”。
* 我们来看一段代码
    ```objc
    NSString *foo = @"Badger 123";
    NSString *bar = [NSString stringWithFormat:@"Badger %i", 123];
    BOOL equalA = (foo == bar);               // equalA = NO
    BOOL equalB = [foo isEqual:bar];          // equalB = YES
    BOOL equalC = [foo isEqualToString:bar];  // equalC = YES
    ```
    `==` 操作符判断 foo 和 bar 是两个不相等的字符串，因为它比较的是指针；
    <br>调用 `isEqualToString:` 方法比调用 `isEqual:` 方法要快，因为后者还要执行额外的步骤，它不知道受测(比较)对象的类型。而 isEqualToString: 知道消息接收者和参数都以 NSString 类型来比较。
* NSObject 协议中定义了两个 “等同性判定方法”
    ```objc
    -(BOOL)isEqual:(id)object;
    -(NSUInteger)hash;
    ```
    这两个方法的默认实现是：当且仅当其指针值完全相等时，这两个对象才相等。也就是说其默认实现相当于 `==` 操作符
* 如果想正确重写这两个方法，来实现自定义对象的 “等同性判定”，就必须遵守规则：
    1. 如果 `isEqual:` 方法判定两个对象相等，那么其 `hash` 方法也必须返回同一个值。
    2. 但是，如果两个对象的 `hash` 方法返回同一个值，其 `isEqual:` 方法未必会认为两者相等。
    举例如下：
    ```objc
    @interface Person : NSObject
    @property (nonatomic, copy) NSString *firstName;
    @property (nonatomic, copy) NSString *lastName;
    @property (nonatomic, assign) NSInteger age;
    @end

    @implementation Person

    - (BOOL)isEqual:(id)other {

        if (other == self) return YES;
        if ([self class] != [other class]) return NO; // 有时候我们可能认为，一个 Person 的实例可以与其子类的实例相等，所以要考虑这一种情况

        Person *otherPerson = (Person *)other;
        if (![_firstName isEqualToString:otherPerson.firstName]) return NO;
        if (![_lastName isEqualToString:otherPerson.lastName]) return NO;
        if (_age != otherPerson.age) return NO;

        return YES;
    }

    - (NSUInteger)hash {

        NSUInteger firstNameHash = [_firstName hash];
        NSUInteger lastNameHash  = [_lastName hash];
        NSUInteger ageHash = _age;
        return firstNameHash ^ lastNameHash ^ ageHash;
    }

    @end
    ```
* 重写的 hash 方法的效率问题
    <br>根据等同性约定：若两个对象相等，则其 hash 也相等，但是两个 hash 相等的对象却未必相等。所以以下这样重写 hash 完全可行：
    ```objc
    - (NSUInteger)hash {
        return 1337;
    }
    ```
    但是这么写的话，在 collection（Array、Dictionary、Set）中使用这种对象将产生性能问题，因为 collection 在检索哈希表时，会用对象的 hash 做索引。假如某个 collection 是用 set 实现的，那么 set 可能会根据 hash 把对象分装到不同的 “箱子数组” 中。在向 set 中添加新对象时，要根据其 hash 找到与之相关的那个数组，依次检查其中各个文件，看数组中已有的对象是否和将要添加的对象相等。如果相等，那就说明要添加的对象已经在 set 里面。由此可知，如果令每个对象都返回相同的 hash，那么在 set 中已有 1000000 个对象的情况下，若是继续向其中添加对象，则需将这 1000000 个对象全部扫面一遍，因为这些相同 hash 的对象都存在一个数组中。
    <br><br>以下做法可行，但是这样做还需负担创建字符串的开销，所以比返回单一值要慢。把这种对象添加进 collection 也会有性能问题，因为要想添加必须先计算其 hash。
    ```objc
    - (NSUInteger)hash {
        NSString *stringToHash = [NSString stringWithFormat:@"%@:%@:%i", _firstName, lastName, _age];
        return [stringToHash hash];
    }
    ```
    最佳做法如下。这种做法既能保持较高效率，又能使生成的 hash 至少位于一定范围之内，而不会过于频繁地重复。当然，此算法生成的 hash 还是会碰撞，不过至少可以保证 hash 有很多种可能的取值。在编写 hash 方法时，应该用当前的对象做做实验，以便在减少碰撞频度与降低运算复杂度之间取舍。
    ```objc
    - (NSUInteger)hash {
        NSUInteger firstNameHash = [_firstName hash];
        NSUInteger lastNameHash  = [_lastName hash];
        NSUInteger ageHash = _age;
        return firstNameHash ^ lastNameHash ^ ageHash;
    }
    ```
* 我们也可以写自己的 “等同性判定方法”。
    1. 好处：
        1. 无须检测参数类型，可以大大提升检测速度；
        2. 自己定义的方法名看上去可以更美观、更易读，就像 NSString 的 isEqualToString: 等一样。
    2. 在编写自己的“等同性判定方法”时，也应一并重写 isEqual: 方法。该方法实现通常如下：
        ```objc
        - (BOOL)isEqualToPerson:(Person *)otherPerson {
            if (self == object) return YES;

            if (![_firstName isEqualToString:otherPerson.firstName]) return NO;
            if (![_lastName isEqualToString:otherPerson.lastName]) return NO;
            if (_age != otherPerson.age) return NO;

            return YES;
        }

        - (BOOL)isEqual:(id)other {
            if ([self class] == [other class]) {
                return [self isEqualToPerson:(Person *)other]
            } else {
                return [super isEqual:other];
            }
        }
        ```
    3. 创建等同性判定方法时，需要决定是根据整个对象来判断等同性，还是仅根据其中几个字段来判断 —— 等同性判定的执行深度。
* NSArray 的 isEqualToArray: 的检测方式：
    1. 先看两个数组所含对象个数是否相同。
    2. 若相同，则在每个对应位置的两个对象身上调用其 isEqual: 方法。
    3. 如果对应位置上的对象均相等，那么这两个数组就相等。这叫做 “深度等同性判定”。
* 有时候也许我们只会根据标识符或者某些字段来判断等同性，所以是否需要在等同性判定方法中检测全部字段取决于受测对象。我们可以根据两个对象实例在何种情况下应判定为相等来编写等同性判定方法。
* 容器中可变类的等同性
    <br>在容器中放入某个对象的时候，就不应该再改变其 hash 值了，否则会有隐患。
    <br>因为 collection 会把各个对象按照其 hash 分装到不同的 “箱子数组” 中，如果某对象在放入箱子后 hash 又变了，那么其现在所处的箱子对它来说就是 “错误” 的。
    <br>所以，要确保添加到容器中对象的 hash 不是根据对象的 “可变部分” 计算出来的，或是保证之后不再改变对象内容。因此，我们最好不要往容器中添加 NSMutableArray 等可变对象，具体可以看书中给的例子。

## 9. 以 “类族模式” 隐藏实现细节

**要点**
* 类族模式可以把实现细节隐藏在一套简单的公共接口后面。
* 系统框架中经常使用类族。
* 从类族的公共抽象基类中继承子类要当心，若有开发文档，则应首先阅读。
**笔者**
* 类族是一种设计模式：定义一个抽象基类，使用 “类族” 把具体行为放在它的子类们中，将它们的实现细节隐藏在抽象基类后面，以保持接口简洁。
* 类族模式的意义在于：使用者无须关心创建出来的实例具体是什么类型，也不用考虑其实现细节，只需调用基类方法来创建即可。具体例子可以看书。
* UIButton 就使用了类族，buttonWithType: 方法返回的 button 类型取决于传入的 type。像这种 “工厂模式” 就是创建类族的办法之一。此外，NSArray、NSMutableArray 等大部分 collection 类都使用了类族模式。
* 如果对象所属类位于某个类族中，那么在查询其类型时就应该当心。应该使用 `isKindOfClass:` 而不是 `isMemberOfClass:` 或者 `[A class] == [B class]`。
* 由于 OC 是没办法指明某个基类是 “抽象” 的，所以应该在文档中写明类的用法，以告诉使用者这是类族的抽象基类，不能使用 init 等方法来实例化抽象基类，而是应该使用指定方法创建实例。我们也可以在基类的方法中抛出异常，来提示使用者这是类族的抽象基类，不过这种做法比较极端，很少使用。
* 向类族中新增实体子类的时候要当心。如果有开发文档，应该首先阅读，了解规则。<br>比如新增一个 NSArray 类族的子类，要遵守以下规则：
    1. 子类应该继承自类族中的抽象基类；
    2. 子类应该定义自己的数据存储方式；
    3. 子类应当覆写超类文档中指明需要覆写的方法 `count` 及 `objectAtIndex:`。

## 10. 在既有类中使用关联对象存放自定义数据
**要点**
* 可以通过 “关联对象” 机制来把两个对象连起来。
* 定义关联对象时可指定内存管理语义，用以模仿定义属性时所采用的 “拥有关系” 与 “非拥有关系”。
* 只有在其他做法不可行时才应选用关联对象，因为这种做法通常会引入难以查找的 bug。
**笔者**
* 当需要在对象中存放相关信息，而通过继承在子类的实例中去存储值这条路行不通的时候，可以通过创建对象所属类的分类，并在分类中通过关联对象来存储值。
* 关联对象相关 API：
    1. 以给定的键和策略为某对象设置关联对象值：
    ```objc
    void objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPolicy policy)
    ```
    2. 根据给定的键从某对象中获取相应的关联对象值：
    ```objc
    id objc_getAssociatedObject(id object, void *key)
    ```
    3. 移除指定对象的全部关联对象
    ```objc
    void objc_removeAssociatedObjects(id object)
    ```
    4. 关联策略由名为 objc_AssociationPolicy 的枚举所定义，其和属性关键字的对应关系如下：
    | 关联类型  | 等效的 @property 属性  |
    | ------------ | ------------ |
    | OBJC_ASSOCIATION_ASSIGN  | assign  |
    | OBJC_ASSOCIATION_RETAIN_NONATOMIC  | nonatomic, retain  |
    | OBJC_ASSOCIATION_COPY_NONATOMIC  | nonatomic, copy  |
    | OBJC_ASSOCIATION_RETAIN  |  retain  |
    | OBJC_ASSOCIATION_COPY  | copy  |
* 由于笔者之前写过一篇详细的 “关联对象” 文章，故这里就不做具体总结了。
    <br>感兴趣的可以参阅 [Link：《OC 底层探索 - Association 关联对象》](https://juejin.im/post/6844903972315070471)。
    
## 11. 理解 objc_msgSend 的作用
**要点**
* 消息由接收者、选择子及参数构成。给某对象 “发送消息” 也就相当于在该对象上 “调用方法”。
* 发给某对象的全部消息都要由 “动态消息派发系统” 来处理，该系统会查出对应的方法，并执行其代码。
**笔者**
* 静态绑定与动态绑定：
    * C 语言的函数调用方式是使用 “静态绑定”。在编译期就能决定运行时所应调用的函数。如果不考虑 “内联”，编译器在编译代码的时候就已经知道程序中哪些函数了，于是就会生成调用这些函数的指令，而函数地址实际上是硬编码在指令之中的。
    * OC 中，如果向某对象传递消息，那就会使用 “动态绑定” 机制来决定需要调用的方法，所要调用的方法在运行时才确定。
* 在 OC 中，给对象发送消息的语法为：
    ```objc
    id returnValue = [someObject messageName:parameter];
    ```
    在这里，someObject 叫做 “`接收者`”，messageName 叫做 “`选择子`”，选择子与参数合起来称为 “`消息`”。编译器看到此消息后，会将其转换为一条标准的 C 语言函数调用，所调用的函数为消息机制的核心函数 `objc_msgSend`：
    ```objc
    void objc_msgSend(id self, SEL _cmd, ...)
    ```
    该函数参数个数可变，能接受两个或两个以上参数。前面两个参数 “self 消息接收者” 和 “_cmd 选择子” 即为 OC 方法的两个隐式参数，后续参数就是消息中的那些参数（也就是方法显式参数）。<br>OC 中的方法调用在编译后会转换成该函数调用，比如以上方法调用后转换为：
    ```objc
    id returnValue = objc_msgSend(someObject, @selector(message:), parameter);
    ```
* 理解 objc_msgSend 函数执行流程
    <br>[Link：《深入浅出 Runtime（三）：消息机制》](https://juejin.im/post/6844904072235974663#heading-3)
* 调用的方法会被缓存，这样可以提高方法的查找速度
    <br>[Link：《深入浅出 Runtime（二）：数据结构》](https://juejin.im/post/6844904072215003143#heading-5)
* 其他 “边界情况（特殊情况）” 的消息交由其他函数处理，以下简单说明：
    * objc_msgSend_stret：待发送的消息返回的是结构体
    * objc_msgSend_fpret：待发送的消息返回的是浮点数
    * objc_msgSendSuper：给超类发消息，[super messege:parameter]
    [Link：《深入浅出 Runtime（四）：super 的本质》](https://juejin.cn/post/6844904072252751880)
* 理解 “尾调用优化” 技术。
    
## 12. 理解消息转发机制

**要点**
* 对象无法响应某个选择子，则进入消息转发流程。
* 通过运行期的动态方法解析功能，我们可以在需要用到这个方法时再将其加入类中。
* 对象可以把其无法解读的某些选择子转交给其他对象来处理。
* 经过上述两步之后，如果还是没办法处理选择子，那就启动完整的消息机制。
**笔者**
* 该章节详细讲诉了 OC 消息机制中的 “消息转发 ”阶段。当对象在 “消息发送” 阶段无法处理消息（找不到方法实现）时，就会进入 “消息转发” 阶段，开发者可以在此阶段处理未知消息。
* 消息机制可分为 “消息发送”、“动态方法解析”、“消息转发” 三大阶段。而该书作者将 “动态方法解析” 归并到了 “消息转发“ 阶段中。
* 消息转发分为两大阶段：
    * 动态方法解析：通过动态添加方法实现，来处理未知消息。
    * 真正的消息转发，此阶段又分为 Fast 和 Normal 两个阶段：
        * Fast：找一个备用接收者，尝试将未知消息转发给备用接收者去处理。
        * Normal：启动完整的消息转发，将消息有关的全部细节都封装到 NSInvocation 对象中，再给接收者最后一次机会去处理未知消息。
* 动态方法解析
    ```objc
    + (BOOL)resolveInstanceMethod:(SEL)selector;
    + (BOOL)resolveClassMethod:(SEL)selector
    ```
    我们可以在消息接收者类中，根据是实例方法还是类方法，实现以上方法来启用 “动态方法解析”，通过动态添加方法实现，来处理未知消息。方法参数即为未知消息的选择子，返回值表示这个类是否已经动态添加方法实现来处理（实际上该返回值只是用来判断是否打印信息，影响不大，不过还是要规范编写）。
    <br>“动态方法解析” 常用来实现 `@dynamic` 属性，在运行时动态添加属性 setter 和 getter 方法的实现。书中以一个完整的例子示范了 “如何以动态方法解析来实现 @dynamic 属性”，感兴趣的可以去看看。
* 消息转发
    * Fast - 找备用接收者：
    ```objc
    +/- (id)forwordingTargetForSelector:(SEL)selector;
    ```
    方法参数即为未知消息的选择子，返回值为备用接收者。
    <br>通过此方案，我们可以用 “组合” 来模拟出 “多重继承” 的某些特性。在一个对象内部，可能还有一系列其他对象，该对象可经由此方法将能够处理某选择子的相关内部对象返回，这样的话，在外界看来，好像是该对象亲自处理来这些消息似的。
    <br>请注意，在此阶段无法修改未知消息的内容，如果需要，请在 Normal 阶段去处理。
    * Normal - 完整的消息转发：
    ```objc
    +/- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
    ```
    通过实现以上方法返回一个适合该未知消息的方法签名，Runtime 会根据这个方法签名，创建一个封装了未知消息的全部内容（target、selector、argument）的 `NSInvocation` 对象。然后调用以下方法并将该 NSInvocation 对象作为参数传入。
    ```objc
    +/- (void)forwardInvocation:(NSInvocation *)invocation;
    ```
    在该方法中只需改变 target 即可，但一般不会这样做，因为这样做不如在 Fast 阶段就处理。比较有用的实现方式是：改变消息内容，比如改变选择子，追加一个参数等，再转发给其他对象处理。
* 实现以上方法时，不应由本类处理的未知消息，应该调用父类的实现，这样继承体系中的每个类都有机会处理未知消息，直至 NSObject。
* NSObject 的默认实现是最终调用 `doseNotRecognizeSelector` 方法，并抛出家喻户晓的异常 `unrecognized selector send to instance/class`，表明未知消息最终未能得到处理。
  ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b68c5b6548584fbfa79941d355475e09~tplv-k3u1fbpfcp-watermark.image)
* 以上几个阶段均有机会处理消息，但处理消息的时间越早，性能就越高。
    * 最好在 “动态方法解析” 阶段就处理完，这样 Runtime 就可以将此方法缓存，这样稍后这个实例再接收到同一消息时就无须再启动消息转发流程。
    * 如果在 “消息转发” 阶段只是单纯想将消息转发给备用接收者，那么最好在 Fast 阶段就完成。否则还得创建并处理 NSInvocation 对象。
* 关于方法缓存和消息机制的详细流程，感兴趣的可以阅读笔者的博客：
     <br>[Link：《深入浅出 Runtime（二）：数据结构》](https://juejin.im/post/6844904072215003143#heading-5)
    <br>[Link：《深入浅出 Runtime（三）：消息机制》](https://juejin.im/post/6844904072235974663#heading-3)
        
## 13. 用 “方法调配技术” 调试 “黑盒方法”

**要点**
* 在运行期，可以向类中新增或替换选择子所对应的方法实现。
* 使用另一份实现来替换原有的方法实现，这道工序叫做 “方法调配”，开发者常用此技术向原有实现中添加新功能。
* 一般来说，只有调试程序的时候才需要在运行期修改方法实现，这种做法不宜滥用。
**笔者**
* 该篇标题很新奇，其实就是利用 Runtime 动态交换方法实现（`method swizzling`）。
* 通过 `method swizzling`，我们既不需要修改源代码，也不需要通过继承子类来覆写方法就能改变这个类本身的功能。可以为那些 “完全不透明（完全不知道具体实现）” 的黑盒方法增加日志记录功能，非常有助于程序调试。但我们应该合理使用它，不能仅仅因为它是 OC 的一个特性就一定要用，若是滥用反而会令代码变得不易读懂且难于维护。
* 方法其实就是 `SEL` 到 `IMP` 的映射。我们调用方法，实际上就是根据方法 SEL 查找 IMP。而 IMP 就是指向方法实现的函数指针。
* 动态交换方法实现的步骤如下，别忘了 #import <objc/runtime.h> 哦：
    ```objc
    Method originalMethod = class_getInstanceMethod([NSString class], @selector(lowercaseString));
    Method swappedMethod  = class_getInstanceMethod([NSString class], @selector(uppercaseString));
    method_exchangeImplementations(originalMethod, swappedMethod);
    ```
    一般情况下，像以上这样直接交换两个方法实现的，意义不大。我们可以通过 `method swizzling` 来为既有的方法实现增添新功能。比方说在调用 lowercaseString 时记录某些信息，如下：
    ```objc
    Method originalMethod = class_getInstanceMethod([NSString class], @selector(lowercaseString));
    Method swappedMethod  = class_getInstanceMethod([NSString class], @selector(eoc_myLowercaseString));
    method_exchangeImplementations(originalMethod, swappedMethod);

    - (NSString *)eoc_myLowercaseString {
        NSString *lowercase = [self eoc_myLowercaseString];
        NSLog(@"%@ => %@", self, lowercase);
        return lowercase;
    }
    ```
    放心，以上代码不会递归调用死循环的，因为方法交换实现后，eoc_myLowercaseString 的 SEL 对应的是 lowercaseString 方法的 IMP。
        
## 14. 理解 “类对象” 的用意
**要点**
* 每个实例都有一个指向 Class 对象的指针，用以表明其类型，而这些 Class 对象则构成了类的继承体系。
* 如果对象类型无法在编译期确定，那么就应该使用类型信息查询方法来探知。
* 尽量使用类型信息查询方法来确定对象类型，而不要直接比较类对象，因为某些对象可能实现了消息转发功能。
**笔者**
* 关于 id 类型：
    * id 能指代任意的 OC 对象类型。一般情况下，应该指明消息接收者的具体类型，这样如果给该对象发送无法解读的消息，编译器就会给出警告。而 id 类型对象则不会，因为编译器假定它能响应所有消息。
    * id 其实就是指向 objc_object 结构体的一个指针。
        ```objc
        // A pointer to an instance of a class.
        typedef struct objc_object *id;
        ```
* 每一个 OC 对象的底层结构都为 `objc_object` 结构体。类和元类对象的底层结构都为 `objc_class` 结构体，其继承自 `objc_object`，它们之间通过 “`is a`” 指针联系。
    <br>关于 OC 对象底层数据结构，isa 指针等的详细讲解，可以参阅：
    <br>[Link：《深入浅出 Runtime（二）：数据结构》](https://juejin.im/post/6844904072215003143)
* super_class 指针确立了继承关系，而 isa 指针描述了实例所属的类。
* 编译器无法确定某类型对象能不能处理未知消息，因为运行期可以动态添加。但即便这样，编译器也觉得至少应该要有方法的声明，据此了解完整的 “方法签名(Type Encoding)”，并生成发送消息所需的正确代码。
* 类型信息查询方法 —— 在运行期检视对象类型。
    ```objc
    -/+ (BOOL)isMemberOfClass:(Class)cls;
    -/+ (BOOL)isKindOfClass:(Class)cls;
    ```
    `isMemberOfClass:` 方法是判断当前 instance/class 对象的 `isa` 指向是不是 class/meta-class 对象类型（也就是判断当前对象是否为某个类的实例）； 
    <br>`isKindOfClass:` 方法是判断当前 instance/class 对象的 `isa` 指向是不是 class/meta-class 对象或者它的子类类型（也就是判断当前对象是否为某个类或其子类的实例）。
    <br>以下这篇文章中详细讲解了一道关于这两个方法的面试题，感兴趣的可以看看。
    <br>[Link:《深入浅出 Runtime（五）：相关面试题》](https://juejin.im/post/6844904072428912653#heading-10)
* 由于 OC 是动态运行时语言，所以 “类型信息查询方法” 非常有用。从 collection (Array、Dictionary、Set) 中获取对象时，通常会查询类型信息，因为它们取出来时通常是 id 类型而不是 “强类型” 的。查询类型信息可以避免意外的调用了该类型对象响应不了的方法而导致 Crash，就比如：
    ```objc
    for (id object in array) {
        if ([object isKindOfClass:[NSString class]]) {
            NSString *string = (NSString *)object;
            NSString *uppercaseString  = [string uppercaseString];
            ...
        }
    }
    ```
    如果我们省去 “类型信息查询” 这一步，直接调用 `[object uppercaseString];`，那么如果该数组中意外的存储了一个非 NSString 实例，那么就会因响应不了未知消息（也就是找不到方法实现）而导致 Crash。
    <br>也可以通过比较类对象是否等同，直接使用 `==` 操作符就行，不必使用 `isEqual:` 方法。原因是，类对象是 “单例”，在应用程序范围内不存在同名的类。使用 isEqual: 方法只会产生不必要的性能开销。
    ```objc
    if ([object class] == [NSString class]) {
        ...
    }
    ```
* 在程序中尽量不要直接比较对象所属的类，而是调用 “类型信息查询方法” 来确定对象类型。因为后者可以正确处理那些使用了消息转发的对象。<br>比方说，某个对象可能会把它所接收的选择子都转发给另一个对象，这样的对象叫做代理 (proxy)，它们均以 NSProxy 为根类。<br>如果在此种代理对象上调用 class 方法，其返回的代理对象本身的类，而非接受的代理的对象所属的类。比如 HTProxy 继承于 NSProxy，其将消息都转发给一个叫 HTPerson 类的实例，现在有一个 HTProxy 的实例 proxy，那么：
    ```objc
    Class cls = [proxy class];
    ```
    cls 为 HTProxy 而非 HTPerson。
    如果改用类型信息查询方法，那么代理对象就会把这条消息转发给 “接受的代理的对象”。
    ```objc
    BOOL res = [proxy isKindOfClass:[HTPerson class]];
    ```
    res 值为 YES。
    <br>因此，以上两种方法所查询处理的对象类型不同。

# 第三章：接口与 API 设计
很少有那种写完就不再复用的代码。即使代码不向更多人公开，也依然有可能用在自己的多个项目中。本章讲解如何编写与 Objective-C 搭配得宜的类。

## 15. 用前缀避免命名空间冲突
**要点**
* 选择与你的公司、应用程序或二者皆有关联之名称作为类名的前缀，并在所有代码中均使用这一前缀。
* 若自己所开发的程序库中用到了第三方库，则应该为其中的名称加上前缀。
**笔者**
* 用前缀避免命名空间冲突。OC 没有其他语言那种内置的命名空间机制，所以我们在起名时要注意。如果发生命名冲突，那么应用程序的链接过程就会因为出现重复符号而出错。比如在工程中的两个文件中都实现了名为 Person 类，那么就会导致 Person 所对应的类符号和元类符号各定义了两次，从而导致编译错误：
    ```objc
    duplicate symbol '_OBJC_CLASS_$_Person' in:
        /Users/.../x86_64/Person.o
        /Users/.../x86_64/Person的副本.o
    duplicate symbol '_OBJC_METACLASS_$_Person' in:
        /Users/.../x86_64/Person.o
        /Users/.../x86_64/Person的副本.o
    ld: 2 duplicate symbols for architecture x86_64
    ```
    出现这种情况的原因大多是因为我们把两个含有重名类的程序库都引入到当前项目中。
    <br>比编译器无法链接更糟糕的是，在运行期载入含有重名类的程序库，从而导致程序 Crash。
* 如何命名？以何前缀命名？
    * 假如你所在的公司叫做 Effective Widgets，那么就可以在所有应用程序中都会用到的那部分代码中使用 EWS 作前缀。
    * 如果有些代码只用于名为 Effective Browser 的浏览器项目中，那就在这部分代码中使用 EWB 作前缀。
    * 需要注意的是，Apple 宣称其保留使用所有 “两字母前缀” 的权利，所以我们选用的前缀最好是三个字母的。
    假如你不遵守这条规则，比如，你使用 TW 这两个字母作为前缀，那么就会出现问题。iOS 5.0 SDK 发布时包含了 Twitter 框架，此框架就使用 TW 作前缀。这就很可能出现重复符号错误。
* 不仅是类名，应用程序中的所有名称都应加前缀
    * 分类以及分类中方法。【🚩 25】中解释这么做的原因。
    * 类实现文件中所用的纯 C 函数及全局变量。因为在编译好的目标文件中，这些名称是要算作 “顶级符号” 的。
* 若自己所开发的程序库中用到了第三方库，则应该为其中的名称加上前缀。
    * 如果你使用了第三方库（以手动导入管理而非 Cocoapods 管理），并准备将其再发布为程序库供他人开发使用时，尤其要注意重复符号问题。因为你的程序库所包含的那个三方库也许还会被他人的应用程序本身，或是其引入的其他三方库所引入，这样就很容易出现重复符号错误。
    * 例如，你准备发布的程序库叫做 EOCLibrary，其中引入了名为 XYZLibrary 的第三方库，那么就应该把 XYZLibrary 中的所有名字都冠以 EOC。

## 16. 提供 “全能初始化方法”
**要点**
* 在类中提供一个全能初始化方法，并于文档里指明。其他初始化方法均应调用此方法。
* 若全能初始化方法与超类不同，则需覆写超类中的对应方法。
* 如果超类的初始化方法不适用于子类，那么应该覆写这个超类方法，并在其中抛出异常。
**笔者**
* 类实例的初始化方法可能不止一种，我们要选定一个作为全能初始化方法，令其他初始化方法均调用此方法。这样当初始化操作有变化时，只需要改动全能初始化方法，无须改动其他初始化方法。
* 对 init 等初始化方法做些改动<br>比如我们指定矩形类 EOCRectangle 的初始化方法为：
    ```objc
    - (instancetype)initWithWidth:(float)width andHeight:(float)height {
        if (self = [super init]) {
            _width = width;
            _height = height;
        }
    }
    ```
    ```objc
    // 1.使用宏 NS_UNAVAILABLE 来禁用该初始化方法
    + (instancetype)new NS_UNAVAILABLE;
    - (instancetype)init NS_UNAVAILABLE;
    // 2.Using default values
    - (instancetype)init {
        return [self initWithWidth:5.0 andHeight:10.0];
    }
    // 3.Throwing an exception
    - (instancetype)init {
        @throw [NSException exceptionWithName:NSInternalInconsistencyException 
                       reason:@"Must use initWithWidth:andHeight: instead." userInfo:nil];
    }
    ```
* 正方形类 EOCSquare 继承于 EOCRectangle<br>指定它的初始化方法为
    ```objc
    - (instancetype)initWithDimension:(float)dimension {
        return [super initWithWidth:dimension andHeight:dimension];
    }
    ```
    然后调用者可能会使用  initWithWidth:andHeight: 或者 init 方法来初始化 EOCSquare 实例
    <br>所以当类继承时，如果子类的全能初始化方法与父类方法的名称不同，那么总应覆写父类的全能初始化方法，如下：
    ```objc
    - (instancetype)initWithWidth:(float)width andHeight:(float)height {
        float dimension = MAX(width, height);
        return [self initWithDimension:dimension];
    }
    ```
    注意此时不需要再重写 init 方法，因为调用到的父类 EOCRectangle 中的 init 方法中调用的是  initWithWidth:andHeight: 方法，而该方法已被子类重写，所以调用的是子类的实现。
    <br><br>如果超类的初始化方法不适用于子类：
    <br>有时候我们不想覆写父类的全能初始化方法，因为那样做没有道理。比如以 float dimension = MAX(width, height); 方式计算边长来初始化 EOCSquare 对象，我们认为这是方法调用者自己犯了错误。这时候我们就可以覆写父类的全能初始化方法（如 initWithWidth:andHeight:）并抛出异常。同时，覆写 init 方法，让其调用自己的初始化方法，如果不覆写的话它也会抛出异常，因为它调用了 initWithWidth:andHeight:。
    >不过，在 OC 程序中，只有当发生严重错误时，才应该抛出异常【🚩 21】，所以，初始化方法抛出异常乃是不得已之举，表明实例真的没办法初始化了。
    ```objc
    - (instancetype)init {
        return [self initWithDimension:5.0f];
    }
    ```
* 如果某对象的实例有两种完全不同的创建方式，必须分开处理，那么就需要编写多个全能初始化方法。比如：
    ```objc
    - (instancetype)initWithCoder:(NSCoder *)coder;
    ```
    该方法要通过 “解码器”（decoder）将对象数据解压缩，所以其实现不调用其他全能初始化方法。实现应该如下所示：
    ```objc
    // EOCRectangle
    - (instancetype)initWithCoder:(NSCoder *)coder {
        if (self = [super init]) {
            _width = [decoder decodeFloatForKey:@"width"];
            _height = [decoder decodeFloatForKey:@"height"];
        }
    }
    // EOCSquare
    - (instancetype)initWithCoder:(NSCoder *)coder {
        if (self = [super initWithCoder:coder]) {
            // EOCSquare's specific initializer
        }
    }
    ```
    每个子类的全能初始化方法都应该调用其父类的对应方法，并逐级向上，实现 initWithCoder: 时也要这样。如果不这么做，EOCSquare 的该方法没有调用父类的该同名方法，而是调用自身或是父类的其他全能初始化方法，那么父类的 initWithCoder: 方法就没机会实现，也就无法将 _width 和 _height 两个实例变量解码了。


## 17. 实现 description 方法
**要点**
* 实现 `description` 方法返回一个有意义的字符串，用以描述该实例。
* 若想在调试时打印出更详尽的对象描述信息，则应实现 `debugDescription` 方法。
**笔者**
* 我们使用 NSLog 打印对象，就会给对象发送 description 消息，该方法返回一个字符串，所以打印对象用 %@。
* 该方法定义在 NSObject 协议里，而不是 NSObject 类里，因为 NSObject 不是唯一的根类，NSProxy 也是遵从了 NSObject 协议的根类。
* 如果我们没有重写 description 方法，那么调用 NSObject 类中的该方法的默认实现为：返回类名和对象的内存地址。如下：
    ```objc
    id object = [NSObject new];
    NSLog(@"object = %@", object);
    // object = <NSObject: 0x600000724650>
    ```
* 如果我们想打印对象的详细信息，可以重写 description 方法并返回我们所需要的信息。比如：
    ```objc
    - (NSString *)description {
        return [NSString stringWithFormat:@"<%@: %p, \"%@ %@\">", [self class], self, _firstName, _lastName];
    }
    ```
    建议重写的 description 方法的实现中，也返回类名和对象的内存地址。
* 借助 NSDictionary 类的 description 方法来更好地打印对象属性信息：
    ```objc
    - (NSString *)description {
        return [NSString stringWithFormat:@"<%@: %p, %@>", [self class], self, @{@"title":_title, @"latitude":_latitude, @"longitude":_longitude}];
    }
    // location = <EOCLocation: 0x7f98f2e01d20>, {latitude = "51.506"; longitude = 0; title = London}>
    ```
    这样比直接拼接打印属性信息写法更易维护。如果以后新增属性并且要在 description 中打印，那么只需修改字典内容即可。
* 还有个 debugDescription 方法，它和 description 的区别在于：它只在调试时在控制台命令打印对象时才调用 (断点，然后使用 LLDB 命令 `po` (print-object))。
    * 在 NSObject 类中的 debugDescription 的默认实现是直接调用 description。
    * 若不想在代码打印对象时输出太详尽的对象描述信息，而是在调试时才要这么做，比如在调试时再打印 “类名和对象的内存地址”（NSArray 类就是这么做的）。那么就实现 debugDescription 方法，返回比 description 更详尽的信息。


## 18. 尽量使用不可变对象
**要点**
* 尽量创建不可变的对象。
* 若某属性仅可于对象内部修改，则在 “class-continuation 分类（类扩展）” 中将其由 readonly 属性扩展为 readwrite 属性。
* 不要把可变的 collection 作为属性公开，而应提供相关方法，以此修改对象中的可变 collection。
**笔者**
* 该篇主要讲述 `readonly` 和 `readwrite` 这两个 “读写权限” 属性关键字及其用法：
    * readwrite：默认，属性可读可写
    * readonly：属性只读
* 应该尽量把对外公布的属性设为只读，而且只在确有必要时才将属性对外公布。
    * 比方说一些模型对象通过初始化方法创建，其后无须改动其属性值（比如地图模型），那么属性对外应设为只读。这样只要使用方试着修改属性值就会编译错误，对象本身的数据结构也就不可能出现不一致的现象，比如地图模型的经纬度等数据不会发生变动。
    * 在【🚩 8】中所述，如果把可变对象放入 collection 后又修改其内容，那么很容易就会破坏 set 的内部数据结构，使其失去固有的意义。所以要尽量减少对象的可变内容。
* 把属性设为只读后，可以不指定其内存管理语义而采用默认，因为其没有 setter 方法，比如：
    ```objc    
    @property (nonatomic, readonly) NSString *title;
    ```
    但虽说如此，我们还是应该至少在文档里指明实现里所用的内存管理语义，这样以后把它变为可读写属性时就会简单一些。
* 有时我们想在对象内部修改属性，但是对外只读。这时候就可以在类扩展中将属性重新声明为 readwrite：
    ```objc
    // .h
    @property (nonatomic, copy, readonly) NSString *title;
    // .m
    @property (nonatomic, copy, readwrite) NSString *title;
    ```
    * 如果该属性是 nonatomic 的，那么这样做可能会产生 “竞争条件”。在对象内部写入某属性时，外部观察者也许正读取该属性。若想避免此问题，可以在必要时通过 GCD 【🚩 41】等手段，将（包括在对象内部的）所有数据的存取操作都设为同步操作。
    * 当属性是 collection 类型时，也建议对外只读并设为不可变属性（如 NSArray），而在对象内部则为可变属性（如 NSMutableArray），并对外供相关方法（插入、删除等）来供使用方操作对象中的可变 collection。
        <br>为什么不直接将可变 collection 直接暴露出去，而要多次一举呢？
        <br>因为有时候我们可能需要在插入或删除方法中做一些其他操作，而让外部直接操作可变 collection 是达不到这一目的的，因为其直接操作底层数据。
* 即便属性对外设置了 readonly，使用方也可以使用 KVC 来修改值，这样的话出现问题需使用方自己来承当后果。<br>更不合规范的是，使用方甚至可能会直接用类型信息查询功能查出属性所对应的实例变量在内存布局中的偏移量，以此来人为设置这个实例变量的值。
* 不要在返回的对象上查询类型，比如使用 isKindOfClass: 方法，来确定其是否可变。
    <br>开发者或许不建议使用方修改对象中的数据，但可能由于 collection 很大，copy 耗时，而直接将可变 collection 返回，但开发者对外声明为不可变 collection 了。我们不要假设其为可变 collection ，然后通过 isKindOfClass: 方法来确定其为可变 collection 后对它进行操作。

## 19. 使用清晰而协调的命名方式
**要点**
* 起名时应遵从标准的 Objective-C 命名规范，这样创建出来的接口更容易为开发者所理解。
* 方法名要言简意骇，从左至右读起来要像个日常用语中的句子才好。
* 方法名里面不要使用缩略后的类型名称。
* 给方法起名时的第一要务就是确保其风格与你自己的代码或所要集成的框架相符。
**笔者**
* OC 中的方法命名虽长，但其所要表达的意思清晰。
    <br>比如替换字符串的方法 `stringByReplaceOccurrencesOfString:withString:`， 
    <br>在其他语言中方法名仅是 `replace(@"A", @"B")`，不能清晰地表达出两个参数 A 和 B 到底是谁替换谁等等。
* 方法与变量名使用 “驼峰式大小写命名法” —— 以小写字母开头，其后每个单词首字母大写；<br>类名也用驼峰式命名法，不过其首字母大写，而且前面通常还有两三个大写的前缀字母。【🚩 15】
* 方法命名
    * 比如创建一个指定宽高的 Rectangle 实例
        <br>在 C++ 中写法可能如下。这样回顾代码时看不出来两个参数是矩形尺寸，或者哪个是宽哪个是高，需要查看函数定义才能确定。
        ```C++
        Rectangle *rectangle = new Rectangle(5.0f, 10.0f);
        ```
        在 OC 中可以这样定义初始化方法，虽然语法上没有问题，但其和 C++ 构造器存在同样的问题 —— 命名不清晰，调用的时候每个变量的含义不清楚。
        ```objc
        - (instancetype)initWithSize:(float)width :(float)height;
        ```
        ```objc
        EOCRectangle *rectangle = [[EOCRectangle alloc] initWithSize:5.0f :10.0f];
        ```
        正确写法如下：
        ```objc
        - (instancetype)initWithWidth:(float)width height:(float)height;
        ```
        ```objc
        EOCRectangle *rectangle = [[EOCRectangle alloc] initWithWidth:5.0f height:10.0f];
        ```
    * 方法命名要言简意骇，也不能太过长，清晰而不啰嗦，能准确表达方法所执行的任务即可。
    * 命名规则
        * 如果方法的返回值是新创建的，那么方法名的首个词应该是返回值的类型，除非前面还有修饰符，例如 localizedString。属性的存取方法不遵循这种命名方式，因为一般认为这些方法不会创建新对象，即便有时返回内部对象的一份拷贝，我们也认为那相当于原有的对象。这些存取方法应该按照其所对应的属性来命名。
        * 应该把表示参数类型的名词放在参数前面。
        * 如果方法要在当前对象上执行操作，那么就应该包含动词；若执行操作时还需要参数，则应该在动词后面加上一个或多个名词。
        * 不要使用 str 这种简称，应该用 string 这样的全称。
        * Boolean 属性应加 is 前缀。如果某方法返回非属性的 Boolean 值，那么应该根据其功能，选用 has 或 is 当前缀。
            ```objc
            @property (nonatomic, assign, getter = isEnabled) BOOL enabled;
            // NSString
            - (BOOL)isEqualToString:(NSString *)string;
            ```
        * 将 get 这个前缀留给那些借由 “输出参数” 来保存返回值的方法，比如说把返回值填充到 “C语言式数组”(C-style array) 里的那种方法就可以使用这个词做前缀。
            ```objc
            - (void)getCharacters:(unichar *)buffer range:(NSRange)aRange;
            ```
* 类与协议的命名
    * 应该为类和协议的名次加上前缀，以避免命名空间冲突。【🚩 15】
    * 命名同方法命名，使其从左至右读起来较为通顺。
    * 继承时要遵守其命名惯例，比如继承自 UIView 的子类，命名末尾必须是 view。
    * 自定义的委托协议，其名称中应该包含委托发起方的名称，后面再跟上 Delegate 一词，参照 UITableViewDeleagte。


## 20. 为私有方法名加前缀
**要点**
* 给私有方法的名称加上前缀，这样可以很容易地将其同公共方法区分开。
* 不要单用一个下划线做私有方法的前缀，因为这种做法是预留给苹果公司用的。
**笔者**
* 给私有方法的名称加上前缀的原因：
    1. 加个前缀便于和公共方法区分开，有助于调试。
    2. 便于修改方法名或方法签名。
        <br>修改公共方法的名称或签名之前要三思，因为公共 API 不便随意改动。如果改了的话，使用这个方法的所有开发者都必须更新其代码。而修改私有方法，则只需同时修改本类内部的相关代码即可，不会影响到公共 API。给私有方法加前缀就能很容易看出来哪些方法可以随意修改，哪些不应该轻易改动。
    3. OC 是动态运行时语言，它不像 C++ 和 Java 那样可以真正声明为私有方法，所以一般我们要在命名中体现出 “私有方法” 语义。
* 使用何种前缀可根据个人喜好，比如可以使用 “ p_ ” 作为前缀，p 表示 private。
* 某次修订后编译器已经不要求使用方法前必须先行声明，所以私有方法一般只在实现的时候声明。
* 苹果喜欢单用一个下划线作为私有方法的前缀，因此苹果在文档中说开发者不应该单用一个下划线做前缀。如果我们这么做了，就可能会在子类中无意覆写父类（苹果类）的同名私有方法，这样会导致本该调用父类的实现而现在却调用自己覆写的实现，从而引发问题。<br>例如，UIViewController 有一个名叫 _resetViewController 的私有方法。你可能会无意间覆写但是你根本不会察觉到，除非你深入研究过 UIViewController。导致的问题是：你的子类的这个方法会被频繁调用。
* 此外，你可能会继承来自三方框架的类，你不知道它们的私有方法以什么名称前缀，除非该框架在文档中明示或者你阅读了源码。同样的别人也可能从你写的类中继承子类。所以为了避免重名问题，可以把自己一贯使用的类名前缀用作私有方法的前缀。

## 21. 理解 Objective-C 错误模型
**要点**
* 只有发生了可使整个应用程序崩溃的严重错误时，才应使用异常。
* 在错误不那么严重的情况下，可以指派 “委托方法”（delegate method）来处理错误，也可以把错误信息放在 NSError 对象里，经由 “输出参数” 返回给调用者。
**笔者**
* 只有发生了可使整个应用程序崩溃的严重错误时，才应使用异常。<br>比如说，有人直接使用了抽象基类，在子类必须覆写的父类方法里抛出异常。
    ```objc
    @throw [NSException exceptionWithName:@"ExceptionName" reason:@"There was an error" userInfo:nil];
    ```
* 在错误不那么严重的情况下，OC 所用的编程范式为：令方法返回 nil/0，或是使用 NSError，以表明其中有错误发生。
    <br>比如说，如果初始化方法无法根据传入的参数来初始化当前实例，那么就可以令其返回 nil/0。
    <br>NSError 的用法更加灵活，因为经由此对象，我们可以把导致错误的原因回报给调用者。
* NSError 对象封装了三条信息：
    * Error domain（错误范围）：错误发生的范围，也就是产生错误的根源。通常定义成 NSString 类型的全局常量。<br>最好为你自己的库指定一个专用的 “错误范围” 字符串，这样使用方可以知道该错误在你的库中产生。
        ```objc
        // EOCErrors.h
        extern NSString *const EOCErrorDomain;
        // EOCErrors.m
        NSString *const EOCErrorDomain = @"EOCErrorDomain"; 
        ```
    * Error code（错误码）：用以指明在某个范围内具体发生了何种错误。通常定义成枚举类型。
        ```objc
        typedef NS_ENUM(NSUInteger, EOCError) {
            EOCErrorUnknown = –1,
            EOCErrorInternalInconsistency = 100,
            EOCErrorGeneralFault = 105,
            EOCErrorBadInput = 500,
        };
        ```
        用枚举不仅可以解释错误码的含义，而且还给它们起了个有意义的名字。还可以在定义这些枚举的头文件里对每个错误类型详加说明。
    * User info（用户信息）：有关此错误的额外信息。
* 创建 NSError 对象的方法：
    ```objc
    + (instancetype)errorWithDomain:(NSErrorDomain)domain code:(NSInteger)code userInfo:(nullable NSDictionary<NSErrorUserInfoKey, id> *)dict;
    ```
* NSError 常见用法：
    1. 通过委托协议来传递错误，有错误发生时，委托方会把错误信息经由协议方法传给 delegate 对象。<br>这比抛出异常好，因为调用方可以自己决定是否要实现该协议方法，是否要处理此错误。
    2. 经由方法的 “输出函数” 返回给调用者。如下。同样的，调用方如果不关心此错误，就可以给 error 参数传入 nil。
        ```objc
        NSError * __autoreleasing error = nil;
        BOOL ret = [object doSomething:&error];
        if (error) {
            // There was an error
        }
        ```

## 22. 理解 NSCopying 协议
**要点**
* 若想令自己所写的对象具有拷贝功能，则需实现 `NSCopying` 协议。
* 如果自定义的对象分为可变版本和不可变版本，那么就要同时实现 NSCopying 与 `NSMutableCopying` 协议。
* 复制对象时需决定采用浅拷贝还是深拷贝，一般情况下应该尽量执行浅拷贝。
* 如果你所写的对象需要深拷贝，那么可考虑新增一个专门执行深拷贝的方法。
**笔者**
* 拷贝的目的：
    * 产生一个副本对象，跟源对象互不影响；
    * 修改了源对象，不会影响副本对象；
    * 修改了副本对象，不会影响源对象。
* iOS 提供了 2 个拷贝方法：
    * copy：不可变拷贝，产生不可变副本；
    * mutableCopy：可变拷贝，产生可变副本。
* 深拷贝和浅拷贝：
    拷贝类型|拷贝方式|特点
    --|--|--
    深拷贝|内存拷贝，让目标对象指针和源对象指针指向 `两片` 内容相同的内存空间。|1. 不会增加被拷贝对象的引用计数；<br>2. 产生了一个内存分配，出现了两块内存。
    浅拷贝|指针拷贝，对内存地址的复制，让目标对象指针和源对象指针指向 `同一片` 内存空间。|1. 会增加被拷贝对象的引用计数；<br>2. 没有进行新的内存分配。<br>注意：如果是小对象如 NSString，可能通过 Tagged Pointer 来存储，没有引用计数。
    >简而言之：<br>1. 深拷贝：内容拷贝，产生新对象，不增加对象引用计数
    ><br>2. 浅拷贝：指针拷贝，不产生新对象，增加对象引用计数
    ><br>区别：1. 是否影响了引用计数；2. 是否开辟了新的内存空间
    ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3edcc7bd64a848ff9c9a8c718479cac8~tplv-k3u1fbpfcp-watermark.image)
* 对 mutable 对象与 immutable 对象 进行 copy 与 mutableCopy 的结果：
    源对象类型|拷贝方式|目标对象类型|拷贝类型（深/浅）
    :--:|:--:|:--:|:--:
    mutable 对象|copy|不可变|深拷贝
    mutable 对象|mutableCopy|可变|深拷贝
    immutable 对象|copy|不可变|`浅拷贝`
    immutable 对象|mutableCopy|可变|深拷贝
    >注：这里的 mutable 对象与 immutable 对象指的是系统类 NSArray、NSDictionary、NSSet、NSString、NSData 与它们的可变版本如 NSMutableArray 等。
* 以上对 collection 容器对象进行的深浅拷贝是指对容器对象本身的，对 collection 中的对象执行的默认都是浅拷贝。也就是说只拷贝容器对象本身，而不复制其中的数据。主要原因是，容器内的对象未必都能拷贝，而且调用者也未必想在拷贝容器时一并拷贝其中的每个对象。
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a62200d3246451080c30e1da14feeb7~tplv-k3u1fbpfcp-watermark.image)
* 如果想要实现对自定义对象的拷贝，需要遵守 `NSCopying` 协议，并实现 `copyWithZone:` 方法。
    >NSZone 是什么？可参阅：[Link：《iOS - 老生常谈内存管理（三）：ARC 面世》](https://juejin.im/post/6844904130431942670#heading-9)
    ><br>对于现在的运行时系统（编译器宏 OBJC2 被设定的环境），不管是 MRC 还是 ARC 下，区域（NSZone）都已单纯地被忽略。
    * 如果要浅拷贝，`copyWithZone:` 方法就返回同一个对象：return self；
    * 如果要深拷贝，`copyWithZone:` 方法中就创建新对象，并给希望拷贝的属性赋值。
* 如果自定义对象支持可变拷贝和不可变拷贝，那么还需要遵守 `NSMutableCopying` 协议，并实现 `mutableCopyWithZone:` 方法，返回可变副本。而 `copyWithZone:` 方法返回不可变副本。使用方可根据需要调用该对象的 copy 或 mutableCopy 方法来进行不可变拷贝或可变拷贝。

# 第四章：协议与分类
协议与分类是两个需要掌握的重要语言特性。若运用得当，则可令代码易读、易维护且少出错。本章将帮助读者精通这两个概念。

## 23. 通过委托与数据源协议进行对象间通信
**要点**
* 委托模式为对象提供了一套接口，使其可由此将相关事件告知其他对象。
* 将委托对象应该支持的接口定义成协议，在协议中把可能需要处理的事件定义成方法。
* 当某对象需要从另外一个对象中获取数据时，可以使用委托模式。这种情况下，该模式亦称 “数据源协议” (data source protoco)。
* 若有必要，可实现含有位段的结构体，将委托对象是否能响应相关协议方法这一信息缓存至其中。
**笔者**
* 可以通过委托 （也就是我们平常所说的 “代理 delegate” ）与数据源（data source）协议进行对象间通信。
* 协议中可以定义什么？方法和属性。
* 在协议中可以通过 `@optional` 和 `@require` 关键字来指定协议方法是可选择实现的还是必须实现的，如果使用方没有实现 @require 方法那么编译器就会给出警告。
* 委托模式（亦称为 “代理模式”）
    * 何为代理？一种软件设计模式；iOS 当中以 `@protocol` 形式体现；传递方式为一对一。
    * 代理模式的主旨：
        <br>定义一个委托协议，若对象想接受另一个对象（委托方）的委托，则需遵守该协议，以成为 “代理方”。而委托方则可以通过协议方法给代理方回传一些信息，也可以在发生相关事件时通知代理方。这样委托方就可以把应对某个行为的责任委托给代理方去处理了。
    * 代理的工作流程：
        * “委托方” 要求 “代理方” 需要实现的接口，全都定义在 “委托协议” 当中；
        * “代理方” 遵守 “协议” 并实现 “协议” 方法；
        * “代理方” 所实现的 “协议” 方法可能会有返回值，将返回值返回给 “委托方” ；
        * “委托方” 调用 “代理方” 遵从的 “协议” 方法。
        ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05abe7e888db497a8dab182bdec25971~tplv-k3u1fbpfcp-watermark.image)
    * delegate 属性一般定义为 weak 以避免循环引用。代理方强引用委托方，委托方弱引用代理方。
    * 如果要向外界公布一个类遵守某协议，那么就在接口中声明；<br>如果这个协议是委托协议，那么就在类扩展中声明，因为该协议通常只会在类的内部使用。
    * 对于 @optional 方法，在委托方中调用时，需要先判断代理方是否能响应（也就是它是否实现了该方法），如果能响应才能给它发送协议消息，否则代理方可能没有实现该方法，调用时就会因找不到方法实现而导致 Crash：
        ```objc
        if ([_delegate respondsToSelector:@selector(protocolOptionalMethod)]) {
            [_delegate protocolOptionalMethod];
        }
        ```
        最好是判断一下 delegate 是否有值，并将此判断条件前置提升执行效率。
        ```
        if (_delegate && [_delegate respondsToSelector:@selector(protocolOptionalMethod)]) {
            [_delegate protocolOptionalMethod];
        }
        ```
* 数据源模式
    * 委托模式的另一用法，旨在向类提供数据，所以也称 “数据源模式”。
        <br>数据源模式是用协议定义一套接口，令某类经由该接口获取其所需的数据。
* 数据源模式与常规委托模式的区别在于：
    * 数据源模式中，信息从数据源（Data Source）流向类（委托方）；
    * 常规委托模式中，信息从类（委托方）流向受委托者（代理方）。
    ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5497d2447b7545dda6b3da5e9815bfcf~tplv-k3u1fbpfcp-watermark.image)
* 通过 UITableView 就可以很好的理解 Data Source 和 Delegate 这两种模式：
    * 通过 UITableViewDataSource 协议获取要在列表中显示的数据；
    * 通过 UITableViewDelegate 协议来处理用户与列表的交互操作。
        ```objc
        @protocol UITableViewDataSource<NSObject>
        @required
        - (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section;
        - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;
        @optional
        - (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView; 
        ...
        @end

        @protocol UITableViewDelegate<NSObject, UIScrollViewDelegate>
        @optional
        -(void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath;
        ...
        @end
        ```
* 性能优化
    <br>在实现委托模式和数据源模式时，如果协议方法时可选的，那么在调用协议方法时就需要判断其是否能响应。
    ```objc
    if (_delegate && [_delegate respondsToSelector:@selector(protocolOptionalMethod)]) {
        [_delegate protocolOptionalMethod];
    }
    ```
    如果我们需要频繁调用该协议方法，那么仅需要第一次判断是否能响应即可。以上代码可做性能优化，将代理方是否能响应某个协议方法这一信息缓存起来：
    1. 在委托方中嵌入一个含有位域（bitfield，又称 “位段”、“位字段”）的结构体作为其实例变量，而结构体中的每个位域则表示 delegate 对象是否实现了协议中的相关方法。该结构体就是用来缓存代理方是否能响应特定的协议方法的。
        ```objc
        @interface EOCNetworkFetcher () {
            struct {
                unsigned int didReceiveData      : 1;
                unsigned int didFailWithError    : 1;
                unsigned int didUpdateProgressTo : 1;
            } _delegateFlags;
        }
        ```
    2. 重写 delegate 属性的 setter 方法，对 _delegateFlags 结构体里的标志进行赋值，实现缓存功能。
        ```objc
        - (void)setDelegate:(id<EOCNetworkFetcher>)delegate {
            _delegate = delegate;
            _delegateFlags.didReceiveData = [delegate respondsToSelector:@selector(networkFetcher:didReceiveData:)];
            _delegateFlags.didFailWithError = [delegate respondsToSelector:@selector(networkFetcher:didFailWithError:)];
            _delegateFlags.didUpdateProgressTo = [delegate respondsToSelector:@selector(networkFetcher:didUpdateProgressTo:)];
        }
        ```
    3. 这样每次调用 delegate 的相关方法之前，就不用通过 respondsToSelector: 方法来检测代理方是否能响应特定协议方法了，而是直接查询结构体中的标志，提升了执行速度。
        ```objc
        if (_delegateFlags.didReceiveData) {
            [_delegate networkFetcher:self didReceiveData:data];
        }
        ```

## 24. 将类的实现代码分散到便于管理的数个分类之中
**要点**
* 使用分类机制把类的实现代码划分成易于管理的小块。
* 将应该视为 “私有” 的方法归入名为 Private 的分类中，以隐藏实现细节。
**笔者**
* 该篇讲解了分类的一种使用场合：分解体积庞大的类文件，可以将一个类按功能拆解成多个模块，方便代码管理。
* 这样还便于调试：对于某个分类的所有方法来说，分类名称都会出现在其符号中。
    <br>例如，“addFriend:” 方法的 “符号名”：
    ```objc
    -[EOCPerson(Friendship) addFriend:]
    ```
    这样根据符号名就可以精确定位到该方法所属的功能区（分类）。
* 创建一个名为 Private 的分类，将私有方法都放在这里，这样使用者就知道这里面的方法不应该直接调用。
* 关于分类详解，可以参阅：[Link:《OC 底层探索 - Category 和 Extension》](https://juejin.im/post/6844904067987144711)。


## 25. 总是为第三方类的分类名称加前缀
**要点**
* 向第三方类中添加分类时，总应给其名称加上你专用的前缀。
* 向第三方类中添加分类时，总应给其中的方法名加上你专用的前缀。
**笔者**
* 分类机制还通常用于向无源代码的既有类中新增功能。
* 分类是运行时决议。何为运行时决议？Category 编译之后的底层结构是 struct category_t，里面存储着分类的对象方法、类方法、属性、协议信息，这时候分类中的数据还没有合并到类中，而是在程序运行的时候通过 Runtime 机制将所有分类数据合并到类（类对象、元类对象）中去。这是分类最大的特点，也是分类和扩展的最大区别，扩展是在编译的时候就将所有数据都合并到类中去了。
* 需要注意的是：
    1. 分类方法会 “覆盖” 同名的宿主类方法，如果使用不当会造成问题。方法被覆盖会导致执行结果和你预期的不同，且这种 bug 很难排查；
    2. 同名分类方法谁能生效取决于编译顺序，最后参与编译的分类中的同名方法会最终生效；
    3. 名字相同的分类会引起编译报错。
* 为避免以上问题，可以以命名空间来区别各个分类的名称与分类中所定义的方法。而在 OC 中实现命名空间功能只有一个办法，就是给相关名称都加上某个共用的前缀。这样分类和宿主类中出现同名方法导致方法被 “覆盖” 的问题的几率就会小很多。
    <br>向第三方类中添加分类时，更应该注意这个问题。

## 26. 勿在分类中声明属性
**要点**
* 把封装数据所用的全部属性都定义在主接口里。
* 在 “class-continuation 分类”（类扩展）之外的其他分类中，可以定义存取方法，但尽量不要定义属性。
**笔者**
* 分类中可以添加属性，但应该尽量避免这样做。
    <br>类扩展是编译时决议，在编译的时候就将扩展中的所有数据都合并到类中去了，所以扩展中添加属性没有任何问题。而分类是运行时决议，类的内存布局在编译时就已经确定，所以分类中无法添加实例变量，分类中添加的属性也不会自动生成实例变量以及 setter 和 getter 方法的实现（因为属性就是对实例变量的封装）。
* 假如你在分类中添加了属性，编译器就会给出警告：
    ```objc
    warning: Property ‘friends’ requires method ‘friends’ to be defined - use @dynamic or provide a method implementation in this category [-Wobjc-property-implementation]
    warning: Property ‘friends’ requires method ‘setFriends’ to be defined - use @dynamic or provide a method implementation in this category [-Wobjc-property-implementation]
    ```
    警告为：属性的 setter 和 getter 方法没有实现。因为分类中的属性不会自动生成实例变量以及 setter 和 getter 方法的实现，这样外部调用该属性的存取方法就会 Crash。有两种解决方式：
    1. 手动添加 setter 和 getter 方法的实现；
    2. 使用 @dynamic 告诉编译器，你会在运行时再提供这些方法的实现，以消除警告。你可以使用动态方法解析为这些方法动态添加方法实现。但如果你没有去处理的话，@dynamic 就仅仅是消除了警告，如果外部调用了该属性的存取方法还是会 Crash。
* 可以通过关联对象来解决分类中无法添加实例变量的问题：
    <br>由于分类底层结构的限制，不能直接给 Category 添加成员变量，但是可以通过关联对象间接实现 Category 有成员变量的效果。
    <br>可参阅：[Link:《OC 底层探索 - Association 关联对象》](https://juejin.im/post/6844903972315070471)
    <br>需要注意的是，存储关联对象时其内存管理语义需要与属性的一致。如果属性的内存管理语义更改，那么关联对象的关联策略也要修改，这是容易忽略的地方。
* 只读属性可以在分类中使用，我们为其实现 getter 方法。由于实现属性所需的全部方法（只读属性只需实现 getter 方法）都已实现，所以编译器就不会再为该属性自动合成实例变量，也不会发出警告。
* 类接口与类扩展是真正能够定义实例变量的地方，而属性只是定义实例变量及相关存取方法所用的 “语法糖”，所以也应该遵循同实例变量一样的规则。尽管分类中可以通过关联对象的手段来实现分类中可以添加实例变量的效果，但其目标在于扩展类的功能，而非封装数据。属性是用来封装数据的，所以在分类中可以定义存取方法，但尽量不要定义属性。


## 27. 使用 “class-continuation 分类” 隐藏实现细节
**要点**
* 通过 “class-continuation 分类”（类扩展）向类中新增实例变量。
* 如果某属性在主接口中声明为 “只读”，而类的内部又要用设置方法来修改此属性，那么就在 “class-continuation 分类” 中将其扩展为 “可读写”。
* 把私有方法的原型声明在 “class-continuation 分类” 里面。
* 若想使类所遵循的协议不为人所知，则可于 “class-continuation 分类” 中声明。
**笔者**
* 虽然 OC 的动态消息系统 Runtime【🚩 11】的工作方式决定了其不可能实现真正的私有方法或私有变量，但我们最好还是只把确实需要对外公布的那部分内容公开，而把无须对外公布的方法及属性、实例变量声明在类扩展中。
* 如果一个属性在类的内部需要进行存取值，而对外只允许使用方进行取值。那么可以在类声明中将该属性的读写权限设置为 `readonly` 只读，而在类扩展中再次声明该属性并设置为 `readwrite` 可读写。
* 若类所遵循的协议只应视为私有，比如委托协议和数据源协议，那么该协议就可以在类扩展中去遵从。
* 类扩展中除了可以声明属性、实例变量，遵从协议，还可以声明私有方法。虽然现在编译器不强制要求我们在使用方法之前必须先声明，直接在 class implementation 中实现即可。但其实在类扩展中声明一下私有方法还是有好处的，这样可以把类里所含的相关方法都统一描述于此，使代码可读性更高。

## 28. 通过协议提供匿名对象
**要点**
* 协议可在某种程度上提供匿名类型。具体的对象类型可以淡化成遵从某协议的 id 类型，协议里规定了对象所应实现的方法。
* 使用匿名对象来隐藏类型名称（或类名）。
* 如果具体类型不重要，重要的是对象能够响应（定义在协议里的）特定方法，那么可使用匿名对象来表示。
**笔者**
* 你可以通过协议提供匿名对象来隐藏类名，做法就是将对象声明为遵从某协议的 id 类型：`id <protocol> object`。
    1. 比如 delegate 属性，其声明为：
        ```objc
        @property (nonatomic, weak) id <EOCDelegate> delegate;
        ```
        委托方无须关心代理方的具体类型，只需代理方遵守委托协议并实现协议方法即可，这样委托方就可以向代理方发送协议消息了。
    2. 在字典中，键和值的标准内存管理语义分别是 “设置时拷贝” 和 “设置时保留”。因此 NSMutableDictionary 设置键值的方法为：
        ```objc
        - (void)setObject:(id)object forKey:(id<NSCopying>)key;
        ```
        参数 key 的类型为 id<NSCopying>，因此你可以传入遵守 NSCopying 协议的任何类型的 OC 对象，这样字典就能向该对象发送拷贝消息了，而这个 key 参数就可以视为匿名对象。
* 可以在运行期查出匿名对象所属类型，但这样做不好，因为匿名对象已经表明它的具体类型无关紧要了，你仅需要通过它来调用协议方法就好。
* 使用匿名对象的情况：
    * 接口背后有多个不同的实现类，而你又不想指明具体使用哪个类。因为有时候这些类可能会变，有时候它们又无法容纳于标准的类继承体系中，因而不能以某个公共基类统一表示。
        <br>这样就可以将这些对象所具备的方法定义在协议中，用 id 类型指代并遵守该协议，即可调用协议中的方法，而在运行期则会根据对象具体类型，调用具体的方法实现。
    * 对象具体类型不重要，重要的是对象能够响应（定义在协议里的）特定方法。即便该对象类型是固定的你也可以这么做，以表示类型在此处不重要。

