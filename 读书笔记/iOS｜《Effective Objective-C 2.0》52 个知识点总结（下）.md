
# 系列文章
* [肝了两个月的《Effective Objective-C 2.0》52 个知识点总结](https://juejin.cn/post/6904439922490441742)
* [《Effective Objective-C 2.0》52 个知识点总结（上）](https://juejin.cn/post/6904440708006936590/)
* [《Effective Objective-C 2.0》52 个知识点总结（下）](https://juejin.cn/post/6904440732287762439/)

# 第五章：内存管理

Objective-C 语言以引用计数来管理内存，这令许多初学者纠结，要是用过以 “垃圾收集器”（garbage collector）来管理内存的语言，那么更会如此。“自动引用计数” 机制缓解了此问题，不过使用时有很多重要的注意事项，以确保对象模型正确，不致内存泄漏。本章提醒读者注意内存管理中易犯的错误。

## 29. 理解引用计数
**要点**
* 引用计数机制通过可以递增递减的计数器来管理内存。对象创建好之后，其保留计数至少为 1。若保留计数为正，则对象继续存活。当保留计数降为 0 时，对象就被销毁了。
* 在对象生命期中，其余对象通过引用来保留或释放此对象。保留与释放操作分别会递增及递减保留计数。
**笔者**
* 苹果从 MacOS X 10.8 开始弃用了 GC（Garbage collector）机制，而使用 ARC 机制。而在 iOS 5 后 MRC 也被 ARC 所替代。虽说 ARC 已经帮我们实现了自动引用计数（也称 “保留计数” ），但掌握内存管理的种种细节是非常有必要的。
* 引用计数工作原理
    * 引用计数机制通过可以递增递减的计数器来管理内存。对象创建好之后，其保留计数至少为 1。若保留计数为正，则对象继续存活。当保留计数降为 0 时，对象就被销毁了。
    * 在对象生命期中，其余对象通过引用来保留或释放此对象。保留与释放操作分别会递增及递减保留计数。
    * 引用计数相关的内存管理方法（在 ARC 下这些方法都是禁止调用的）：
        * retain：递增保留引用计数
        * release：递减保留引用计数
        * autorelease：待稍后清理自动释放池时，再递减保留计数
        * retainCount： 查看保留计数，此方法不太好用，即便在调试时也如此，因此不推荐使用。原因之一是调用方可能通过 autorelease 来延迟释放对象，所以其保留计数值不是我们所期望的。【🚩 36】
    * 按 “引用树” 回溯，有一个根对象对它们进行保留，在 iOS 中是 UIApplication 对象。在 MacOS 中是 NSApplication 对象，两者都是应用程序启动时创建的单例。
    * 当引用计数降至 0 时，就不应该再使用它，这可能会导致 Crash。
        <br>当引用计数降至 0 时对象所占内存就会 “解除分配”，放回 “可用内存池”，如果这时候使用该对象且该对象内存未被覆写，那么就不会 Crash。如果对象内存被覆写，那么就会 Crash。
        <br>为避免在不经意间使用了无效对象，一般调用完 release 之后都会清空指针。这就能保证不会出现可能指向无效对象的指针，这种指针通常称为 “悬垂指针”。
* 属性存取方法中的内存管理
    <br>在 MRC 中声明为 retain 的属性，其 setter 方法的实现如下：
    ```objc
    @property (nonatomic, retain) NSNumber *count;

    - (void)setCount:(NSNumber *)newCount {
        [newCount retain];
        [_count release];
        _count = newCount;
    }
    ```
    必须先 retain 新值再 release 旧值，否则如果新旧对象是同一个对象的话，先 release 可能会导致其引用计数减为 0 而被销毁。而后续的 retain 操作无法令其起死回生，于是实例变量就成了悬垂指针，从而导致 Crash。
* 自动释放池
    <br>调用 release 会立刻递减对象的引用计数，而调用 autorelease 则是延迟 release 以延长对象生命期，对象会被添加进自动释放池。通常是在下一次 “事件循环” 时给对象发送 release 消息，除非你将其添加进手动创建的自动释放池。
    <br>autorelease 经常用于在方法中返回对象时，使对象在跨越方法调用边界后依然可以存活一段时间。而如果使用 release，在方法返回前系统就把该对象回收了。
* 保留环
    <br>如果两个对象相互强引用对方，或者多个对象，每个对象都强引用下一个对象直到回到第一个，那么就会产生 “保留环”，也就是平常所说的 “循环引用”。这时候循环中的所有对象的引用计数就不会降为 0，也就不会被释放，就产生了内存泄露。因此，我们要避免保留环的产生。通常采用 “弱引用” 或是从外界命令循环中的某个对象不再保留另外一个对象来避免或打破保留环，从而避免内存泄漏。

## 30. 以 ARC 简化引用计数
**要点**
* 有 ARC 之后，程序员就无须担心内存管理问题了。使用 ARC 来编程，可省去类中的许多 “样板代码”。
* ARC 管理对象生命期的办法基本上就是：在合适的地方插入 “保留” 及 “释放” 操作。在 ARC 环境下，变量的内存管理语义可以通过修饰符指明，而原来则需要手工执行 “保留” 及 “释放” 操作。
* 由方法返回的对象，其内存管理语义总是通过方法名来体现。ARC 将此确定为开发者必须遵守的规则。
* ARC 只负责管理 Objective-C 对象的内存。尤其要注意：Core Foundation 对象不归 ARC 管理，开发者必须适时调用 CFRetain/CFRelease。
**笔者**
* ARC 是一种编译器功能，它通过 `LLVM` 编译器和 `Runtime` 协作来进行自动管理内存。LLVM 编译器会在编译时在合适的地方为 OC 对象插入 retain、release 和 autorelease 代码来自动管理对象的内存。
* 在 ARC 下 retain、release、autorelease、dealloc、retainCount 等方法都是禁止调用的，否则编译错误，因为手动调用会干扰 ARC 分析对象生命期的工作。
* ARC 在调用这些方法时，并不通过 OC 消息派发机制（也就是 objc_msgSend），而是直接调用其底层 C 语言版本。这样做性能更好，因为保留及释放等操作需要频繁执行，所以直接调用底层函数能节省很多 CPU 周期。
    <br>比如调用与 `retain` 等价的底层函数 `objc_retain`。这也是不能覆写 retain、release 或 autorelease 的缘由，因为这些方法不会被调用。
* ARC 带来的优化：
    1. 优化一：自动调用 “保留” 与 “释放” 方法。
        <br>在 MRC 中，若方法名以 alloc、new、copy、mutableCopy 开头，则返回的对象归调用者所有，因此调用这四种方法的那段代码要负责释放方法所返回的对象；若方法名不以上述四种词语开头，则返回的对象不归调用者所有，返回的对象会自动释放，所以调用方在使用该返回的对象前应该对其进行保留，并在使用完毕后对它进行释放。而在 ARC 中这些不用我们自己处理，ARC 通过命名约定将内存管理规则标准化。
    2. 优化二：在编译期，ARC 会把能够相互抵消的 retain、release、autorelease 操作约简。
        <br>如果发现在同一个对象上执行了多次 “保留” 与 “释放” 操作，那么 ARC 有时可以成对地移除这两个操作。这是手工操作很难甚至无法完成的优化。
    3. 优化三：如果调用方法返回一个本将(`autorelease`)添加进自动释放池的对象，其后紧跟着(`retain`)保留该对象操作，则 ARC 会调用其他函数来取代 autorelease 和 retain，让该对象不添加进自动释放池而是直接返回给调用方，以减去多余操作，提升性能。（考虑到向后兼容性，以兼容不使用 ARC 的代码，不能将 autorelease 与 retain 操作删去，也不能舍弃 autorelease，来进行这个优化）
        <br>ARC 会在运行期进行检测：
        1. 在方法中返回自动释放的对象时，会调用 `objc_autoreleaseReturnValue` 函数而不是 `autorelease`，该函数会检视当前方法返回之后即将要执行的那段代码。若发现那段代码要在返回的对象上执行 retain 操作，则设置全局数据结构（此数据结构的具体内容因处理器而异）中的一个标志位，并直接返回该对象，否则执行 autorelease 再返回。
        2. 如果方法返回了一个自动释放的对象，而调用方法的代码要保留此对象，会调用 `objc_retainAutoreleasedReturnValue` 函数而不是 `retain`。该函数会检视那个标志位，若已经置位，则不执行 retain 操作，否则执行 retain。
            >设置并检测标识位，要比调用 autorelease 和 retain 要快。
    4. 优化四：ARC 也会处理局部变量和实例变量的内存管理。
        <br>ARC 默认情况下，每个变量都是指向对象的强引用。
        <br>在 ARC 下，我们编写 setter 方法只需这样：
        ```objc
        - (void)setObject:(id)object {
            _object = object;
        }
        ```
        其经过编译会变为：
        ```objc
        - (void)setObject:(id)object {
            [object retain];
            [_object release];
            _object = object;
        }
        ```
        替我们省去手工 retain 与 release 操作的同时，还避免了我们可能会先释放旧对象再保留新对象，而恰巧新旧又是同一对象导致 Crash 的问题。
    5. 优化五：在对象销毁（dealloc）时帮我们执行实例变量的释放
        <br>以下我们在 MRC 下必须要执行的操作，而 ARC 会自动帮我们处理。ARC 会借用 Objcetive-C++ 的一项特性来生成清理例程。回收 Objcetive-C++ 对象时，待回收的对象会调用所有 C++ 对象的析构函数。编译器如果发现这个对象里含有 C++ 对象，就会生成名为 .cxx_destruct 的方法。而 ARC 则借用此特性，在该方法中生成清理内存所需的代码。
        ```objc
        - (void)dealloc {
            [_foo release];
            [_bar release];
            [super release];
        }
        ```
        不过，如果有非 Objective-C 的对象，比如 CoreFoundation 中的对象或是由 malloc() 分配在堆中内存，需要手动清理。但不需要也不能调用 `[super dealloc]`，因为 ARC 会在 .cxx_destruct 方法中生成并执行调用此方法的代码。
        ```objc
        - (void)dealloc {
            CFRelease(_coreFoundationObject);
            free(_heapAllocatedMemoryBlob);
        }
        ```
* 所有权修饰符
    * __strong: 默认，保留此值
    * __unsafe_unretained: 不保留此值，不过其会产生悬垂指针，不安全
    * _ _weak: 不保留此值，在对象销毁时会把 __weak 指针置空，所以安全
    * __autoreleasing: 把对象 “按引用传递” 给方法时，使用这个修饰符。此值在方法返回时自动释放
* 在 MRC 下，我们可能会覆写内存管理方法。比方说，在实现单例类的时候我们经常会覆写 release 方法，将其替换为空操作，因为单例不可释放。
    <br>而在 ARC 下，不能调用和覆写内存管理方法，因为这会干扰 ARC 分析对象生命期的工作。而且正由于不能这么做，ARC 就可以执行各种优化。比方说，ARC 能够优化 retain、release、autorelease 操作，使之不经过 OC 消息派发机制，而是直接调用隐藏在运行期库 objc4 中的 C 函数。还有前面提到的 ARC 会优化 “如果方法命令即将返回的对象稍后 ‘自动释放’，而方法调用者立刻 ‘保留’ 这个返回后的对象” 中的 autorelease 和 retain 操作。

## 31. 在 dealloc 方法中只释放引用并解除监听
**要点**
* 在 dealloc 方法里，应该做的事情就是释放指向其他对象的引用，并取消原来订阅的 “键值观测”（KVO）或 NSNotificationCenter 等通知，不要做其他事情。
* 如果对象持有文件描述符等系统资源，那么应该专门编写一个方法来释放此种资源。这样的类要和其使用者约定：用完资源后必须调用 close 方法。
* 执行异步任务的方法不应在 dealloc 里调用；只能在正常状态下执行的那些方法也不应在 dealloc 里调用，因为此时对象已处于正在回收的状态了。
**笔者**
* 在 dealloc 方法中应该做什么？
    * 释放对象所拥有的引用。ARC 会通过生成 .cxx_destruct 方法在 dealloc 中为我们自动添加释放代码来释放 OC 对象，而非 OC 对象比如 CoreFoundation 对象就必须手动释放。
    * 移除 KVO 观察者。在调用 KVO 注册方法后，KVO 并不会对观察者进行强引用，所以需要注意观察者的生命周期。至少需要在观察者销毁之前，调用 KVO 移除方法移除观察者，否则如果在观察者被销毁后，再次触发 KVO 监听方法就会导致 Crash。
        <br>[Link:《iOS - 关于 KVO 的一些总结》](https://juejin.im/post/6844903972528979976#heading-22)
    * 移除通知观察者。移除后通知中心就不会再把通知发送给已经销毁回收的对象，若是还向其发送通知，则必然导致 Crash。
        >iOS9 之后通知不必手动移除，原因是通知中心现在用 __weak 保留观察者，而以前是用 __unsafe_unretain 从而如果不移除观察者的话会产生悬垂指针，再给该对象发送通知就会 Crash。不过通过以下 API 注册的通知还是需要手动移除。
        >```objc
        >- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block;
        >```
* 在 dealloc 方法中不应该做什么？
    * 不应该释放开销较大或系统内稀缺的资源，比如文件描述符、套接字、大块内存等。
        <br>原因是：
        1. 不能指望 dealloc 方法会在自己期望的时机调用，因为可能会有一些无法预料的东西持有此对象，这样就不能在自己期望的时机释放稀缺资源。
            > 在应用程序终止时，系统会销毁所有未被销毁的对象，而这时候系统为了优化程序效率并不调用每个对象的 dealloc 方法。如果一定要清理某些对象，可以在以下方法中调用那些对象的清理方法，以下方法会在程序终止时调用。
            >```objc
            >// iOS，UIApplicationDelegate
            >- (void)applicationWillTerminate:(UIApplication *)application
            >// Mac OS X，NSApplicationDelegate
            >- (void)applicationWillTerminate:(NSNotification *)notification
            >```
        2. 不能保证每个对象的 dealloc 方法都会执行，因为可能存在内存泄露。
            通常的做法是，实现另外一个清理方法，当用完资源后就调用此方法释放资源。这样资源对象的生命期就变得更为明确了。
            <br>而在 dealloc 中最好也要调用下清理方法，以防开发者忘了清理这些资源。并输出错误信息或是抛出异常，来提醒开发者在回收对象之前必须调用清理方法来释放资源。
            ```objc
            - (void)close {
                /* clean up resources */
                _closed = YES;
            }

            - (void)dealloc {
                if (!_closed) {
                    NSLog(@"Error: close was not called before dealloc!"); // or @throw expression
                    [self close];
                }
            }
            ```
    * 不应该随便调用其他方法。
        <br>因为在 dealloc 中对象已经即将销毁回收。如果在这里调用的方法又要异步执行某些任务，或是又要继续调用它们自己的某些方法，那么等到那些任务执行完毕时，系统已经把当前的这个待回收的对象彻底销毁了。如果此时在回调中又使用该对象，就会 Crash。
        <br>再注意一个问题，调用 dealloc 方法的那个线程会执行 “最终的释放操作”，而某些方法必须在特定的线程里（比如主线程里）调用才行。若在 dealloc 方法中调用了那些方法，则无法保证当前这个线程就是那些方法所需的线程。
    * 不应该调用属性的存取方法。
        <br>因为有人可能会覆写这些方法，并于其中做一些无法在回收阶段安全执行的操作。此外，属性可能正处于 KVO 机制的监控之下，该属性的观察者可能会在属性值改变时保留或使用这个即将回收的对象。这样会令运行期系统的状态完全失调，从而导致一些莫名其妙的错误。

## 32. 编写 “异常安全代码” 时留意内存管理问题
**要点**
* 捕获异常时，一定要注意将 try 块内所创立的对象清理干净。
* 在默认情况下，ARC 不生成安全处理异常所需的清理代码。开启编译器标志后，可生成这种代码，不过会导致应用程序变大，而且会降低运行效率。
**笔者**
* 编写 “异常安全代码 `try-catch-finally`” 时要留意内存管理问题，否则很容易发生内存泄漏。
* 一般情况下，只有当发生严重错误，应用程序必须因异常而终止时才应抛出异常，而程序若终止则是否还会发生内存泄漏就无关紧要了。
* 有时仍然需要编写代码来捕获并处理异常。比如使用 Objective-C++ 来编码时（C++ 与 Objective-C 的异常相互兼容），或是编码中用到了第三方库而该库所抛出的异常又不受你控制时。
* 捕获异常时，一定要注意将 try 块内所创立的对象清理干净。
    * 如果在 MRC 下，如果对象 release 操作在 try 块中执行并且它前面的代码中抛出了异常，会导致 release 操作不会执行从而内存泄漏。解决办法是将 release 操作放在 finally 块中执行，而该对象就必须声明为全局。缺点就是，如果 try 块中的逻辑复杂，就很容易忘记对某个对象执行 release 而导致内存泄漏，如果对象是稀缺资源则更严重。
        ```objc
        EOCSomeClass *object;
        @try {
            object = [[EOCSomeClass alloc] init];
            [object doSomethingThatMayThrow];
        }
        @catch (...) {
            NSLog(@"Whoops, there was an error, Oh well...");
        }
        @finally {
            [object release];
        }
        ```
    * 如果在 ARC 下，在 “异常安全代码” 中，ARC 默认是不会自动帮我们插入 release 操作的。其原因是加入大量的这些代码会严重影响运行期的性能，即便在不抛异常时也是如此，而且还会明显增加应用程序的大小。
        1. 可以通过开启编译器标志 -fobjc-arc-exceptions ，来生成这种附加代码（release）。其默认不开启的原因即为上面提到的：一般情况下，只有当发生严重错误，应用程序必须因异常而终止时才应抛出异常，而程序若终止则是否还会发生内存泄漏就无关紧要了，添加安全处理异常所用的附加代码（比如 release）也没有意义。而且开启还会导致应用程序变大，降低运行效率等。
        2. 有种情况下，编译器会自动开启  -fobjc-arc-exceptions，就是当文件是 Objective-C++ 文件时。
* 当发现大量异常捕获操作时，应考虑重构代码，用 NSError 式错误信息传递法来取代异常。

## 33. 以弱引用避免保留环
**要点**
* 将某些引用设为 weak，可避免出现 “保留环”。
* weak 引用可以自动清空，也可以不自动清空，自动清空（autonilling）是随着 ARC 而引入的新特性，由运行期系统来实现。在具备自动清空功能的弱引用上，可以随意读取其数据，因为这种引用不会指向回收过的对象。
**笔者**
* 保留环：
    * 两个对象通过彼此之间的强引用而构成保留环。
    * 多个对象，每个对象都强引用下一个对象直到回到第一个，构成保留环。
* 保留环会导致内存泄漏：
    <br>如果最后一个指向保留环里对象的引用被移除，那么保留环里的对象就无法被外界所访问，而这些对象彼此之间都互相强引用就不会销毁，于是就导致了内存泄漏。（内存泄漏是指没有释放已分配的不再被使用的内存。）
    >之前 Mac OS X 平台的 Objective-C 程序可以启用垃圾收集器（garbage collector），它会检测保留环，若发现外界不再引用其中的对象，则将其回收。但从 Mac OS X 10.8 开始，垃圾收集机制就被废弃了，而改用 ARC 机制。不管是 MRC 还是 ARC 机制都从未支持自动检测回收保留环，因此我们要避免保留环的产生。
* 以弱引用避免保留环
    * `unsafe_unretained`：顾名思义，不保留对象但也不安全。不安全的原因在于：如果指针所指向的对象被销毁，指针仍然指向该对象的内存地址，变成悬垂指针，如果这时候再去使用该指针就会导致 Crash。
    * `weak`：ARC 下才能使用，与 unsafe_unretained 一样不会保留对象。不同的是，如果指针所指向的对象被销毁，指针会自动置为 nil，再去使用该指针也是安全的。
    ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/474c520b4f9f427fb7740a9432927b5d~tplv-k3u1fbpfcp-watermark.image)

## 34. 以 “自动释放池块” 降低内存峰值
**要点**
* 自动释放池排布在栈中，对象收到 autorelease 消息后，系统将其放入最顶端的池里。
* 合理运用自动释放池，可降低应用程序中的内存峰值。
* @autoreleasepool 这种新式写法能创建出更为轻便的自动释放池。
**笔者**
* 调用 autorelease 方法的对象会被加入到自动释放池中，`@autoreleasepool{}` 自动释放池于左花括号处创建，于右花括号处销毁。当自动释放池销毁时，会给其当中的所有对象发送 release 消息。
* 如果在没有创建自动释放池的情况下给对象发送 autorelease 消息，那么控制台就会输出以下信息。
    ```objc
    MISSING POOLS: (0xabcd1234) Object 0xabcd0123 of class __NSCFString autoreleased with no pool in place - just leaking - break on objc_autoreleaseNoPool() to debug
    ```
    不过一般情况下无须担心自动释放池的创建问题，系统会自动创建一些线程，比如说主线程或是 GCD 机制中的线程，这些线程默认都有自动释放池，每次执行事件循环时就会将其清空。
* 合理运用自动释放池，可降低应用程序中的内存峰值（内存峰值指应用系统在某个特定时间段内的最大内存用量）。不过，尽管自动释放池的开销不大，但毕竟还是有的。使用其来优化效率前应该先监控内存用量来判断是否有用自动释放池的必要，不要滥用。
    <br>以下每次执行循环中都创建并清空自动释放池，可以及时释放每次循环中产生的 autorelease 对象，以降低内存峰值。
    ```objc
    for (int i = 0; i < 100000; i++) {
        @autoreleasepool {
            [self doSomethingWithInt:i];
        }
    }
    ```
* 在 MRC 下，可以使用 NSAutoreleasePool 或者 @autoreleasepool 来创建自动释放池，但在 ARC 下只能使用后者。想比于 NSAutoreleasePool，@autoreleasepool 的优点有：
    * 更轻量级，苹果说它比 NSAutoreleasePool 快大约六倍。NSAutoreleasePool 则更为重量级，通常用来创建偶尔需要清空的自动释放池。而 @autoreleasepool 可以在每次执行循环中创建并清空自动释放池。
        ```objc
        NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
        for (int i = 0; i < 100000; i++) {
            [self doSomethingWithInt:i];
            // Drain the pool only every 10 cycles
            if (++i == 10) {
                [pool drain];
            }
        }
        // Also drain ai the end in case the loop is not a multiple of 10
        [pool drain];
        ```
    * 其范围即是左右花括号，可以避免无意间误用了那些在清空池后已被系统所回收的对象。而 NSAutoreleasePool 则无法避免。
        ```objc
        NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
        id object = [self createObject];
        [pool drain];
        [self useObject:object];
        ```
        ```objc
        @autoreleasepool {
            id object = [self createObject];
        }
        [self useObject:object]; // 访问不了 object，无法编译
        ```
* 关于自动释放池更详细的解析，可以参阅：[Link:《iOS - 聊聊 autorelease 和 @autoreleasepool》](https://juejin.cn/post/6844904094503567368#heading-1)

## 35. 用 “僵尸对象” 调试内存管理问题
**要点**
* 系统在回收对象时，可以不将其真的回收，而是把它转化为僵尸对象。通过环境变量 NSZombieEnabled 可开启此功能。
* 系统会修改对象的 isa 指针，令其指向特殊的僵尸类，从而使该对象变为僵尸对象。僵尸类能够响应所有的选择子，响应方式为：打印一条包含消息内容及其接收者的消息，然后终止应用程序。
**笔者**
* 向业已回收的对象发送消息是不安全的，对象所占内存在 “解除分配(deallocated)” 之后，只是放回可用内存池。如果对象所占内存还没有分配给别人，这时候访问没有问题，如果已经分配给了别人，再次访问就会崩溃。
    >更精确的解释是：如果被回收的对象的内存只被复用了其中一部分，那么对象中的某些二进制数据依然有效，所以可能不会导致崩溃。如果那块内存恰好为另外一个有效且存活的对象所占据。那么运行期系统会把消息发到新对象那里，而此对象也许能应答，也许不能。如果能，程序就不会崩溃，但接收消息的对象已经不是我们预想的那个了；如果不能响应，则程序依然会崩溃。
    这在调试的时候可能不太方便，我们可以通过 “僵尸对象(Zombie Object)” 来更好地调试内存管理问题。
* 僵尸对象的启用：
    <br>通过环境变量 NSZombieEnabled 启用 “僵尸对象” 功能。
* 僵尸对象的工作原理：
    <br>它的实现代码深植与 Objective-C 的运行期库、Foundation 框架及 CoreFoundation 框架中。系统在即将回收对象时，如果发现 NSZombieEnabled == YES，那么就把对象转化为僵尸对象，而不是将其真的回收。接下来给该对象（此时已是僵尸对象）发送消息，就会在控制台打印一条包含消息内容及其接收者的信息（如下），然后终止应用程序。这样我们就能知道在何时何处向业已回收的对象发送消息了。
    ```objc
    [EOCClass message] : message sent to deallocated instance 0x7fc821c02a00
    ```
* 僵尸对象的实现原理：
    1. 在启用僵尸对象后，运行期系统就会 swizzle 交换 dealloc 方法实现，当每个对象即将被系统回收时，系统都会为其创建一个 `_NSZombie_OriginalClass` 类。（`OriginalClass` 为对象所属类类名，这些类直接由 `_NSZombie_` 类拷贝而来而不是使用效率更低的继承，然后赋予类新的名字 `_NSZombie_OriginalClass` 来记住旧类名。记住旧类名是为了在给僵尸对象发送消息时，系统可由此知道该对象原来所属的类。）。
        <br>然后将对象的 isa 指针指向僵尸类，从而待回收的对象变为僵尸对象。（由于是交换了 dealloc 方法，所有 free() 函数就不会执行，对象所占内存也就不会释放。虽然这样内存泄漏了，但也只是调试手段而已，所以泄漏问题无关紧要）。
    2. 由于 `_NSZombie_` 类（以及所有从该类拷贝出来的类 `_NSZombie_OriginalClass`）没有实现任何方法，所以给僵尸对象发送任何消息，都会进入 “完整的消息转发阶段”。而在 “完整的消息转发阶段” 中，`__forwarding__` 函数是核心。它首先要做的事情包括检查接收消息的对象所属的类名。若前缀为 `_NSZombie_` 则表明消息接收者是僵尸对象，需要特殊处理：在控制台打印一条信息（信息中指明僵尸对象所接收到的消息、原来所属的类、内存地址等，`[OriginalClass message] : message sent to deallocated instance 0x7fc821c02a00`），然后终止应用程序。


## 36. 不要使用 retainCount
**要点**
* 对象的保留计数看似有用，实则不然，因为任何给定时间上的 “绝对保留计数”（absolute retain count）都无法反映对象生命期的全貌。
* 引入 ARC 之后，retainCount 方法就正式废止了，在 ARC 下调用该方法会导致编译器报错。
**笔者**
* ARC 下已经禁止调用 retainCount 方法。但即便 MRC 下可以调用，或者只为了调试，我们也不应该使用 retainCount，其值是不精确的。
* 不要使用 retainCount 的原因：
    1. 它所返回的保留计数只是某个给定时间点上的值，它没有考虑到有些对象是调用 autorelease 方法添加进自动释放池的，这些对象会延迟释放（保留计数值 - 1），而 retainCount 只是直接把当前保留计数值返回，所以结果往往不符合预期。
        <br>比如以下代码有两个错误。一是，如果 object 在自动释放池中，那么当自动释放池清空的时候，就会因过度释放 object 而导致程序崩溃；二是，retainCount 可能永远不返回 0，因为有时系统会优化对象的释放行为，在保留计数还是 1 的时候就把它回收了。这样对象回收之后 while 循环可能仍在执行，就会导致程序崩溃。
        ```objc
        while ([object retainCount]) {
            [object release];
        }
        ```
    2. 当对象是字符串常量，或是使用 Tagged Point 来存储时，retainCount 的值会特别大，就没有查看的意义。单例对象的保留计数也不会变，因为单例不可释放，所以我们可能会覆写其内存管理方法，把保留和释放操作替换为空操作。
    所以，我们不应该总是依赖保留计数的具体值来编码。假如你根据 NSNumber 对象的具体保留计数来增减其值，而系统却以 Tagged Point 来实现此对象，那么编出来的代码就错了。

# 第六章：块与大中枢派发

苹果公司引入了 “块” 这一概念，其语法类似于 C 语言扩展中的 “闭包”（closure）。在 Objective-C 语言中，我们通常用块来实现一些原来需要很多样板代码才能完成的事情，块还能实现 “代码分离”（code separation）。“大中枢派发”（Grand Central Dispatch，GCD）提供了一套用于多线程环境的简单接口。“块” 可视为 GCD 的任务，根据系统资源状况，这些任务也许能并发执行。本章将教会读者如何充分运用这两项核心技术。

## 37. 理解 “块” 这一概念
**要点**
* 块是 C、C++、Objective-C 中的词法闭包。
* 块可接受参数，也可返回值。
* 块可以分配在栈或堆上，也可以是全局的。分配在栈上的块可拷贝到堆里，这样的话，就和标准的 Objective-C 对象一样，具备引用计数了。
**笔者**
* 块是 C、C++、Objective-C 中的词法闭包。块本身也是 OC 对象。
* 如果块中没有显式地使用 self 来访问实例变量，那么块就会隐式捕获 self，这很容易在我们不经意间造成循环引用。如下代码，编译器会给出警告，建议用 `self->_anInstanceVariable` 或 `self.anInstanceVariable` 来访问。
    ```objc
    self.block = ^{
        _anInstanceVariable = @"Something"；// ⚠️ Block implicitly retains ‘self’; explicitly mention ‘self’ to indicate this is intended behavior
    }
    ```
* 块的内存布局
    * isa 指针指向 Class 对象
    * invoke 变量是个函数指针，指向块的实现代码
    * descriptor 变量是指向结构体的指针，其中声明了块对象的总体大小，还声明了保留和释放捕获的对象的 copy 和 dispose 这两个函数所对应的函数指针
    * 块还会把它所捕获的所有变量都拷贝一份，放在 descriptor 变量的后面
    ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebf284ed0ae449efa1874cfc35492656~tplv-k3u1fbpfcp-watermark.image)
* 块的三种类型：栈块、堆块、全局块
    * 栈块
        <br>定义块的时候，其所占的内存区域是分配在栈中的。块只在定义它的那个范围内有效。
        ```objc
        void (^block)();
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
        上面的代码有危险，定义在 if 及 else 中的两个块都分配在栈内存中，当出了 if 及 else 的范围，栈块可能就会被销毁。如果编译器覆写了该块的内存，那么调用该块就会导致程序崩溃。
        >若是在 ARC 下，上面 block 会被自动 copy 到堆，所以不会有问题。但在 MRC 下我们要避免这样写。
    * 堆块
        <br>为了解决以上问题，可以给块对象发送 copy 消息将其从栈拷贝到堆区，堆块可以在定义它的那个范围之外使用。堆块是带引用计数的对象，所以在 MRC 下如果不再使用堆块需要调用 release 进行释放。
        ```objc
        void (^block)();
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
    * 全局块
        <br>如果运行块所需的全部信息都能在编译期确定，包括没有访问自动局部变量等，那么该块就是全局块。全局块可以声明在全局内存中，而不需要在每次用到的时候于栈中创建。
        <br>全局块的 copy 操作是空操作，因为全局块决不可能被系统所回收，其实际上相当于单例。
        ```objc
        void (^block)() = ^{
            NSLog(@"This is a block");
        };
        ```

## 38. 为常用的块类型创建 typedef
**要点**
* 以 typedef 重新定义块类型，可令块变量用起来更加简单。
* 定义新类型时应遵从现有的命名习惯，勿使其名称与别的类型相冲突。
* 不妨为同一个块签名定义多个类型别名。如果要重构的代码使用了块类型的某个别名，那么只需修改相应的 typedef 中的块签名即可，无须改动其他 typedef。
**笔者**
* 每个块都具备其 “固有类型（由其参数及返回值组成）”，因而可将其赋值给适当类型的变量。
* 以 typedef 关键字重新定义块类型：
    ```objc
    typedef int(^EOCSomeBlock)(BOOL flag, int value);
    EOCSomeBlock block = ^(BOOL flag, int value) {
        // Implementation
    }
    ```
* 以 typedef 关键字重新定义块类型的好处
    * 易读，定义、声明块方便
        <br>定义块变量语法与定义其他类型变量的语法不同，而定义方法参数所用的块类型语法又和定义块变量语法不同。这种语法很难记，所以以 typedef 关键字重新定义块类型，可令块变量用起来更加简单，起个更为易读的名字来表示块的用途，而把块的类型隐藏在其后面。
    * 重构块的类型签名时很方便
        <br>当修改 typedef 类型定义语句时，使用到这个类型定义的地方都无法编译，可逐个修复。而若是不用类型定义直接写块类型，修改的地方就更多，而且很容易忘掉其中一两处的修改而引发难于排查的 bug。
* 最好在使用块类型的类中定义这些 typedef，而且新类型名称还应该以该类名开头，以阐明块的用途。
* 可以根据不同用途为同一个块签名定义多个类型别名。如果要重构的代码使用了块类型的某个别名，那么只需修改相应的 typedef 中的块签名即可，无须改动其他 typedef。

## 39. 用 handler 块降低代码分散程度
**要点**
* 在创建对象时，可以使用内联的 handler 块将相关业务逻辑一并声明。
* 在有多个实例需要监控时，如果采用委托模式，那么经常需要根据传入的对象来切换，而若改用 handler 块来实现，则可直接将块与相关对象放在一起。
* 设计 API 时，如果用到了 handler 块，那么可以增加一个参数，使调用者可通过此参数来决定应该把块安排在哪个队列上执行。
**笔者**
* 当异步方法在执行完任务之后，需要以某种手段通知相关代码时，我们可以使用委托模式，也可以将 completion handler 定义为块类型。在对象的初始化方法中添加 handle 块参数，在创建对象时就将 handler 块传入。这样可以降低代码分散程度，令代码更加清晰整洁，令 API 更紧致，同时也令开发者调用起来更加方便。
    ```objc
    [fetcher startWithCompletionHandler:^(NSData *data) {
        self->_fetchedFooData = data;
    }];
    ```
* 委托模式的缺点：在有多个实例需要监控时，如果采用委托模式，那么就得在 delegate 回调方法里根据传入的对象来判断执行何种操作，回调方法的代码就会变得很长。
    ```objc
    - (void)networkFetcher:(EOCNetworkFetcher *)networkFetcher didFinishWithData:(NSData *)data {
        if (networkFetcher == _fooFetcher) {
            ...
        } else if (networkFetcher == _barFetcher) {
            ...
        }
        // etc.
    }
    ```
    改用 handler 块来实现，可直接将块与相关对象放在一起，就不用再判断是哪个对象。
    ```objc
    [_fooFetcher startWithCompletionHandler:^(NSData *data) {
        ...
    }];
    [_barFetcher startWithCompletionHandler:^(NSData *data) {
        ...
    }];
    ```
* 当异步的回调有成功和失败两种情况时，可以有两种 API 设计方式：
    * 用两个 handler 块来分别处理成功和失败的回调，这样可以让代码更易读，还可以根据需要对不想处理的回调块参数传 nil。
        ```objc
        [fetcher startWithCompletionHandler:^(NSData *data) {
            // Handle success
        } failureHander:^(NSError *error) {
            // Handle failure
        }];
        ```
    * 用一个 handler 块来同时处理成功和失败的回调，并给块添加一个 error 参数来处理失败情况。
        ```objc
        [fetcher startWithCompletionHandler:^(NSData *data, NSError *error) {
            if (error) {
                // Handle failure
            } else {
                // Handle success
            }
        }];
        ```
        * 缺点：全部逻辑放在一起会令块变得很长且复杂
        * 优点：更灵活，成功和失败情况有时候可能要统一处理
* 有时需要在相关时间点执行回调参数，就比如下载时想在每次有下载进度时都得到通知，这种情况也可以用 handler 块处理。
* 设计 API 时，如果用到了 handler 块，那么可以增加一个参数，使调用者可通过此参数来决定应该把块安排在哪个队列上执行。可以参照通知的 API：
    ```objc
    - (id)addObserverForName:(NSString *)name object:(id)object queue:(NSOperationQueue *)queue usingBlock:(void(^)(NSNotification *))block;
    ```

## 40. 用块引用其所属对象时不要出现保留环
**要点**
* 如果块所捕获的对象直接或间接地保留了块本身，那么就得当心保留环问题。
* 一定要找个适当的时机解除保留环，而不能把责任推给 API 的调用者。
**笔者**
* 本节讲的是块的循环引用问题。当使用块的时候很容易形成保留环，我们要分析它们间的引用关系，避免保留环的产生。如果形成了保留环，那么一定要找个适当的时机解除保留环。而且应该在内部去处理，而不能把断环的任务交给 API 的调用者去处理，因为无法保证调用者会这么做。


## 41. 多用派发队列，少用同步锁
**要点**
* 派发队列可用来表述同步语义，这种做法要比使用 @synchronized 块或 NSLock 对象更简单。
* 将同步与异步派发结合起来，可以实现与普通加锁机制一样的同步行为，而这么做却不会阻塞执行异步派发的线程。
* 使用同步队列及栅栏块，可以令同步行为更加高效。
**笔者**
* 用锁来实现同步，会有死锁的风险，而且效率也不是很高。而用 GCD 能以更简单、更高效的形式为代码加锁。
* 用锁实现同步
    * @synchronized
        ```objc
        - (void)synchronizedMethod {
            @synchronized(self) {
                // Safe
            }
        }
        ```
        * 原理：@synchronized 会根据给定的对象，自动创建一个锁，并等待块中的代码执行完毕，然后释放锁。
        * 缺点：滥用 @synchronized(self) 会降低代码效率，因为共用同一个锁的那些同步块，都必须按顺序执行，也就是说所有的 @synchronized(self) 块中的代码之间都同步了。若是在 self 对象上频繁加锁，那么程序可能要等另一段与此无关的代码执行完毕，才能继续执行当前代码，这样做其实并没有必要。
    * NSLock 等
        ```objc
        _lock = [[NSLock alloc] init];

        - (void)synchronizedMethod {
            [_lock lock];
            // Safe
            [_lock unlock];
        }
        ```
* 用 GCD 实现同步
    * GCD 串行同步队列
        <br>将读取操作和写入操作都安排在同一个队列里，即可保证数据同步。
        ```objc
        _syncQueue = dispatch_queue_create("com.effectiveobjectivec.syncQueue", NULL);

        - (NSString *)someString {
            __block NSString *localSomeString;
            dispatch_sync(_syncQueue, ^{
                localSomeString = _someString;
            });
            return localSomeString;
        }

        - (void)setSomeString:(NSString *)someString {
            dispatch_sync(_syncQueue, ^{
                _someString = _someString;
            });
        }
        ```
        也可以将 setter 方法的代码异步执行。由于 getter 方法需要返回值，所以需要同步执行以阻塞线程来防止提前 return，而 setter 方法不需要返回值所以可以异步执行。
        <br>异步执行时需要拷贝 block，所以这里异步执行是否能提高执行速度取决于 block 任务的繁重程度。如果拷贝 block 的时间超过执行 block 的时间，那么异步执行反而降低效率，而如果 block 任务繁重，那么是可以提高执行速度的。
        ```objc
        - (void)setSomeString:(NSString *)someString {
            dispatch_async(_syncQueue, ^{
                _someString = _someString;
            });
        }
        ```
    * GCD 栅栏函数
        <br>以上虽保证了读写安全，但并不是最优方案，因为读取方法之间同步执行了。
        <br>保证读写安全，只需满足三个条件即可：
        1. 同一时间，只能有一个线程进行写操作；
        2. 同一时间，允许有多个线程进行读操作；
        3. 同一时间，不允许既有读操作，又有写操作。
        我们可以针对第二点进行优化，让读取方法可以并发执行。使用 GCD 栅栏函数：
        ```objc
        _syncQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

        - (NSString *)someString {
            __block NSString *localSomeString;
            dispatch_sync(_syncQueue, ^{
                localSomeString = _someString;
            });
            return localSomeString;
        }

        - (void)setSomeString:(NSString *)someString {
            // 这里也可以根据 block 任务繁重程度选择 dispatch_barrier_async
            dispatch_barrier_sync(_syncQueue, ^{ 
                _someString = _someString;
            });
        }
        ```
        ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92baadd4ca5d4da38d24964db32ab4a1~tplv-k3u1fbpfcp-watermark.image)
        关于 GCD 栅栏函数使用的详细说明可以参阅：[Link:《iOS - 关于 GCD 的一些总结》](https://juejin.cn/post/6844904077642432526)。

## 42. 多用 GCD，少用 performSelector 系列方法
**要点**
* performSelector 系列方法在内存管理方面容易有疏失。它无法确定将要执行的选择子具体是什么，因而 ARC 编译器也就无法插入适当的内存管理方法。
* performSelector 系列方法所能处理的选择子太过局限了，选择子的返回值类型及发送给方法的参数个数都受到限制。
* 如果想把任务放在另一个线程上执行，那么最好不要用 performSelector 系列方法，而是应该把任务封装到块里，然后调用大中枢派发机制的相关方法来实现。
* 笔者
* NSObject 定义了几个 performSelector 系列方法，可以让开发者随意调用任何方法，可以推迟执行方法调用，也可以指定执行方法的线程等等。
    ```objc
    - (id)performSelector:(SEL)aSelector;
    - (id)performSelector:(SEL)aSelector withObject:(id)object;
    - (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
    - (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait;
    - (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait;
    - (void)performSelectorInBackground:(SEL)aSelector withObject:(nullable id)arg;
    // ...
    ```
* `performSelector:`方法有什么用处？
    1. 如果你只是用来调用一个方法的话，那么它确实有点多余
    2. 用法一：selector 是在运行期决定的
        ```objc
        SEL selector;
        if ( /* some condition */ ) {
            selector = @selector(foo);
        } else if ( /* some other condition */ ) {
            selector = @selector(bar);
        } else {
            selector = @selector(baz);
        }
        [object performSelector:selector];
        ```
    3. 用法二：把 selector 保存起来等某个事件发生后再调用
* `performSelector:`方法的缺点：
    1. 存在内存泄漏的隐患：
        <br>由于 selector 在运行期才确定，所以编译器不知道所要执行的 selector 是什么。如果在 ARC 下，编译器会给出警告，提示可能会导致内存泄漏。
        ```objc
        waring: PerformSelector may cause a leak because its selector is unknown [-Warc-performSelector-leaks]
        ```
        由于编译器不知道所要执行的 selector 是什么，也就不知道其方法名、方法签名及返回值等，所以就没办法运用 ARC 的内存管理规则来判定返回值是不是应该释放。鉴于此，ARC 采用了比较谨慎的做法，就是不添加释放操作，然而这样可能会导致内存泄漏，因为方法在返回对象时可能已经将其保留了。
        >如果是调用以 `alloc/new/copy/mutableCopy` 开头的方法，创建时就会持有对象，ARC 环境下编译器就会插入 release 方法来释放对象，而使用 performSelector 的话编译器就不添加释放操作，这就导致了内存泄漏。而其他名称开头的方法，返回的对象会被添加到自动释放池中，所以无须插入 release 方法，使用 performSelector 也就不会有问题。
    2. 返回值只能是 void 或对象类型
        <br>如果想返回基本数据类型，就需要执行一些复杂的转换操作，且容易出错；如果返回值类型是 C struct，则不可使用 performSelector 方法。
    3. 参数类型和个数也有局限性
        <br>类型：参数类型必须是 id 类型，不能是基本数据类型；
        <br>个数：所执行的 selector 的参数最多只能有两个。而如果使用 performSelector 延后执行或是指定线程执行的方法，那么 selector 的参数最多只能有一个。
* 使用 GCD 替代 performSelector
    1. 如果要延后执行，可以使用 dispatch_after
    2. 如果要指定线程执行，那么 GCD 也完全可以做到

## 43. 掌握 GCD 及操作队列的使用时机
**要点**
* 在解决多线程与任务管理问题时，派发队列并非唯一方案。
* 操作队列提供了一套高层的 Objective-C API，能实现纯 GCD 所具备的绝大部分功能，而且还能完成一些更为复杂的操作，那些操作若改用 GCD 来实现，则需另外编写代码。
**笔者**
* 在解决多线程与任务管理问题时，我们要根据实际情况使用 GCD 或者 NSOperation，如果选错了工具，则编写的代码就会难以维护。以下是它们的区别：
    多线程方案|区别
    :--|:--
    GCD|GCD 是 iOS4.0 推出的，主要针对多核 CPU 做了优化，是 C 语言的技术；<br>GCD 是纯 C 的 API；<br>GCD 是将任务（block）添加到队列（串行/并发/全局/主队列），并且以同步/异步的方式执行任务；<br>GCD 提供了一些 NSOperation 不具备的功能：<br>&emsp;&emsp;① 队列组<br>&emsp;&emsp;② 一次性执行<br>&emsp;&emsp;③ 延迟执行
    NSOperation|NSOperation 是 iOS2.0 推出的，iOS4 之后重写了 NSOperation，底层由 GCD 实现；<br>NSOperation 是 OC 对象；<br>NSOperation 是将操作（异步的任务）添加到队列（并发队列），就会执行指定操作；<br>NSOperation 里提供的方便的操作：<br>&emsp;&emsp;①  最大并发数<br>&emsp;&emsp;② 队列的暂停/继续/取消操作<br>&emsp;&emsp;③  指定操作之间的依赖关系（GCD 中可以使用同步实现）
* 使用 NSOperation 和 NSOperationQueue 的优势：
    * 取消某个操作
        <br>可以在执行操作之前调用 NSOperation 的 cancel 方法来取消，不过正在执行的操作无法取消。iOS8 以后 GCD 可以用 dispatch_block_cancel 函数取消尚未执行的任务，正在执行的任务同样无法取消。
    * 指定操作间的依赖关系
        <br>使特定的操作必须在另外一个操作顺利执行完以后才能执行。
    * 通过 KVO 监控 NSOperation 对象的属性
        <br>在某个操作任务变更其状态时得到通知，比如 isCancelled、isFinished。而 GCD 不行。
    * 指定操作的优先级
        <br>指定一个操作与队列中其他操作之间的优先级关系，优先级高的操作先执行，优先级低的则后执行。GCD 没有直接实现此功能的办法。
    * 重用 NSOperation 对象
        <br>可以使用系统提供的 NSOperation 子类（比如 NSBlockOperation），也可以自定义子类。
* GCD 任务用块来表示，块是轻量级数据结构，而 NSOperation 则是更为重量级的 Objective-C 对象。虽说如此，但 GCD 并不是最佳方案。有时候采用对象所带来的开销微乎其微，反而它所到来的好处大大反超其缺点。另外，“应该尽可能选用高层 API，只在确有必要时才求助于底层” 这个说法并不绝对。某些功能确实可以用高层的 API 来做，但这并不等于说它就一定比底层实现方案好。要想确定哪种方案更佳，最好还是测试一下性能。

## 44. 通过 Dispatch Group 机制，根据系统资源状况来执行任务
**要点**
* 一系列任务可归入一个 dispatch group 之中。开发者可以在这组任务执行完毕时获得通知。
* 通过 dispatch group，可以在并发式派发队列里同时执行多项任务。此时 GCD 会根据系统资源情况来调度这些并发执行的任务。开发者若自己来实现此功能，则需编写大量代码。
**笔者**
* GCD 队列组，又称 “调度组”，实现所有任务执行完成后有一个统一的回调。
    <br>GCD 有并发队列机制，所以能够根据可用的系统资源状况来并发执行任务，使用队列组，既可以并发执行一系列给定的任务，又能在这些给定的任务全部执行完毕时得到通知。
    <br>有时候我们需要在多个异步任务都并发执行完毕以后再继续执行其他任务，这时候就可以使用队列组。
* 创建一个队列组。
    ```objc
    dispatch_group_t dispatch_group_create(void);
    ```
* 异步执行一个 block，并与指定的队列组关联。
    ```objc
    void dispatch_group_async(dispatch_group_t group, dispatch_queue_t queue, dispatch_block_t block);
    ```
* 也可以通过以下两个函数指定任务所属的队列组。
    <br>需要注意的是，调用了 dispatch_group_enter 之后必须要有与之对应的 dispatch_group_leave，如果没有的话那么这个队列组的任务就永远执行不完。
    ```objc
    void dispatch_group_enter(dispatch_group_t group);
    void dispatch_group_leave(dispatch_group_t group);
    ```
* 同步等待先前 dispatch_group_async 添加的 block 都执行完毕或指定的超时时间结束为止才返回。可以传入 DISPATCH_TIME_FOREVER，表示函数会一直等待任务都执行完，而不会超时。
    >注意：dispatch_group_wait 会阻塞线程
    ```objc
    // @return long 如果 block 在指定的超时时间内完成，则返回 0；超时则返回非 0。
    long dispatch_group_wait(dispatch_group_t group, dispatch_time_t timeout);
    ```
* 等待先前 dispatch_group_async 添加的 block 都执行完毕以后，将 dispatch_group_notify 中的 block 提交到指定队列。
    >dispatch_group_notify 不会阻塞线程
    ```objc
    void dispatch_group_notify(dispatch_group_t group, dispatch_queue_t queue, dispatch_block_t block);
    ```

## 45. 使用 dispatch_once 来执行只需运行一次的线程安全代码
**要点**
* 经常需要编写 “只需执行一次的线程安全代码”（thread-safe single-code execution）。通过 GCD 所提供的 dispatch_once 函数，很容易就能实现此功能。
* 标记应该声明在 static 或 global 作用域中，这样的话，在把只需执行一次的块传给 dispatch_once 函数时，传进去的标记也是相同的。
**笔者**
* dispatch_once 可以用来实现 “只需执行一次的线程安全代码”。
* 使用 dispatch_once 来实现单例，它比 @synchronized 更高效。它没有使用重量级的同步机制，而是采用 “原子访问” 来查询标记，以判断其所对应的代码原来是否已经执行过。
    ```objc
    // 普通单例，线程不安全
    + (id)sharedInstance {
        static EOCClass *sharedInstance = nil;
        if (sharedInstance == nil) {
            sharedInstance = [[self alloc]init];
        }
        return sharedInstance;
    }
    // 加锁，线程安全
    + (id)sharedInstance {
        static EOCClass *sharedInstance = nil;
        @synchronized(self) {
            if (!sharedInstance) {
                sharedInstance = [[self alloc] init];
            }
        }
        return sharedInstance;
    }
    // dispatch_once，线程安全，效率更高
    + (id)sharedInstance {
        static EOCClass *sharedInstance = nil;
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            sharedInstance = [[self alloc] init];
        });
        return sharedInstance;
    }
    ```
* 对于只需执行一次的 block 来说，每次调用函数时传入的标记都必须完全相同，通常标记变量声明在 static 或 global 作用域里。


## 46. 不要使用 dispatch_get_current_queue
**要点**
* dispatch_get_current_queue 函数的行为常常与开发者所预期的不同。此函数已经废弃，只应做调试之用。
* 由于派发队列是按层级来组织的，所以无法单用某个队列对象来描述 “当前队列” 这一概念。
* dispatch_get_current_queue 函数用于解决由不可重入的代码所引发的死锁，然而能用此函数解决的问题，通常也能改用 “队列特定数据” 来解决。
**笔者**
* `dispatch_get_current_queue` 函数返回当前正在执行代码的队列，但从 iOS6.0 开始就已经正式弃用此函数了。我们只应该在调试时使用该函数。
* `dispatch_get_current_queue` 函数有个典型的错误用法，就是用它检测当前队列是不是某个特定的队列，试图以此来避免执行同步派发时可能遭遇的死锁问题，但这样仍然可能会产生死锁。
* 由于队列间有层级关系，所以通过 `dispatch_get_current_queue` 函数来 “检察当前队列是否为执行同步派发所用的队列” 这种办法并不总是奏效。
    >什么是 “队列间的层级关系”？
    >
    >![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/13985ec076d641ae9b8936d62fb2310d~tplv-k3u1fbpfcp-watermark.image)
    ><br>排在某条队列中的块，会在其上层队列（也叫父队列）中执行，层级里地位最高的那个队列总是全局并发队列。
    ><br>上图中，排在队列 B，C 中的块会在 A 里依序执行。于是排在 A、B、C 中的块总是要彼此错开执行。而排在 D 中的块有可能与 A（包括 B、C）的块并行，因为 A 和 D 的目标队列是个并发队列。
    比方说，开发者可能会认为排在队列 C 中的块一定会在 C 中执行，通过 `dispatch_get_current_queue` 函数判断当前队列是 C，就认为在 A 中同步执行该块一定安全，然而该块可能会在队列 A 执行，就导致了死锁。
* `dispatch_get_current_queue` 函数用于解决由不可重入的代码所引发的死锁，然而能用此函数解决的问题，通常也能改用 “队列特定数据” 来解决。
    ```objc
    dispatch_queue_t queueA = dispatch_queue_create("com.effectiveobjectivec.queueA", NULL);
    dispatch_queue_t queueB = dispatch_queue_create("com.effectiveobjectivec.queueB", NULL);
    dispatch_set_target_queue(queueB, queueA); // 将 B 的目标队列设为 A

    static int kQueueSpecific;
    CFStringRef queueSpecificValue = CFSTR("queueA");
    // 给队列 A 设置队列特定数据
    dispatch_queue_set_specific(queueA, 
                        &kQueueSpecific, 
                        (void *)queueSpecificValue, 
                        (dispatch_function_t)CFRelease); 

    dispatch_sync(queueB, ^{
        dispatch_block_t block = ^{ NSLog(@"NO deadlock"); };

        CFStringRef retrievedValue = dispatch_get_specific(&kQueueSpecific);
        // 根据队列特定数据判断，如果当前是队列 A，则直接执行 Block，否则将同步块提交到队列 A
        if (retrievedValue) { 
            block();
        } else {
            dispatch_sync(queueA, block);
        }
    });
    ```
    
# 第七章：系统框架

大家通常会用 Objective-C 来开发 Mac OS X 或 iOS 程序。在这两种情况下都有一套完整的系统框架可供使用，前者名为 Cocoa，后者名为 Cocoa Touch。本章将总览这些框架，并深入研究其中某些类。

## 47. 熟悉系统框架
**要点**
* 许多系统框架都可以直接使用。其中最重要的是 Foundation 与 CoreFoundation，这两个框架提供了构建应用程序所需的许多核心功能。
* 很多常见任务都能用框架来做，例如音频与视频处理、网络通信、数据管理等。
* 请记住：用纯 C 写成的框架与用 Objective-C 写成的一样重要，若想成为优秀的 Objective-C 开发者，应该掌握 C 语言的核心概念。
**笔者**
* 该篇主要介绍了一些系统框架，在编写新的工具类之前，最好查一下系统是否有提供所需功能的框架可直接使用。
    * Foundation 与 CoreFoundation：基础框架
    * UIKit 与 AppKit：核心 UI 框架
    * CFNetwork：提供 C 语言级别的网络通信能力
    * CoreAudio：提供 C 语言 API 用来操作设备上的音频硬件
    * AVFoundation：提供的 Objective-C 对象可用来回放并录制音频和视频
    * CoreData：提供的 Objective-C 接口可将对象放入数据库，便于持久保存
    * CoreText：提供的 C 语言接口可以高效执行文字排版及渲染操作
    * CoreAnimation：用来渲染图形并播放动画
    * CoreGraphics：提供 2D 渲染所必备的数据结构与函数
    * MapKit：提供地图功能
    * Social：提供社交网络功能
* Foundation 与 CoreFoundation 框架分别是用 Objective-C 和 C 实现的，它们之间可以 “无缝桥接”（toll-free bridging）。
* 用 C 实现的 API 可以绕过 Objective-C 的运行期系统，从而提高执行速度。但由于 ARC 只负责 Objective-C 对象，所以使用这些 API 时尤其需要注意内存管理问题。

## 48. 多用块枚举，少用 for 循环 
**要点**
* 遍历 collection 有四种方式。最基本的办法是 for 循环，其次是 NSEnumerator 遍历法及快速遍历法，最新、最先进的方式则是 “块枚举法”。
* “块枚举法” 本身就能够通过 GCD 来并发执行遍历操作，无须另行编写代码。而采用其他遍历方式则无法轻易实现这一点。
* 若提前知道待遍历的 collection 含有何种对象，则应修改块签名，指出对象的具体类型。
**笔者**
* 遍历 collection 有以下四种方式，我们应该多用块枚举，少用 for 循环。
* for 循环
    ```objc
    // Array
    NSArray *anArray = /* ... */;
    for (int i = 0; i < anArray.count; i++) {
        id object = anArray[i];
        // Do something with 'object'
    }

    // Dictionary
    NSDictionary *aDictionary = /* ... */;
    NSArray *keys = [aDictionary allKeys];
    for (int i = 0; i < keys.count; i++) {
        id key = keys[i];
        id value = aDictionary[key];
        // Do something with 'key' and 'value'
    }

    // Set
    NSSet *aSet = /* ... */;
    NSArray *objects = [aSet allObjects];
    for (int i = 0; i < objects.count; i++) {
        id object = objects[i];
        // Do something with 'object'
    }
    ```
    字典和 set 是无序的，所以无法根据下标来访问值，于是需要创建一个有序数组来帮助遍历，但创建和销毁数组却产生了额外的开销。而其他遍历方式则无须创建这种中介数组。
    <br>在执行反向遍历时，使用 for 循环会比其他方式简单许多。
* 使用 Objective-C 1.0 的 NSEumerator 来遍历
    ```objc
    // Array
    NSArray *anArray = /* ... */;
    NSEnumerator *enumerator = [anArray objectEnumerator];
    // NSEnumerator *enumerator = [anArray reverseObjectEnumerator]; //反向遍历
    id object;
    while ((object = [enumerator nextObject]) != nil) {
        // Do something with 'object'
    }

    // Dictionary
    NSDictionary *aDictionary = /* ... */;
    NSEnumerator *enumerator = [aDictionary objectEnumerator];
    id key;
    while ((key = [enumerator nextObject]) != nil) {
        id value = aDictionary[key];
        // Do something with 'object'
    }

    // Set
    NSSet *aSet = /* ... */;
    NSEnumerator *enumerator = [aSet objectEnumerator];
    id object;
    while ((object = [enumerator nextObject]) != nil) {
        // Do something with 'object'
    }
    ```
    nextObject 方法返回枚举（NSEnumerator）里的下个对象。等到枚举中的全部对象都已返回之后，再调用就将返回 nil，表示达到枚举末端，也就是遍历结束了。
    <br>NSEnumerator 相比 for 循环：
    1. 代码多了些，无法获取下标
    2. 优势在于遍历数组、字典、集合的写法都相似
    3. 对于反向遍历，使用 NSEnumerator 来实现，代码读起来更顺畅
* 快速遍历（Objective-C 2.0 引入）
    ```objc
    // Array
    NSArray *anArray = /* ... */;
    for (id object in anArray) {
        // Do something with 'object'
    }

    // Dictionary
    NSDictionary *aDictionary = /* ... */;
    for (id key in aDictionary) {
        id value = aDictionary[key];
        // Do something with 'key' and 'value'
    }

    // Set
    NSSet *aSet = /* ... */;
    for (id object in aSet) {
        // Do something with 'object'
    }

    // 反向遍历，由于 NSEumerator 对象实现了 NSFastEnumeration 协议，故其支持快速遍历
    NSArray *anArray = /* ... */;
    for (id object in [anArray reverseObjectEnumerator]) {
        // Do something with 'object'
    }
    ```
    如果要让某个类的对象支持快速遍历，只需遵守 NSFastEnumeration 协议并实现协议中定义的唯一的方法即可。
    ```objc
    - (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id __unsafe_unretained _Nullable [_Nonnull])buffer count:(NSUInteger)len;
    ```
    快速遍历语法更简洁更高效，它为 for 循环开设了 in 关键字。缺点是无法轻松获取当前遍历的元素的下标。
* “块枚举法” 遍历
    ```objc
    // Array
    NSArray *anArray = /* ... */;
    [anArray enumerateObjectsUsingBlock:^(id object, NSUInteger idx, BOOL *stop){
        // Do something with 'object'
        if (shouldStop) {
            *stop = YES; 
        }
    }];

    // Dictionary
    NSDictionary *aDictionary = /* ... */;
    [aDictionary enumerateKeysAndObjectsUsingBlock:^(id key, id object, BOOL *stop){
        // Do something with 'key' and 'object'
        if (shouldStop) {
            *stop = YES;
        }
    }];

    // Set
    NSSet *aSet = /* ... */;
    [aSet enumerateObjectsUsingBlock:^(id object, BOOL *stop){
        // Do something with 'object'
        if (shouldStop) {
            *stop = YES;
        }
    }];
    ```
    块枚举法拥有其他遍历方式都具备的优势。虽然其代码量较多，但带来了很多好处：
    1. 提供遍历时所针对的下标
    2. 遍历字典时能同时提供键与值，无须额外编码
    3. 可以使用 NSEnumerationOptions 来指定遍历方式，比如 NSEnumerationReverse 反向遍历，NSEnumerationConcurrent 并发执行遍历操作等（底层通过 GCD 来实现）
    4. 可以通过设定 stop 变量值来终止遍历操作，其他遍历方式可以用 break
    5. 能够修改块的方法签名（参数类型），以免去类型转换操作
        <br>之所以能这么做，是因为 id 类型可以被其他类型所覆写。
        <br>指定对象的具体类型，编译器就可以检测出开发者是否调用了该对象所不具备的方法，并在发现这种问题时报错。如果知道 collection 里的对象的具体类型，那就应该使用这种方式指明其类型。
        ```objc
        // 其他遍历方式需要进行类型强转操作
        for (NSString *key in aDictionary) {
            NSString *object = (NSString *)aDictionary[key];
            // Do something with 'key' and 'object'
        }

        // 块枚举法可以直接修改块的方法签名，以免去类型转换操作
        NSDictionary *aDictionary = /* ... */;
        [aDictionary enumerateKeysAndObjectsUsingBlock:^(NSString *key, NSString *obj, BOOL *stop){
            // Do something with 'key' and 'obj'
        }];
        ```

## 49. 对自定义其内存管理语义的 collection 使用无缝桥接
**要点**
* 通过无缝桥接技术，可以在 Foundation 框架中的 Objective-C 对象与 CoreFoundation 框架中的 C 语言数据结构之间来回转换。
* 在 CoreFoundation 层面创建 collection 时，可以指定许多回调函数，这些函数表示此 collection 应如何处理其元素。然后，可运用无缝桥接技术，将其转换成具备特殊内存管理语义的 Objective-C collection。
**笔者**
* 有 `__bridge` 、 `__bridge_retained` 、 `__bridge_transfer` 三种桥接方案，它们的区别为：
     1. `__bridge`：不改变对象的内存管理权所有者。
     2. `__bridge_retained`：用在 Foundation 对象转换成 Core Foundation 对象时，进行 ARC 内存管理权的剥夺。
     3. `__bridge_transfer`：用在 Core Foundation 对象转换成 Foundation 对象时，进行内存管理权的移交。
    详细可以参阅：[Link:《iOS - 老生常谈内存管理（三）：ARC 面世 - Managing Toll-Free Bridging》](https://juejin.im/post/6844904130431942670#heading-32)
* 我们用纯 Objective-C 来编写应用程序，为何要用到 CoreFoundation 框架对象以及桥接技术呢？
    <br>因为 Foundation 框架中的 Objective-C 类所具备的某些功能，是 CoreFoundation 框架中的 C 语言数据结构所不具备的，反之亦然。
* 作者举了一个使用到 CoreFoundation 对象的例子：在使用 Foundation 框架中的字典对象时会遇到一个大问题，其键的内存管理语义为 “拷贝”，而值的语义是 “保留”。也就是说，在向 NSMutableDictionary 中加入键和值时，字典会自动 “拷贝” 键并 “保留” 值。如果用做键的对象不支持拷贝操作（如果要支持，就必须遵守 NSCopying 协议，并实现 copyWithZone: 方法），那么编译器会给出警告并在运行期 Crash：
    ```objc
    NSMutableDictionary *mDict = [NSMutableDictionary dictionary];
    [mDict setObject:@"" forKey:[Person new]]; // ⚠️ warning : Sending 'Person *' to parameter of incompatible type 'id<NSCopying> _Nonnull'

    Runtime:
    *** Terminating app due to uncaught exception 'NSInvalidArgumentException', 
    reason: '-[Person copyWithZone:]: unrecognized selector sent to instance 0x60000230c210'
    ```
    我们是无法直接修改 NSMutableDictionary 的键和值的内存管理语义的。这时候我们可以通过创建 CoreFoundation 框架的 CFMutableDictionary C 数据结构，修改内存管理语义，对键执行 “保留” 而非 “拷贝” 操作，然后再通过无缝桥接技术，将其转换 NSMutableDictionary 对象。
   > 也可以使用 NSMapTable，指定 key 和 value 的内存管理语义。
* 在 CoreFoundation 层面创建 collection 时，可以指定许多回调函数，这些函数表示此 collection 应如何处理其元素。然后，可运用无缝桥接技术，将其转换成具备特殊内存管理语义的 Objective-C collection。
    <br>下面我们就以 CFMutableDictionary 为例，解决上述问题。
    1. 我们先来看一下创建字典的方法定义：
        ```objc
        /**
         * 创建字典
         * @param allocator 表示将要使用的内存分配器（CoreFoundation 对象里的数据结构需要占用内存，而分配器负责分配和回收这些内存）。通常传 NULL，表示采用默认的分配器。
         * @param capacity  字典的初始大小，同 NSMutableDictionary 的 initWithCapacity:
         * @param keyCallBacks
         * @param valueCallBacks  最后两个参数都是指向结构体的指针，它们定义了很多回调函数，用于指示字典中的键和值在遇到各种事件时应该执行何种操作。其定义见如下的 CFDictionaryKeyCallBacks 和 CFDictionaryValueCallBacks
         */
        CFMutableDictionaryRef CFDictionaryCreateMutable(
            CFAllocatorRef allocator, 
            CFIndex capacity, 
            const CFDictionaryKeyCallBacks *keyCallBacks, 
            const CFDictionaryValueCallBacks *valueCallBacks
        ); 

        /** 
         * 键的回调函数
         * @param version  版本号，目前应设为 0
         * @param retain、release、copyDescription、equal、hash  都为函数指针，定义了当各种事件发生时应该采用哪个函数来执行相关任务。比如往字典中添加键值对就会调用 retain 函数，其定义见如下的 CFDictionaryRetainCallBack
         */
        typedef struct {
            CFIndex                version;
            CFDictionaryRetainCallBack        retain;
            CFDictionaryReleaseCallBack        release;
            CFDictionaryCopyDescriptionCallBack    copyDescription;
            CFDictionaryEqualCallBack        equal;
            CFDictionaryHashCallBack        hash;
        } CFDictionaryKeyCallBacks;

        /** 
         * 值的回调函数
         * 同 CFDictionaryKeyCallBacks
         */
        typedef struct {
            CFIndex                version;
            CFDictionaryRetainCallBack        retain;
            CFDictionaryReleaseCallBack        release;
            CFDictionaryCopyDescriptionCallBack    copyDescription;
            CFDictionaryEqualCallBack        equal;
        } CFDictionaryValueCallBacks;

        /** 
         * retain 函数定义
         * @param allocator 
         * @param value  即将加入字典中的键
         * @return void*  表示要加到字典里的最终值
         */
        typedef const void *(*CFDictionaryRetainCallBack)(CFAllocatorRef allocator, const void *value);
        ```
    2. 示例代码，实现对键执行 “保留” 而非 “拷贝” 操作的字典 NSMutableDictionary：
        ```objc
        #import <Foundation/Foundation.h>
        #import <CoreFoundation/CoreFoundation.h>

        const void * EOCRetainCallBack(CFAllocatorRef allocator, const void *value) {
            return CFRetain(value);  // 将加入字典中的键或值 retain 后返回，而不进行 copy
        }

        void EOCReleaseCallBack(CFAllocatorRef allocator, const void *value) {
            CFRelease(value);
        }

        CFDictionaryKeyCallBacks keyCallBacks = {
            0,
            EOCRetainCallBack,
            EOCReleaseCallBack,
            NULL,    // copyDescription 传 NULL，代表采用默认实现
            CFEqual, // CFEqual 最终会调用 NSObject 的 isEqual: 方法
            CFHash   // CFHash  最终会调用 NSObject 的 hash 方法
        };

        CFDictionaryValueCallBacks valueCallBacks = {
            0,
            EOCRetainCallBack,
            EOCReleaseCallBack,
            NULL,
            CFEqual
        };

        CFMutableDictionaryRef aCFDictionary = CFDictionaryCreateMutable(
            NULL,
            0,
            &keyCallBacks,
            &valueCallBacks
        );
        // 进行无缝桥接
        NSMutableDictionary *aNSDictionary = (__bridge_transfer NSMutableDictionary *)aCFDictionary;
        ```

 
## 50. 构建缓存时选用 NSCache 而非 NSDictionary
**要点**
* 实现缓存时应选用 NSCache 而非 NSDictionary 对象。因为 NSCache 可以提供优雅的自动删减功能，而且是 “线程安全” 的，此外，它与字典不同，并不会拷贝键。
* 可以给 NSCache 对象设置上限，用以限制缓存中的对象总个数及 “总成本”，而这些尺度则定义了缓存删减其中的对象的时机。但绝对不要把这些尺度当成可靠的 “硬限制”（hard limit），它们仅对 NSCache 起指导作用。
* 将 NSPurgeableData 与 NSCache 搭配使用，可实现自动清除数据的功能，也就是说，当 NSPurgeableData 对象所占内存为系统所丢弃时，该对象自身也会从缓存中移除。
* 如果缓存使用得当，那么应用程序的响应速度就能提高。只有那种 “重新计算起来很费事的” 数据，才值得放入缓存，比如那些需要从网络获取或从磁盘读取的数据。
**笔者**
* 构建缓存时选用 NSCache 而非 NSDictionary，NSCache 的优势在于：
    1. 当系统资源将要耗尽时，它可以优雅的自动删减缓存，且会先行删减最久未使用的缓存。使用 NSDictionary 虽也可以自己实现但很复杂。
    2. NSCache 不会拷贝键，而是保留它。使用 NSDictionary 虽也可以实现但比较复杂，见 【🚩 49】。
    3. NSCache 是线程安全的。不编写加锁代码的前提下，多个线程可以同时访问 NSCache。而 NSDictionary 不是线程安全的。
* 可以操控 NSCache 删减缓存的时机
    1. `totalCostLimit` 限制缓存中所有对象的总开销
    2. `countLimit` 限制缓存中对象的总个数
    3. `- (void)setObject:(ObjectType)obj forKey:(KeyType)key cost:(NSUInteger)g;` 将对象添加进缓存时，可指定其开销值
    可能会删减缓存对象的时机：
    1. 当对象总数或者总开销超过上限时
    2. 在可用的系统资源趋于紧张时
    需要注意的是：
    1. 可能会删减某个对象，并不意味着一定会删减
    2. 删减对象的顺序，由具体实现决定的
    3. 想通过调整开销值来迫使缓存优先删减某对象是不建议的，绝对不要把这些尺度当成可靠的 “硬限制”，它们仅对 NSCache 起指导作用。
* 使用 `- (void)setObject:(ObjectType)obj forKey:(KeyType)key cost:(NSUInteger)g;` 可以在将对象添加进缓存时指定其开销值。但这种情况只适用于开销值能很快计算出来的情况，因为缓存的本意就是为了增加应用程序响应用户操作的速度。
    1. 比方说，计算开销值时必须访问磁盘或者数据库才能确定文件大小，那么就不适用这种方法。
    2. 如果要加入缓存的是 NSData 对象，其数据大小已知，直接访问属性即可 `data.length`。
* NSPurgeableData 
    1. NSPurgeableData 继承自 NSMutableData，它与 NSCache 搭配使用，可实现自动清除数据的功能。它实现了 NSDiscardableContent 协议（如果某个对象所占内存能够根据数据需要随时丢弃，就可以实现该协议定义的接口），将其加入 NSCache 后当该对象被系统所丢弃时，也会自动从缓存中清除。可以通过 NSCache 的 evictsObjectWithDiscardedContent 属性来开启或关闭此功能。
    2. 使用 NSPurgeableData 的方式和 “引用计数” 很像，当需要访问某个 NSPurgeableData 对象时，可以调用 `beginContentAccess` 进行 “持有”，并在用完时调用 `endContentAccess` 进行 “释放”。NSPurgeableData 在创建的时候其 “引用计数” 就为 1，所以无须调用 `beginContentAccess`，只需要在使用完毕后调用 `endContentAccess` 就行。
        * `beginContentAccess`：告诉它现在还不应该丢弃自己所占据的内存
        * `endContentAccess`：告诉它必要时可以丢弃自己所占据的内存
* NSPurgeableData 与 NSCache 一起实现缓存的代码示例：
    ```objc
    // Network fetcher class
    typedef void(^EOCNetworkFetcherCompletionHandler)(NSData *data);

    @interface EOCNetworkFetcher : NSObject

    - (id)initWithURL:(NSURL*)url;
    - (void)startWithCompletionHandler:(EOCNetworkFetcherCompletionHandler)handler;

    @end

    // Class that uses the network fetcher and caches results
    @interface EOCClass : NSObject
    @end

    @implementation EOCClass {
        NSCache *_cache;
    }

    - (id)init {

        if ((self = [super init])) {
            _cache = [NSCache new];

            // Cache a maximum of 100 URLs
            _cache.countLimit = 100;

            /**
             * The size in bytes of data is used as the cost,
             * so this sets a cost limit of 5MB.
             */
            _cache.totalCostLimit = 5 * 1024 * 1024;
        }
        return self;
    }

    - (void)downloadDataForURL:(NSURL*)url { 

        NSPurgeableData *cachedData = [_cache objectForKey:url];

        if (cachedData) {
            [cachedData beginContentAccess];
            // Cache hit：存在缓存，读取
            [self useData:cachedData];
            [cachedData endContentAccess];
        } else {
            // Cache miss：没有缓存，下载
            EOCNetworkFetcher *fetcher = [[EOCNetworkFetcher alloc] initWithURL:url];      

            [fetcher startWithCompletionHandler:^(NSData *data){
                NSPurgeableData *purgeableData = [NSPurgeableData dataWithData:data];
                [_cache setObject:purgeableData forKey:url cost:purgeableData.length];    
                [self useData:data];
                [purgeableData endContentAccess];
            }];
        }
    }
    @end
    ```

## 51. 精简 initialize 与 load 的实现代码
**要点**
* 在加载阶段，如果类实现了 load 方法，那么系统就会调用它。分类里也可以定义此方法，类的 load 方法要比分类中的先调用。与其他方法不同，load 方法不参与覆写机制。
* 首次使用某个类之前，系统会向其发送 initialize 消息。由于此方法遵从普通的覆写规则，所以通常应该在里面判断当前要初始化的是哪个类。
* load 与 initialize 方法都应该实现得精简一些，这有助于保持应用程序的响应能力，也能减少引入 “依赖环”（interdependency cycle，也称 “环状依赖”）的几率。
* 无法在编译期设定的全局常量，可以放在 initialize 方法里初始化。
**笔者**
* 掌握 load 和 initialize 方法的调用时刻、调用方式、调用顺序等。
    <br>可以参阅 [Link:《OC 底层探索 - load 和 initialize》](https://juejin.im/post/6844904068071194638)
* 使用 load 方法的问题和注意事项：
    1. 在 load 方法中使用其他类是不安全的。比方说，类 A 和 B 没有继承关系，它们之间 load 方法的执行顺序是不确定的，而你在类 A 的 load 方法中去实例化 B，而类 B 可能会在其 load 方法中去完成实例化 B 前的一些重要操作，此时类 B 的 load 方法可能还未执行，所以不安全。
    2. load 方法务必实现得精简一些，尽量减少其所执行的操作，不要执行耗时太久或需要加锁的任务，因为整个应用程序在执行 load 方法时都会阻塞。
    3. 如果任务没必要在类加载进内存时就执行，而是可以在类初始化时执行，那么改用 initialize 替代 load 方法。
* initialize 除了在调用时刻、调用方式、调用顺序方面与 load 有区别以外。initialize 方法还是安全的。
    <br>运行期系统在执行 initialize 时，是处于正常状态的，因为这时候可以安全使用并调用任意类中的任意方法了。而且运行期系统也能确保 initialize 方法一定会在 “线程安全的环境” 中执行，只有执行 initialize 的那个线程可以操作类或类实例，其他线程都要先阻塞等着 initialize 执行完。
* 如果子类没有实现 initialize 方法，那么就会调用父类的，所以通常会在 initialize 实现中对消息接收者做一下判断：
    ```objc
    + (void)initialize {
        if (self == [EOCBaseClass class]) {
            NSLog(@"%@ initialized", self);
        }
    }
    ```
* initialize 的实现也要保持精简，其原因在于：
    1. 如果在主线程初始化一个类，那么初始化期间就会一直阻塞。
    2. 无法控制类的初始化时机。编写代码时不能令代码依赖特定的时间点执行，否则如果以后运行期系统更新改变了类的初始化方式，那么就会很危险。
    3. 如果在 initialize 中给其他类发送消息，那么会迫使这些类都进行初始化。如果其他类在执行 initialize 时又依赖该类的某些数据，而该类的这些数据又在 initialize 中完成，就会发生问题，产生 “环状依赖”。
        <br>所以，initialize 方法只应该用来设置内部数据，例如无法在编译期设定的全局常量，可以放在 initialize 方法里初始化。不应该调用其他方法，即便是本类自己的方法，也最好别调用。
* 实现 load 和 initialize 方法时，一定要注意以上问题，精简代码。除了初始化全局状态之外，如果还有其他事情要做，那么可以专门创建一个方法来执行这些操作，并要求该类的使用者必须在使用本类之前调用此方法。比如说，如果 “单例类” 在首次使用之前必须执行一些操作，那就可以采用这个办法。

## 52. 别忘了 NSTimer 会保留其目标对象
**要点**
* NSTimer 对象会保留其目标，直到计时器本身失效为止，调用 invalidate 方法可令计时器失效，另外，一次性的计时器在触发完任务之后也会失效。
* 反复执行任务的计时器（repeating timer），很容易引入保留环，如果这种计时器的目标对象又保留了计时器本身，那么肯定会导致保留环。这种环状保留关系，可能是直接发生的，也可能是通过对象图里的其他对象间接发生的。
* 可以扩充 NSTimer 的功能，用 “块” 来打破保留环。不过，除非 NSTimer 将来在公共接口里提供此功能，否则必须创建分类，将相关实现代码加入其中。（笔者注：NSTimer 现已提供此功能）
**笔者**
* 如果使用传 target 参数的方法来创建定时器，要注意定时器是会保留 target 的。如果使用不当，很容易造成内存泄露。
* 以下代码中，EOCClass 实例和 _pollTimer 之间形成了保留环，断环的方式有两种。
    1. 该实例被系统回收，然后在 dealloc 方法中令定时器实效。然而由于定时器保留了该实例，所以在定时器实效前该实例是不会回收的，所以这个办法不可行，必定造成内存泄漏。
    2. 主动调用 stopPolling 方法打破保留环，但这种通过主动调用某方法来避免内存泄露的做法，也不是个好主意。
        ```objc
        #import <Foundation/Foundation.h>

        @interface EOCClass : NSObject
        - (void)startPolling;
        - (void)stopPolling;
        @end

        @implementation EOCClass {
            NSTimer *_pollTimer;
        }

        - (id)init {
            return [super init];
        }

        - (void)dealloc {
            [_pollTimer invalidate];
        }

        - (void)stopPolling {
            [_pollTimer invalidate];
            _pollTimer = nil;
        }

        - (void)startPolling {
            _pollTimer = [NSTimer scheduledTimerWithTimeInterval:5.0
                                                          target:self
                                                        selector:@selector(p_doPoll)
                                                        userInfo:nil
                                                         repeats:YES];
        }

        - (void)p_doPoll {
            // Poll the resource
        }

        @end
        ```
* 解决这个问题的方法有很多种，比如：
    1. 在 viewWillDisappear 方法中令定时器实效。但是这种方式不推荐，原因是 push 的时候 vc 并没有销毁，但也会调用该方法。以下代码也有局限性。
        ```objc
        - (void)viewWillDisappear:(BOOL)animated {
            [super viewWillDisappear:animated];
            if ([self.navigationController.viewControllers indexOfObject:self] == NSNotFound) {
                [self.timer invalidate];
                self.timer = nil;
            }
        }
        ```
    2. didMoveToParentViewController。缺点是不能用于多个控制器情况。
        ```objc
        - (void)didMoveToParentViewController:(UIViewController *)parent {
            if (parent == nil) {
                [self.timer invalidate];
                self.timer = nil;
            }
        }
        ```
    3. 使用 block 的方式创建 timer
        ```objc
        __weak typeof(self) weakSelf = self;
        self.timer = [NSTimer scheduledTimerWithTimeInterval:5.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
            [weakSelf message];
        }];
        ```
        然后在 self 的析构方法中令定时器实效即可。
    4. 封装一层 NSTimer
    5. 使用中间变量 NSProxy
        1. timer 对 proxy 持有强引用
        2. proxy 对 self 持有弱引用，并将消息转发给 self
        3. 在 self 的析构方法中正常释放定时器
* 作者给出的方式是给 NSTimer 封装一个以 block 方式创建的方法，将 NSTimer 类对象作为 target 传入，因为类对象是个单例，所以定时器保留它也无所谓。由于现在苹果已经提供了通过传 block 方式创建 NSTimer 的方法，故没必要再像该作者这样再自己封装一个方法，这里也就不贴代码了。
