

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c47e8aa85838419f8c9b3fff83a9e174~tplv-k3u1fbpfcp-zoom-1.image)


# 1. GCD 初识
## 1.1 GCD 介绍
* 全称是 Grand Central Dispatch，也简称 Dispatch；
* 纯 C 语言，提供了非常多强大的函数；
* GCD 是苹果公司为多核的并行运算提出的解决方案；
* GCD 会自动充分利用设备的多核（比如双核、四核）；
* GCD 会自动管理线程的生命周期（创建线程、调度任务、销毁线程）；
* 开发者只需要告诉 GCD 想要执行什么任务，不需要编写任何线程管理代码。

## 1.2 GCD 的使用步骤

### GCD 的两个核心
* 任务：执行什么操作
* 队列：用来存放任务
### GCD 的任务
GCD 中的任务有两种封装：dispatch_block_t 和 dispatch_function_t。

#### <font color=red>●</font> dispatch_block_t（常用）
提交给指定队列的 block，无参无返回值。
```objc
typedef void (^dispatch_block_t)(void);
```
#### <font color=red>●</font> dispatch_function_t
提交给指定队列的 function，`void(*)()`类型的函数指针。
```objc
typedef void (*dispatch_function_t)(void *);
```

### GCD 的使用步骤
1. 创建/获取队列：创建/获取一个并发/串行队列；
2. 创建任务：确定要做的事；
3. 将任务添加进队列中（同时指定任务的执行方式）：

&emsp;&emsp;GCD 会自动将队列中的任务取出，放到对应的线程中执行；<br>
&emsp;&emsp;任务的取出遵循队列的 FIFO 原则：先进先出，后进后出；<br>
&emsp;&emsp;GCD 中，要执行队列中的任务时，会自动开启一个线程，当任务执行完，线程不会立刻销毁，而是放到了线程池中。如果接下来还要执行任务的话就从线程池中取出线程，这样节省了创建线程所需要的时间。但如果一段时间内没有执行任务的话，该线程就会被销毁，再执行任务就会创建新的线程。


![队列 FIFO 原则](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa0707ff135c48a4b46f0eb4b128b2a9~tplv-k3u1fbpfcp-zoom-1.image)

```objc
    // 1.创建一个队列
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    // 2.创建一个任务
    dispatch_block_t block = ^{
        NSLog(@"%@",[NSThread currentThread]);
    };
    // 3.将任务添加进队列中（同时指定任务的执行方式）  
    dispatch_async(queue, block);
```



## 1.3 GCD 执行任务的方式
### 1.3.1 同步

#### <font color=red>●</font> dispatch_sync
提交一个 block 对象到指定队列以同步执行，并在该 block 完成执行后返回（阻塞）。
（因为这个特性，使用该函数要注意死锁的问题，后面会讲到）

```objc
/*!
 * @param queue
 * 提交block的队列，这个队列会被系统retain直到block运行完成；
 * 此参数不能为空（NULL）
 *
 * @param block
 * 要执行的block，block会被自动copy与release；
 * 该block没有返回值，也没有参数；
 * 此参数不能为空（NULL）
 */
void dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
```
```objc
- (void)test
{
    dispatch_queue_t queue = dispatch_queue_create("com.junteng.queue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"0");
    dispatch_sync(queue, ^{
        NSLog(@"1");
    });
    dispatch_sync(queue, ^{
        NSLog(@"2");
    });
    NSLog(@"3");
}
/*
2020-01-31 20:35:48.958272+0800 多线程[4653:706706] 0
2020-01-31 20:35:48.958533+0800 多线程[4653:706706] 1
2020-01-31 20:35:48.958696+0800 多线程[4653:706706] 2
2020-01-31 20:35:48.958810+0800 多线程[4653:706706] 3
 */
```
#### <font color=red>●</font> dispatch_sync_f
提交一个 function 到指定队列以同步执行，并在该 function 完成执行后返回（阻塞）。
```objc
/*!
 * @param queue
 * 提交函数的队列，这个队列会被系统retain直到block运行完成；
 * 此参数不能为空（NULL）
 *
 * @param context
 * 传递给函数的参数，即work的参数
 * 
 * @param work
 * 要执行的函数；
 * 此参数不能为空（NULL）
 */
void dispatch_sync_f(dispatch_queue_t queue, void *context, dispatch_function_t work);
```
```objc
- (void)test
{
    dispatch_queue_t queue = dispatch_queue_create("com.junteng.queue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"0");
    dispatch_sync_f(queue, NULL, testFunc);
    NSLog(@"2");
}

void testFunc() {
    NSLog(@"1");
}
/*
2020-01-31 21:05:35.017838+0800 多线程[4757:726399] 0
2020-01-31 21:05:35.017959+0800 多线程[4757:726399] 1
2020-01-31 21:05:35.018047+0800 多线程[4757:726399] 2
 */
```

### 1.3.2 异步
#### <font color=red>●</font> dispatch_async
提交一个 block 对象到指定队列以异步执行，并直接返回（不会阻塞）。
```objc
void dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```
```objc
- (void)test
{
    dispatch_queue_t queue = dispatch_queue_create("com.junteng.queue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"0");
    dispatch_async(queue, ^{
        NSLog(@"1");
    });
    dispatch_async(queue, ^{
        NSLog(@"2");
    });
    NSLog(@"3");
}
/*
2020-01-31 21:09:43.675233+0800 多线程[4801:730375] 0
2020-01-31 21:09:43.675389+0800 多线程[4801:730375] 3
2020-01-31 21:09:43.675458+0800 多线程[4801:730469] 1
2020-01-31 21:09:43.675550+0800 多线程[4801:730469] 2
 */
```
#### <font color=red>●</font> dispatch_async_f
道理同 dispatch_sync_f，不再赘述。

### 1.3.3 同步和异步的区别
* 同步：必须等待当前语句执行完毕，才会执行下一条语句（阻塞）；
	<br>&emsp;&emsp;&emsp;在`当前`线程中执行任务，`不具备`开启新线程的能力。
* 异步：不用等待当前语句执行完毕，就可以执行下一条语句（不会阻塞）；
	<br>&emsp;&emsp;&emsp;在`新的`线程中执行任务，`具备`开启新线程的能力。
	<br>&emsp;&emsp;&emsp;（具备开启新线程的能力，不代表一定能开启新线程。如在主队列异步执行，不会开启新线程，因为主队列的任务在主线程上执行）


## 1.4 GCD 的队列

### 1.4.1 GCD 队列介绍
**Dispatch Queue：** 一个用于管理主线程或子线程上串行或并发执行的任务的对象。<br>
调度队列是 FIFO 队列，您可以以 block 对象的形式向其提交任务。调度队列可以串行或并发执行任务。提交给调度队列的任务在系统管理的线程池上执行。除了主队列在主线程上执行以外，系统无法保证它使用哪个线程来执行任务。


### 1.4.2 GCD 队列类型
* **串行队列**（`DISPATCH _QUEUE _SERIAL`）<br>
以 FIFO 顺序处理传入的任务，即让任务一个接着一个执行。
* **并发队列**（`DISPATCH _QUEUE _CONCURRENT`）<br>
可以让多个任务并发（同时）执行（自动开启多个线程执行任务）；<br>
并发功能只有在异步函数`dispatch_async`下才有效；<br>
尽管任务同时执行，但是您可以使用 barrier 栅栏函数在队列中创建同步点（关于栅栏函数后面会讲到）。
* **主队列**（`dispatch_queue_main_t`）<br>
主队列是一种特殊的串行队列，它特殊在与主线程关联，主队列的任务都在主线程上执行，主队列在程序一开始就被系统创建并与主线程关联。
#### <font color=red>●</font> dispatch_get_main_queue
```objc
// @return 主队列
dispatch_queue_main_t dispatch_get_main_queue(void); 
```
&emsp;&emsp;系统创建主队列并与主线程进行关联的时机：<br>
&emsp;&emsp;&emsp;① 调用 dispatch_main()；<br>
&emsp;&emsp;&emsp;② 调用 UIApplicationMain（iOS）或者 NSApplicationMain（macOS）；<br>
&emsp;&emsp;&emsp;③ 在主线程使用 CFRunLoopRef。<br>
&emsp;&emsp;大多数情况下我们的应用程序会在 main() 函数里使用第 2 种方式。

* **全局并发队列**（`dispatch_queue_global_t`）<br>
一种特殊的并发队列，可以指定服务质量（服务质量有助于确定队列执行的任务的优先级）。
#### <font color=red>●</font> dispatch_get_global_queue
```objc
/*!
 * @param identifier
 * 队列的服务质量，传0就是默认
 *
 * @param flags
 * 苹果留着以后用的，传0就行
 * 
 * @return dispatch_queue_global_t
 * 可以指定服务质量的系统定义的全局并发队列
 */
dispatch_queue_global_t dispatch_get_global_queue(long identifier, unsigned long flags);
```
**注意：** 对主队列和全局并发队列使用`dispatch_suspend`、`dispatch_resume`、`dispatch_set_context`是无效的。

#### 全局并发队列与手动创建的并发队列的区别：

1. 手动创建的并发队列可以设置唯一标识，可以跟踪错误，而全局并发队列没有；
2. 在 ARC 中不需要考虑释放内存，`dispatch_release(q);`不需要也不允许调用。而在 MRC 中由于手动创建的并发队列是 create 出来的，所以需要调用`dispatch_release(q);`来释放内存，而全局并发队列不需要;
3. 全局并发队列可以指定服务质量（服务质量有助于确定队列执行的任务的优先级）;
4. 一般我们使用全局并发队列。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/049a60d4fa2a48a79533cffe40ce5d78~tplv-k3u1fbpfcp-zoom-1.image)

#### <font color=red>●</font> dispatch_queue_t（队列）
&emsp;&emsp;应用程序向其提交块（任务）以进行后续执行的轻量级对象，它是一个对象，这也很好的解释了在 MRC 下为何要手动管理`dispatch_queue_t`的内存。<br>
&emsp;&emsp;队列遵循 FIFO 原则。串行队列一次只能调用一个块，但是不同队列可以各自相对于彼此同时调用它们的块。并发队列也是按 FIFO 顺序调用块，但不等待它们完成，从而允许并发调用多个块。<br> 
&emsp;&emsp;系统管理一个线程池，该线程池处理队列并调用提交给它们的块。<br> 
&emsp;&emsp;队列是通过调用`dispatch_retain`和`dispatch_release `来进行引用计数的。提交到队列的待处理块也保留对该队列的引用，直到它们完成为止。一旦释放了对队列的所有引用，系统将重新分配该队列。
```objc
typedef NSObject<OS_dispatch_queue> *dispatch_queue_t;
```


#### <font color=red>●</font> dispatch_queue_create 
创建队列。
```objc
/*!
 * @param label
 * 给队列一个字符串标签进行唯一标识，以便在调试时区分队列
 * 建议使用反向DNS命名方式（com.example.myqueue）
 * 该参数可以为空（NULL）
 *
 * @param attr
 * 指定队列类型
 * DISPATCH_QUEUE_SERIAL     为串行队列
 * DISPATCH_QUEUE_CONCURRENT 为并发队列
 * 该参数可以为空（NULL），传空时默认为串行队列（在iOS4.3版本之前该参数只能传空）
 * 
 * @return dispatch_queue_t
 * 新创建的队列
 */
dispatch_queue_t dispatch_queue_create(const char *label, dispatch_queue_attr_t attr);
```

#### 创建/获取一个队列：
```objc
// 创建一个串行队列 
dispatch_queue_t queue = dispatch_queue_create("com.junteng.myqueue", DISPATCH_QUEUE_SERIAL);
// 创建一个并发队列
dispatch_queue_t queue = dispatch_queue_create("com.junteng.myqueue", DISPATCH_QUEUE_CONCURRENT);
// 获取主队列
dispatch_queue_t queue = dispatch_get_main_queue();
// 获取全局并发队列
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
```


#### <font color=red>●</font> dispatch_queue_get_label 
获取队列的唯一标识 label。
```objc
/*!
 * @param queue
 * 需要获取label的队列;
 * 如果需要获取当前队列的label则使用 DISPATCH_CURRENT_QUEUE_LABEL
 * 
 * @return 
 * 创建队列时给队列设置的标签
 */
const char * dispatch_queue_get_label(dispatch_queue_t queue);
```
```objc
    dispatch_queue_t queue = dispatch_queue_create("com.junteng.myqueue", NULL);
    dispatch_sync(queue, ^{
        NSLog(@"%s", dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
    });
    NSLog(@"%s", dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
    NSLog(@"%s", dispatch_queue_get_label(queue));
/*
com.junteng.myqueue
com.apple.main-thread
com.junteng.myqueue
 */
```


### 1.4.3 GCD 各种队列的执行效果



执行方式|并发队列|手动创建的串行队列|主队列
:--:|:--:|:--:|:--:
同步（sync）|`没有`开启新线程<br>`串行`执行任务|`没有`开启新线程<br>`串行`执行任务|`没有`开启新线程<br>`串行`执行任务
异步（async）|`有`开启新线程<br>`并发`执行任务|`有`开启新线程<br>`串行`执行任务|`没有`开启新线程<br>`串行`执行任务




```objc
// 同步并发
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 5; i++) {
        dispatch_sync(queue, ^{
            NSLog(@"%@",[NSThread currentThread]);
        });
    }
/*
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
 */

// 同步串行（手动创建的串行队列）
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    for (int i = 0; i < 5; i++) {
        dispatch_sync(queue, ^{
            NSLog(@"%@",[NSThread currentThread]);
        });
    }
/*
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
 */

// 同步串行（主队列）
    dispatch_queue_t queue = dispatch_get_main_queue();
    dispatch_queue_t serialQueue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    dispatch_async(serialQueue, ^{
        for (int i = 0; i < 5; i++) {
            dispatch_sync(queue, ^{
                NSLog(@"%@",[NSThread currentThread]);
            });
        }
    });
/*
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
<NSThread: 0x600001e6cbc0>{number = 1, name = main}
 */

// 异步并发：开多个线程，线程数由 GCD 决定
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 5; i++) {
        dispatch_async(queue, ^{
            NSLog(@"%@",[NSThread currentThread]);
        });
    }
/*
<NSThread: 0x600001ee53c0>{number = 8, name = (null)}
<NSThread: 0x600001ee57c0>{number = 9, name = (null)}
<NSThread: 0x600001e16a80>{number = 10, name = (null)}
<NSThread: 0x600001e17500>{number = 11, name = (null)}
<NSThread: 0x600001ee53c0>{number = 8, name = (null)}
 */

// 异步串行（手动创建的串行队列）
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    for (int i = 0; i < 5; i++) {
        dispatch_async(queue, ^{
            NSLog(@"%@",[NSThread currentThread]);
        });
    }
/*
<NSThread: 0x600001ee53c0>{number = 8, name = (null)}
<NSThread: 0x600001ee53c0>{number = 8, name = (null)}
<NSThread: 0x600001ee53c0>{number = 8, name = (null)}
<NSThread: 0x600001ee53c0>{number = 8, name = (null)}
<NSThread: 0x600001ee53c0>{number = 8, name = (null)}
 */

// 异步串行（主队列）
    dispatch_queue_t queue = dispatch_get_main_queue();
    for (int i = 0; i < 5; i++) {
        dispatch_async(queue, ^{
            NSLog(@"%@",[NSThread currentThread]);
        });
    }
/*
<NSThread: 0x600002658680>{number = 1, name = main}
<NSThread: 0x600002658680>{number = 1, name = main}
<NSThread: 0x600002658680>{number = 1, name = main}
<NSThread: 0x600002658680>{number = 1, name = main}
<NSThread: 0x600002658680>{number = 1, name = main}
 */
```


## 1.5 死锁
### 1.5.1 死锁的四大条件

1. 互斥：某种资源一次只允许一个进程访问，即该资源一旦分配给某个进程，其他进程就不能再访问，直到该进程访问结束。
2. 占有且等待：一个进程本身占有资源（一种或多种），同时还有资源未得到满足，正在等待其他进程释放该资源。
3. 不可抢占：别人已经占有了某项资源，你不能因为自己也需要该资源，就去把别人的资源抢过来。
4. 循环等待：存在一个进程链，使得每个进程都占有下一个进程所需的至少一种资源。

### 1.5.2 GCD 中的死锁
* **死锁情况**：
使用`dispatch_sync`函数往`当前串行队列`中添加任务，会卡住当前的串行队列（产生死锁）。
* **死锁原因**：
队列引起的循环等待。
* **示例1**：
```objc
/*
 队列的特点：FIFO (First In First Out) 先进先出
 以下将 block（任务2）提交到主队列，主队列将来要取出这个任务放到主线程执行。
 而主队列此时已经有任务，就是执行（viewDidLoad方法），
 所以主队列要想取出 block（任务2），就要等上一个任务（viewDidLoad方法）先执行完，才能取出该任务执行。
 而 dispatch_sync 函数必须执行完 block（任务2）才会返回，才能往下执行代码。
 所以（任务2）要等待（viewDidLoad方法）执行完，（viewDidLoad方法）要等待（任务2）执行完。互相等待，就产生了死锁。
 */
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"执行任务1");

    dispatch_queue_t queue = dispatch_get_main_queue();
    dispatch_sync(queue, ^{
        NSLog(@"执行任务2");
    });

    NSLog(@"执行任务3");
}
/*
 打印：
 2020-01-19 00:16:26.980630+0800 多线程[25011:5507937] 执行任务1
 (lldb) 
 */

/*
 解决方案：打破（使用`dispatch_sync`函数往`当前串行队列`中添加任务）这一条件即可
 以下将（任务2）异步执行，打印结果为：132
 */
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"执行任务1");

    dispatch_queue_t queue = dispatch_get_main_queue();
    dispatch_async(queue, ^{
        NSLog(@"执行任务2");
    });

    NSLog(@"执行任务3");
}
/*
 打印：
 2020-01-19 03:16:47.472682+0800 多线程[25416:5603048] 执行任务1
 2020-01-19 03:16:47.472890+0800 多线程[25416:5603048] 执行任务3
 2020-01-19 03:16:47.474389+0800 多线程[25416:5603048] 执行任务2
 */
```
* **示例2**：
```objc
/*
 block0（任务2）和 block1（任务3）都添加到串行队列里去，
 由于队列任务先进先出，在当前子线程执行 block1 必须要先执行完 block0
 而 block0 执行完的前提是 sync 的 block1（任务3）要执行完，才能执行（任务4）
 所以产生了死锁
 */
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"执行任务1");
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue, ^{
        NSLog(@"执行任务2");
        dispatch_sync(queue, ^{
            NSLog(@"执行任务3");
        });
        NSLog(@"执行任务4");
    });
    NSLog(@"执行任务5");
}
/*
 打印：
 2020-01-19 02:55:20.608987+0800 多线程[25339:5586331] 执行任务1
 2020-01-19 02:55:20.609307+0800 多线程[25339:5586331] 执行任务5
 2020-01-19 02:55:20.609446+0800 多线程[25339:5586387] 执行任务2
 (lldb) 
 */

/*
 解决方案：打破（使用`dispatch_sync`函数往`当前串行队列`中添加任务）这一条件即可
 1.以下将（任务3）异步执行，打印结果为：15243
 */
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"执行任务1");
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue, ^{
        NSLog(@"执行任务2");
        dispatch_async(queue, ^{
            NSLog(@"执行任务3");
        });
        NSLog(@"执行任务4");
    });
    NSLog(@"执行任务5");
}
/*
 2020-01-19 03:25:52.761192+0800 多线程[25474:5609516] 执行任务1
 2020-01-19 03:25:52.761393+0800 多线程[25474:5609516] 执行任务5
 2020-01-19 03:25:52.761429+0800 多线程[25474:5609578] 执行任务2
 2020-01-19 03:25:52.761584+0800 多线程[25474:5609578] 执行任务4
 2020-01-19 03:25:52.761749+0800 多线程[25474:5609578] 执行任务3
 */
/*
 2.以下将（任务3）添加到其他串行队列，打印结果为：15234
 */
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"执行任务1");
    dispatch_queue_t queue1 = dispatch_queue_create("queue1", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue2 = dispatch_queue_create("queue2", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue1, ^{
        NSLog(@"执行任务2");
        dispatch_sync(queue2, ^{
            NSLog(@"执行任务3");
        });
        NSLog(@"执行任务4");
    });
    NSLog(@"执行任务5");
}
/*
 2020-01-19 03:25:52.761192+0800 多线程[25474:5609516] 执行任务1
 2020-01-19 03:25:52.761393+0800 多线程[25474:5609516] 执行任务5
 2020-01-19 03:25:52.761429+0800 多线程[25474:5609578] 执行任务2
 2020-01-19 03:25:52.761584+0800 多线程[25474:5609578] 执行任务3
 2020-01-19 03:25:52.761749+0800 多线程[25474:5609578] 执行任务4
 */
/*
 3.改为并发队列，打印结果为：15234
 */
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"执行任务1");
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(queue, ^{
        NSLog(@"执行任务2");
        dispatch_sync(queue, ^{
            NSLog(@"执行任务3");
        });
        NSLog(@"执行任务4");
    });
    NSLog(@"执行任务5");
}
/*
 2020-01-19 03:25:52.761192+0800 多线程[25474:5609516] 执行任务1
 2020-01-19 03:25:52.761393+0800 多线程[25474:5609516] 执行任务5
 2020-01-19 03:25:52.761429+0800 多线程[25474:5609578] 执行任务2
 2020-01-19 03:25:52.761584+0800 多线程[25474:5609578] 执行任务3
 2020-01-19 03:25:52.761749+0800 多线程[25474:5609578] 执行任务4
 */
```

# 2 GCD进阶
## 2.1 GCD 队列的服务质量与优先级
### 2.1.1 Quality of Service(QoS) 介绍
[来自文章：关于GCD开发的一些事儿](https://www.jianshu.com/p/f9e01c69a46f)

这是在 iOS8 之后提供的新功能，苹果提供了几个 Quality of Service 枚举来使用：user interactive, user initiated, utility 和 background，通过这告诉系统我们在进行什么样的工作，然后系统会通过合理的资源控制来最高效的执行任务代码，其中主要涉及到 CPU 调度的优先级、IO 优先级、任务运行在哪个线程以及运行的顺序等等，我们通过一个抽象的 Quality of Service 参数来表明任务的意图以及类别。

* NSQualityOfServiceUserInteractive<br>
与用户交互的任务，这些任务通常跟 UI 级别的刷新相关，比如动画，这些任务需要在一瞬间完成；
* NSQualityOfServiceUserInitiated<br>
由用户发起的并且需要立即得到结果的任务，比如滑动 scroll view 时去加载数据用于后续 cell 的显示，这些任务通常跟后续的用户交互相关，在几秒或者更短的时间内完成；
* NSQualityOfServiceUtility<br>
一些可能需要花点时间的任务，这些任务不需要马上返回结果，比如下载的任务，这些任务可能花费几秒或者几分钟的时间；
* NSQualityOfServiceBackground<br>
这些任务对用户不可见，比如后台进行备份的操作，这些任务可能需要较长的时间，几分钟甚至几个小时；
* NSQualityOfServiceDefault<br>
优先级介于 user-initiated 和 utility，当没有 QoS 信息时默认使用，开发者不应该使用这个值来设置自己的任务。


#### <font color=red>●</font> dispatch_qos_class_t
服务质量类，用于确定队列执行的任务的优先级。
```objc
typedef qos_class_t dispatch_qos_class_t;
```
服务质量枚举类型：
 *  QOS_CLASS_USER_INTERACTIVE
 *  QOS_CLASS_USER_INITIATED
 *  QOS_CLASS_DEFAULT
 *  QOS_CLASS_UTILITY
 *  QOS_CLASS_BACKGROUND

队列的优先级与服务质量的对应关系：
队列的优先级|服务质量（QoS）
:--|:--
Main Thread|QOS_CLASS_USER_INTERACTIVE
DISPATCH_QUEUE_PRIORITY_HIGH|QOS_CLASS_USER_INITIATED
DISPATCH_QUEUE_PRIORITY_DEFAULT|QOS_CLASS_DEFAULT
DISPATCH_QUEUE_PRIORITY_LOW|QOS_CLASS_UTILITY
DISPATCH_QUEUE_PRIORITY_BACKGROUND|QOS_CLASS_BACKGROUND

#### <font color=red>●</font> dispatch_queue_get_qos_class
返回指定队列的服务质量。
```objc
dispatch_qos_class_t dispatch_queue_get_qos_class(dispatch_queue_t queue, int *relative_priority_ptr);
```


### 2.1.2 给队列设置 QoS
`dispatch_queue_create`创建队列的 QoS/优先级 跟全局并发队列的默认 QoS 一样，假如我们需要设置队列的 QoS，可以通过以下两个方法：
* dispatch_queue_attr_make_with_qos_class
* dispatch_set_target_queue

#### <font color=red>●</font> dispatch_queue_attr_make_with_qos_class
```objc
/*!
 * @param attr
 * 队列类型，传 DISPATCH_QUEUE_SERIAL 或 DISPATCH_QUEUE_CONCURRENT
 *
 * @param qos_class
 * 服务质量
 *
 * @param relative_priority
 * 服务质量的最大支持负偏移量
 * 必须 <0 且 >QOS_MIN_RELATIVE_PRIORITY(-15)，否则该函数返回NULL。
 * 
 * @return dispatch_queue_attr_t
 * 用于创建具有服务质量信息的队列的属性。
 */
dispatch_queue_attr_t
dispatch_queue_attr_make_with_qos_class(dispatch_queue_attr_t _Nullable attr,
		dispatch_qos_class_t qos_class, int relative_priority);
```
```objc
- (void)test
{
    dispatch_queue_attr_t userInitiatedQueue_attr = dispatch_queue_attr_make_with_qos_class (DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, -1);
    dispatch_queue_attr_t backgroundQueue_attr = dispatch_queue_attr_make_with_qos_class (DISPATCH_QUEUE_SERIAL, QOS_CLASS_BACKGROUND, -1);
    dispatch_queue_t userInitiatedQueue = dispatch_queue_create("myqueue1", userInitiatedQueue_attr);
    dispatch_queue_t backgroundQueue = dispatch_queue_create("myqueue2", backgroundQueue_attr);
    for (int i = 0; i < 3; i++) {
        dispatch_async(backgroundQueue, ^{
            NSLog(@"backgroundQueue,%d",i);
        });
    }
    for (int i = 0; i < 3; i++) {
        dispatch_async(userInitiatedQueue, ^{
            NSLog(@"userInitiatedQueue,%d",i);
        });
    }
}
/*
2020-02-01 02:24:15.957997+0800 多线程[6004:953386] userInitiatedQueue,0
2020-02-01 02:24:15.958233+0800 多线程[6004:953386] userInitiatedQueue,1
2020-02-01 02:24:15.958455+0800 多线程[6004:953386] userInitiatedQueue,2
2020-02-01 02:24:15.960363+0800 多线程[6004:953388] backgroundQueue,0
2020-02-01 02:24:15.961167+0800 多线程[6004:953388] backgroundQueue,1
2020-02-01 02:24:15.962437+0800 多线程[6004:953388] backgroundQueue,2
 */
```
#### <font color=red>●</font> dispatch_set_target_queue
设置队列的 QoS 或者优先级和另一个队列一样。
```objc
/*!
 * @param object  要设置 QoS 或者优先级的队列
 * @param queue   参照队列
 */
void dispatch_set_target_queue(dispatch_object_t object,
		dispatch_queue_t _Nullable queue);
```
```objc
- (void)test
{
    dispatch_queue_t userInitiatedGlobalQueue = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED,0);
    dispatch_queue_t backgroundGlobalQueue = dispatch_get_global_queue(QOS_CLASS_BACKGROUND,0);
    dispatch_queue_t userInitiatedSerialQueue = dispatch_queue_create("myqueue1",NULL);
    dispatch_queue_t backgroundSerialQueue = dispatch_queue_create("myqueue2",NULL);
    //设置 serialQueue 的优先级跟 globalQueue 的优先级一样
    dispatch_set_target_queue(userInitiatedSerialQueue, userInitiatedGlobalQueue);
    dispatch_set_target_queue(backgroundSerialQueue, backgroundGlobalQueue);
    for (int i = 0; i < 3; i++) {
        dispatch_async(backgroundSerialQueue, ^{
            NSLog(@"backgroundSerialQueue,%d",i);
        });
    }
    for (int i = 0; i < 3; i++) {
        dispatch_async(userInitiatedSerialQueue, ^{
            NSLog(@"userInitiatedSerialQueue,%d",i);
        });
    }
}
/*
2020-02-01 02:36:06.912674+0800 多线程[6071:963254] userInitiatedSerialQueue,0
2020-02-01 02:36:06.912841+0800 多线程[6071:963254] userInitiatedSerialQueue,1
2020-02-01 02:36:06.912967+0800 多线程[6071:963254] userInitiatedSerialQueue,2
2020-02-01 02:36:06.917597+0800 多线程[6071:963251] backgroundSerialQueue,0
2020-02-01 02:36:06.917730+0800 多线程[6071:963251] backgroundSerialQueue,1
2020-02-01 02:36:06.917822+0800 多线程[6071:963251] backgroundSerialQueue,2
 */
```

## 2.2 GCD 队列任务间依赖关系
#### <font color=red>●</font> dispatch_set_target_queue
除了能用来设置队列的 QoS 之外，还能够创建队列的层次体系。当我们想让不同队列中的任务同步的执行时，我们可以创建一个串行队列，然后将这些队列的 target 指向新创建的队列即可，比如：
![队列体系.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a5a80cdadbc46329c9a252d5f1a770d~tplv-k3u1fbpfcp-zoom-1.image)
```objc
    dispatch_queue_t targetQueue = dispatch_queue_create("target_queue", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue1 = dispatch_queue_create("queue1", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue2 = dispatch_queue_create("queue2", DISPATCH_QUEUE_CONCURRENT);
    dispatch_set_target_queue(queue1, targetQueue);
    dispatch_set_target_queue(queue2, targetQueue);
    dispatch_async(queue1, ^{
        NSLog(@"执行任务1,%s",dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
        sleep(1);
    });
    dispatch_async(queue2, ^{
        NSLog(@"执行任务2,%s",dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
        sleep(1);
    });
    dispatch_async(queue2, ^{
        NSLog(@"执行任务3,%s",dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
        sleep(1);
    });
/*
2020-02-01 19:05:18.142729+0800 多线程[6669:1213619] 执行任务1,queue1
2020-02-01 19:05:19.143926+0800 多线程[6669:1213619] 执行任务2,queue2
2020-02-01 19:05:20.147740+0800 多线程[6669:1213619] 执行任务3,queue2
 */
```
**注意点：** 避免相互依赖，如将队列 A 的目标队列设置为队列 B，并将队列 B 的目标队列设置为队列 A。


## 2.3 Dispatch Block
前面说过，GCD 中的任务有两种封装：dispatch_block_t 和 dispatch_function_t，且 dispatch_block_t 比较常用。
#### <font color=red>●</font> dispatch_block_t
提交给指定队列的 block，无参无返回值。
```objc
typedef void (^dispatch_block_t)(void);
```
#### <font color=red>●</font> dispatch_block_create
创建一个 dispatch_block_t 对象。
```objc
dispatch_block_t dispatch_block_create(dispatch_block_flags_t flags, dispatch_block_t block);
```
```objc
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    //创建一个 block
    dispatch_block_t block = ^{
        NSLog(@"%@",[NSThread currentThread]);
    };
    dispatch_async(queue, block);
```
#### <font color=red>●</font> dispatch_block_create_with_qos_class
创建一个带有 QoS 的 block，指定 block 的优先级。
```objc
dispatch_block_t dispatch_block_create_with_qos_class(dispatch_block_flags_t flags, dispatch_qos_class_t qos_class, int relative_priority, dispatch_block_t block);
```
```objc
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_block_t BackGroundBlock = dispatch_block_create_with_qos_class(0, QOS_CLASS_BACKGROUND, -1, ^{
            NSLog(@"BackGroundBlock");
    });
    dispatch_block_t UserInitiatedBlock = dispatch_block_create_with_qos_class(0, QOS_CLASS_USER_INITIATED, -1, ^{
            NSLog(@"UserInitiatedBlock");
    });
    for (int i = 0; i < 3; i++) {
        dispatch_async(queue, BackGroundBlock);
        dispatch_async(queue, UserInitiatedBlock);
    }
/*
2020-02-01 19:52:39.673428+0800 多线程[6817:1247472] UserInitiatedBlock
2020-02-01 19:52:39.673494+0800 多线程[6817:1247700] UserInitiatedBlock
2020-02-01 19:52:39.673517+0800 多线程[6817:1247701] UserInitiatedBlock
2020-02-01 19:52:39.674075+0800 多线程[6817:1247699] BackGroundBlock
2020-02-01 19:52:39.676642+0800 多线程[6817:1247472] BackGroundBlock
2020-02-01 19:52:39.676751+0800 多线程[6817:1247699] BackGroundBlock
 */
```
#### <font color=red>●</font> dispatch_block_notify
在被观察块 block 执行完毕之后，立即将通知块 block 提交到指定队列。
```objc
/*!
 * @param block               需要观察的block
 * @param queue               notification_block提交的队列
 * @param notification_block  需要通知的block
 */
void dispatch_block_notify(dispatch_block_t block, dispatch_queue_t queue, dispatch_block_t notification_block);
```
```objc
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);
    dispatch_block_t observation_block = dispatch_block_create(0, ^{
        NSLog(@"observation_block begin");
        [NSThread sleepForTimeInterval:1];
        NSLog(@"observation_block done");
    });
    dispatch_async(queue, observation_block);
    dispatch_block_t notification_block = dispatch_block_create(0, ^{
        NSLog(@"notification_block");
    });
    //当observation_block执行完毕后，提交notification_block到global queue中执行
    dispatch_block_notify(observation_block, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), notification_block);
/*
2020-02-01 20:04:04.340404+0800 多线程[6851:1257066] observation_block begin
2020-02-01 20:04:05.342687+0800 多线程[6851:1257066] observation_block done
2020-02-01 20:04:05.342877+0800 多线程[6851:1257060] notification_block
 */
```

#### <font color=red>●</font> dispatch_block_wait
同步等待，直到指定的 block 执行完成或指定的超时时间结束为止才返回；<br>
设置等待时间 DISPATCH_TIME_NOW 会立刻返回，
设置 DISPATCH_TIME_FOREVER 会无限期等待指定的 block 执行完成才返回。
```objc
/*!
 * @param block
 *
 * @param timeout
 * 超时时长
 * 
 * @return long
 * 如果 block 在指定的超时时间内完成，则返回0；
 * 超时则返回非0。
 */
long dispatch_block_wait(dispatch_block_t block, dispatch_time_t timeout);
```
```objc
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);
    dispatch_block_t block = dispatch_block_create(0, ^{
        NSLog(@"begin");
        [NSThread sleepForTimeInterval:1];
        NSLog(@"done");
    });
    dispatch_async(queue, block);
    //等待前面的任务执行完毕
    dispatch_block_wait(block, DISPATCH_TIME_FOREVER);
    NSLog(@"coutinue");
/*
2020-02-01 20:16:18.361881+0800 多线程[6894:1266019] begin
2020-02-01 20:16:19.363144+0800 多线程[6894:1266019] done
2020-02-01 20:16:19.363419+0800 多线程[6894:1265943] coutinue
 */
```

#### <font color=red>●</font> dispatch_block_cancel
异步取消指定的 block，正在执行的 block 不会被取消。
```objc
void dispatch_block_cancel(dispatch_block_t block);
```
#### <font color=red>●</font> dispatch_block_testcancel
测试指定的 block 是否被取消。返回非0代表已被取消；返回0代表没有取消。
```objc
long dispatch_block_testcancel(dispatch_block_t block);
```
```objc
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);
    dispatch_block_t block1 = dispatch_block_create(0, ^{
        NSLog(@"block1 begin");
        [NSThread sleepForTimeInterval:1];
        NSLog(@"block1 done");
    });
    dispatch_block_t block2 = dispatch_block_create(0, ^{
        NSLog(@"block2");
    });
    dispatch_async(queue, block1);
    dispatch_async(queue, block2);
    //取消block2
    dispatch_block_cancel(block2);
    //测试block2是否被取消
    NSLog(@"block2是否被取消:%ld",dispatch_block_testcancel(block2));
/*
2020-02-01 20:29:42.118735+0800 多线程[7018:1278505] block1 begin
2020-02-01 20:29:42.118750+0800 多线程[7018:1278469] block2是否被取消:1
2020-02-01 20:29:43.122961+0800 多线程[7018:1278505] block1 done
 */
```

## 2.4 Dispatch Group 队列组
### 2.4.1 队列组的使用
GCD 队列组，又称“调度组”，实现所有任务执行完成后有一个统一的回调。<br>
有时候我们需要在多个异步任务都执行完毕以后再继续执行其他任务，这时候就可以使用队列组。

#### <font color=red>●</font> dispatch_group_create
创建一个队列组。
```objc
dispatch_group_t dispatch_group_create(void);
```
#### <font color=red>●</font> dispatch_group_async
异步执行一个 block，并与指定的队列组关联。
```objc
void dispatch_group_async(dispatch_group_t group, dispatch_queue_t queue, dispatch_block_t block);
```
#### <font color=red>●</font> dispatch_group_notify
等待先前 dispatch_group_async 添加的 block 都执行完毕以后，将 dispatch_group_notify 中的 block 提交到指定队列。
```objc
void dispatch_group_notify(dispatch_group_t group, dispatch_queue_t queue, dispatch_block_t block);
```
#### <font color=red>●</font> dispatch_group_wait
同步等待先前 dispatch_group_async 添加的 block 都执行完毕或指定的超时时间结束为止才返回。
```objc
// @return long 如果 block 在指定的超时时间内完成，则返回0；超时则返回非0。
long dispatch_group_wait(dispatch_group_t group, dispatch_time_t timeout);
```


```objc
    // 创建队列组
    dispatch_group_t group = dispatch_group_create();
    // 获取全局并发队列
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    // 添加异步任务：把任务添加到队列，等所有任务都执行完毕，通知队列组
    dispatch_group_async(group, queue, ^{
        for (int i = 0; i < 3; i++) {
            NSLog(@"%@,执行任务1",[NSThread currentThread]);
        }
    });
    dispatch_group_async(group, queue, ^{
        for (int i = 0; i < 3; i++) {
            NSLog(@"%@,执行任务2",[NSThread currentThread]);
        }
    });
    // 所有（dispatch_group_async）任务都执行完毕，获得通知（异步执行），将（dispatch_group_notify）中的 block 任务添加到指定队列 
    // 这行代码是会立刻执行的
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        // 但是里面的任务需要等到队列组的都执行完毕，等待通知
        for (int i = 0; i < 3; i++) {
            NSLog(@"%@,执行任务3",[NSThread currentThread]);
        }
    });
/*
<NSThread: 0x600000cbd200>{number = 4, name = (null)},执行任务1
<NSThread: 0x600000c68440>{number = 7, name = (null)},执行任务2
<NSThread: 0x600000c68440>{number = 7, name = (null)},执行任务2
<NSThread: 0x600000cbd200>{number = 4, name = (null)},执行任务1
<NSThread: 0x600000c68440>{number = 7, name = (null)},执行任务2
<NSThread: 0x600000cbd200>{number = 4, name = (null)},执行任务1
<NSThread: 0x600000cedbc0>{number = 1, name = main},执行任务3
<NSThread: 0x600000cedbc0>{number = 1, name = main},执行任务3
<NSThread: 0x600000cedbc0>{number = 1, name = main},执行任务3
 */
```
例如：异步下载歌曲，等所有歌曲都下载完毕以后，转到主线程提示用户。
```objc
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    dispatch_group_async(group, queue, ^{
        NSLog(@"%@,下载歌曲1",[NSThread currentThread]);
    });
    dispatch_group_async(group, queue, ^{
        NSLog(@"%@,下载歌曲2",[NSThread currentThread]);
    });
    dispatch_group_async(group, queue, ^{
        NSLog(@"%@,下载歌曲3",[NSThread currentThread]);
    });
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        [NSThread sleepForTimeInterval:1];
        NSLog(@"%@,下载完毕",[NSThread currentThread]);
    });
/*
<NSThread: 0x6000028c8c00>{number = 6, name = (null)},下载歌曲2
<NSThread: 0x60000283f400>{number = 5, name = (null)},下载歌曲1
<NSThread: 0x600002803440>{number = 4, name = (null)},下载歌曲3
<NSThread: 0x60000285c5c0>{number = 1, name = main},下载完毕
 */
```

### 2.4.2 队列组的原理
真正实现统一回调的操作：
```objc
void dispatch_group_enter(dispatch_group_t group);
void dispatch_group_leave(dispatch_group_t group);
```
```objc
    dispatch_group_async(group, queue, ^{ 
    }); 
    //等价于
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        dispatch_group_leave(group);
    });
```
```objc
    // 1.创建队列组
    dispatch_group_t group = dispatch_group_create();
    // 2.获取全局并发队列
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    //ARC中不用写也不能写
//    dispatch_retain(group);
    // 3.进入队列组，执行此函数后，再添加的异步执行的block任务都会被group监听
    dispatch_group_enter(group);
    // 4.添加任务
    dispatch_async(queue, ^{
        NSLog(@"%@,执行任务1",[NSThread currentThread]);
        // 5.离开队列组
        dispatch_group_leave(group);
        //ARC中不用写也不能写
//        dispatch_release(group);
    });
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"%@,执行任务2",[NSThread currentThread]);
        dispatch_group_leave(group);
    });
    // 6.获得队列组的通知
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"%@,执行任务3",[NSThread currentThread]);
    });
    // 7.等待队列组，监听的队列中的所有任务都执行完毕，才会执行后续代码，会阻塞线程（很少使用）
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    NSLog(@"%@,执行任务4",[NSThread currentThread]);
/*
<NSThread: 0x600003d98d00>{number = 3, name = (null)},执行任务1
<NSThread: 0x600003d4de80>{number = 9, name = (null)},执行任务2
<NSThread: 0x600003dcd2c0>{number = 1, name = main},执行任务4
<NSThread: 0x600003dcd2c0>{number = 1, name = main},执行任务3
 */
```

## 2.5 Dispatch Once 一次性执行
### 2.5.1 一次性执行的使用

#### <font color=red>●</font> dispatch_once
在应用程序的生命周期内只执行一次提交的 block。
```objc
/*!
 * @param predicate 用来确定这个block是不是已经被执行过了
 * @param block
 */
void dispatch_once(dispatch_once_t *predicate, dispatch_block_t block);
```


使用以下`dispatch_once`代码块实现一次性执行（在当前线程执行）
```objc
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        <#code to be executed once#>
    });
```
```objc
- (void)test
{
    for (int i = 0; i < 100; i++) {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            NSLog(@"%@",[NSThread currentThread]);
        });
    }
}
// <NSThread: 0x600002c4e1c0>{number = 1, name = main}
```

#### <font color=red>●</font> dispatch_once_f
道理同 dispatch_sync_f，不再赘述。

### 2.5.2 一次性执行的原理
判断一个全局静态变量的值，默认是 0，执行完`dispatch_once`后设置为 -1。
`dispatch_once`内部会判断这个变量的值，如果是 0 才执行。


### 2.5.3 实现单例线程安全
使用`dispatch_once`可以让单例线程安全，并且比加锁的效率更高。
```objc
// 普通单例，线程不安全
+ (instancetype)shareGCDTest {
    static id instance = nil;
    if (instance == nil) {
        instance = [[self alloc]init];
    }
    return instance;
}
// 加锁，线程安全
+ (instancetype)shareGCDTestLock {
    static id instance = nil;
    @synchronized (self) {
        if (instance == nil) {
            instance = [[self alloc]init];
        }
    }
    return instance;
}
// dispatch_once，线程安全，效率更高
+ (instancetype)shareGCDTestOnce {
    static id instance = nil;
    // dispatch_once
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        if (instance == nil) {
            instance = [[self alloc]init];
        }
    });
    return instance;
}
```

## 2.6 Dispatch After 延迟执行
#### <font color=red>●</font> dispatch_time
创建一个 dispatch_time_t 对象，通常与 dispatch_after 函数配合使用。
```objc
dispatch_time_t dispatch_time(dispatch_time_t when, int64_t delta);
```


#### <font color=red>●</font> dispatch_after
延迟对应时间后，异步添加 block 到指定的 queue。

**注意点：** `dispatch_after`函数并不是延迟对应时间后立即执行`block`中的任务，而是在指定时间后将任务加到指定队列中，考虑到队列阻塞等情况，这个任务延迟执行的时间是不准确的。
```objc
/*!
 * @param when   延迟多长时间（精确到纳秒）
 * @param queue  队列
 * @param block  任务
 */
void dispatch_after(dispatch_time_t when, dispatch_queue_t queue, dispatch_block_t block);
```

```objc
- (void)test
{
    //指定2s后，将任务加到队列中
    NSLog(@"%@",[NSThread currentThread]);
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        NSLog(@"%@",[NSThread currentThread]);
    });
}
/*
2020-01-20 00:44:08.035740+0800 多线程[27530:6054500] <NSThread: 0x600002d553c0>{number = 1, name = main}
2020-01-20 00:44:10.224500+0800 多线程[27530:6054500] <NSThread: 0x600002d553c0>{number = 1, name = main}
 */
```
#### <font color=red>●</font> dispatch_after_f
道理同 dispatch_sync_f，不再赘述。

## 2.7 Dispatch Barrier 栅栏函数

### 2.7.1 Dispatch Barrier 简介

**Dispatch Barrier：** 在并发调度队列中执行的任务的同步点。<br> 
使用栅栏来同步调度队列中一个或多个任务的执行。在向并发调度队列添加栅栏时，该队列会延迟栅栏任务（以及栅栏之后提交的所有任务）的执行，直到所有先前提交的任务都执行完成为止。在完成先前的任务后，队列将自己执行栅栏任务。栅栏任务执行完毕后，队列将恢复其正常执行行为。

**Dispatch Barrier 栅栏函数：**
* 同步栅栏函数<br> 
dispatch_barrier_sync：提交一个栅栏 block 以同步执行，并等待该 block 执行完；<br> 
dispatch_barrier_sync_f：提交一个栅栏 function 以同步执行，并等待该 function 执行完。
* 异步栅栏函数<br> 
dispatch_barrier_async：提交一个栅栏 block 以异步执行，并直接返回；<br> 
dispatch_barrier_async_f：提交一个栅栏 function 以异步执行，并直接返回。

### 2.7.2 同步栅栏函数 dispatch_barrier_sync
```objc
void dispatch_barrier_sync(dispatch_queue_t queue, dispatch_block_t block);
```

`dispatch_barrier_sync`同步栅栏函数会等待该函数传入的队列中的任务都执行完毕，再执行`dispatch_barrier_sync`函数中的任务以及后面的任务，会阻塞当前线程。

#### 使用场景：保证顺序执行，会阻塞当前线程


**需求：** 现有任务1、2、3、4，前两个任务执行完毕，再执行后两个任务以及主线程的代码。

**解决方法：**
* 1.使用 GCD 队列组；
* 2.使用`dispatch_barrier_sync`函数。
```objc
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    /* 1.异步函数 */
    dispatch_async(queue, ^{
        for (NSUInteger i = 0; i < 3; i++) {
            NSLog(@"执行任务1-%zd-%@",i,[NSThread currentThread]);
        }
    });
    dispatch_async(queue, ^{
        for (NSUInteger i = 0; i < 3; i++) {
            NSLog(@"执行任务2-%zd-%@",i,[NSThread currentThread]);
        }
    });
    /* 2. 同步栅栏函数 */
    dispatch_barrier_sync(queue, ^{
        NSLog(@"------------------dispatch_barrier_sync-%@",[NSThread currentThread]);
    });
    NSLog(@"任务1、2执行完毕");
    /* 3. 异步函数 */
    dispatch_async(queue, ^{
        for (NSUInteger i = 0; i < 3; i++) {
            NSLog(@"执行任务3-%zd-%@",i,[NSThread currentThread]);
        }
    });
    NSLog(@"正在执行任务3、4");
    dispatch_async(queue, ^{
        for (NSUInteger i = 0; i < 3; i++) {
            NSLog(@"执行任务4-%zd-%@",i,[NSThread currentThread]);
        }
    });
/*
执行任务1-0-<NSThread: 0x600003ffa780>{number = 5, name = (null)}
执行任务2-0-<NSThread: 0x600003fefa00>{number = 3, name = (null)}
执行任务2-1-<NSThread: 0x600003fefa00>{number = 3, name = (null)}
执行任务1-1-<NSThread: 0x600003ffa780>{number = 5, name = (null)}
执行任务2-2-<NSThread: 0x600003fefa00>{number = 3, name = (null)}
执行任务1-2-<NSThread: 0x600003ffa780>{number = 5, name = (null)}
------------------dispatch_barrier_sync-<NSThread: 0x600003fa5b80>{number = 1, name = main}
任务1、2执行完毕
正在执行任务3、4
执行任务3-0-<NSThread: 0x600003ffa780>{number = 5, name = (null)}
执行任务4-0-<NSThread: 0x600003fefa00>{number = 3, name = (null)}
执行任务3-1-<NSThread: 0x600003ffa780>{number = 5, name = (null)}
执行任务4-1-<NSThread: 0x600003fefa00>{number = 3, name = (null)}
执行任务3-2-<NSThread: 0x600003ffa780>{number = 5, name = (null)}
执行任务4-2-<NSThread: 0x600003fefa00>{number = 3, name = (null)}
 */
```

### 2.7.3 异步栅栏函数 dispatch_barrier_async
```objc
void dispatch_barrier_async(dispatch_queue_t queue, dispatch_block_t block);
```
我们先来看一下上面的代码改为异步栅栏函数的执行效果：
```objc
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    /* 1.异步函数 */
    dispatch_async(queue, ^{
        for (NSUInteger i = 0; i < 3; i++) {
            NSLog(@"执行任务1-%zd-%@",i,[NSThread currentThread]);
        }
    });
    dispatch_async(queue, ^{
        for (NSUInteger i = 0; i < 3; i++) {
            NSLog(@"执行任务2-%zd-%@",i,[NSThread currentThread]);
        }
    });
    /* 2. 异步栅栏函数 */
    dispatch_barrier_async(queue, ^{
        NSLog(@"------------------dispatch_barrier_async-%@",[NSThread currentThread]);
    });
    NSLog(@"任务1、2执行完毕");
    /* 3. 异步函数 */
    dispatch_async(queue, ^{
        for (NSUInteger i = 0; i < 3; i++) {
            NSLog(@"执行任务3-%zd-%@",i,[NSThread currentThread]);
        }
    });
    NSLog(@"正在执行任务3、4");
    dispatch_async(queue, ^{
        for (NSUInteger i = 0; i < 3; i++) {
            NSLog(@"执行任务4-%zd-%@",i,[NSThread currentThread]);
        }
    });
/*
任务1、2执行完毕
执行任务2-0-<NSThread: 0x6000020f3100>{number = 4, name = (null)}
正在执行任务3、4
执行任务1-0-<NSThread: 0x6000020c6a00>{number = 6, name = (null)}
执行任务2-1-<NSThread: 0x6000020f3100>{number = 4, name = (null)}
执行任务1-1-<NSThread: 0x6000020c6a00>{number = 6, name = (null)}
执行任务2-2-<NSThread: 0x6000020f3100>{number = 4, name = (null)}
执行任务1-2-<NSThread: 0x6000020c6a00>{number = 6, name = (null)}
------------------dispatch_barrier_async-<NSThread: 0x6000020c6a00>{number = 6, name = (null)}
执行任务4-0-<NSThread: 0x6000020d2e00>{number = 7, name = (null)}
执行任务3-0-<NSThread: 0x6000020c6a00>{number = 6, name = (null)}
执行任务4-1-<NSThread: 0x6000020d2e00>{number = 7, name = (null)}
执行任务3-1-<NSThread: 0x6000020c6a00>{number = 6, name = (null)}
执行任务4-2-<NSThread: 0x6000020d2e00>{number = 7, name = (null)}
执行任务3-2-<NSThread: 0x6000020c6a00>{number = 6, name = (null)}
 */
```
从打印日志可以看到，改为异步栅栏函数，任务3、4仍然可以等到任务1、2以及栅栏任务都执行完毕再执行，但不会阻塞主线程，并且栅栏任务是在子线程执行。

`dispatch_barrier_async`异步栅栏函数会等待该函数传入的队列中的任务都执行完毕，再执行`dispatch_barrier_async`函数中的任务以及后面的任务，执行该函数会直接返回，继续往下执行代码，不会阻塞当前线程。

* 主要用于在多个异步操作完成之后，统一对非线程安全的对象进行更新；
* 适合于大规模的 I/O 操作；
* 当访问数据库或者文件的时候，更新数据的时候不能和其他更新或者读取的操作在同一时间执行，可以使用队列组不过有点复杂，可以使用`dispatch_barrier_async`解决；
* 保证线程安全、读写安全。


#### 使用场景1：保证线程安全
**例如：** 我们要从网络上异步获取很多图片，然后将它们添加到非线程安全的对象——数组中去：异步并发。<br>
&emsp;&emsp;同一时间点，可能有多个线程执行给数组添加对象的方法，所以可能会丢掉 1 到多次，我们执行 1000 次，可能数组就保存了 990 多个，还有程序出现奔溃的可能。

**解决办法：**
* ① 加锁：比较耗时，而且下载完什么时候添加进数组也不一定。我们希望所有图片都下载完，再往数组里面添加；
* ② 使用 GCD 队列组；
* ③ 使用`dispatch_barrier_async`函数，栅栏中的任务会等待队列中的所有任务执行完成，才会执行栅栏中的任务，保证了线程安全。
```objc
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 5; i++) {
        dispatch_async(queue, ^{
            //下载图片
            NSLog(@"图片下载完成%d,%@",i,[NSThread currentThread]);
            dispatch_barrier_async(queue, ^{
                //将图片添加进数组
                NSLog(@"添加图片%d,%@",i,[NSThread currentThread]);
            });
        });
    }
/*
图片下载完成2,<NSThread: 0x600000348d80>{number = 5, name = (null)}
图片下载完成0,<NSThread: 0x600000341f40>{number = 4, name = (null)}
图片下载完成1,<NSThread: 0x60000036f480>{number = 6, name = (null)}
图片下载完成3,<NSThread: 0x60000039ce80>{number = 7, name = (null)}
图片下载完成4,<NSThread: 0x600000348d80>{number = 5, name = (null)}
添加图片2,<NSThread: 0x600000348d80>{number = 5, name = (null)}
添加图片0,<NSThread: 0x600000348d80>{number = 5, name = (null)}
添加图片1,<NSThread: 0x600000348d80>{number = 5, name = (null)}
添加图片3,<NSThread: 0x600000348d80>{number = 5, name = (null)}
添加图片4,<NSThread: 0x600000348d80>{number = 5, name = (null)}
*/
```

#### 使用场景2：实现读写安全
`dispatch_barrier_async`可以用来实现“读写安全”。我们将“写”操作放在`dispatch_barrier_async`中，这样能确保在“写”操作的时候会等待前面的“读”操作完成，而后续的“读”操作也会等到“写”操作完成后才能继续执行，提高文件读写的执行效率。
```objc
    // 创建一个并发队列
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    // 读：执行“读”操作
    dispatch_async(queue, ^{
        
    });
    // 写：执行“写”操作
    dispatch_barrier_async(queue, ^{
        
    });
```

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    self.queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 3; i++) {
        [test read];
        [test read];
        [test write];
        [test read];
    }
}

- (void)read {
    dispatch_async(self.queue, ^{
        sleep(1);
        NSLog(@"read,%@",[NSThread currentThread]);
    });
}

- (void)write {
    dispatch_barrier_async(self.queue, ^{
        sleep(1);
        NSLog(@"write,%@",[NSThread currentThread]);
    });
}

/*
2020-01-20 01:45:09.847878+0800 多线程[27767:6103230] read,<NSThread: 0x600002d42ac0>{number = 7, name = (null)}
2020-01-20 01:45:09.847849+0800 多线程[27767:6096149] read,<NSThread: 0x600002d8ed80>{number = 4, name = (null)}
2020-01-20 01:45:10.849965+0800 多线程[27767:6096149] write,<NSThread: 0x600002d8ed80>{number = 4, name = (null)}
2020-01-20 01:45:11.851259+0800 多线程[27767:6103230] read,<NSThread: 0x600002d42ac0>{number = 7, name = (null)}
2020-01-20 01:45:11.851265+0800 多线程[27767:6103231] read,<NSThread: 0x600002d42640>{number = 8, name = (null)}
2020-01-20 01:45:11.851277+0800 多线程[27767:6096149] read,<NSThread: 0x600002d8ed80>{number = 4, name = (null)}
2020-01-20 01:45:12.854305+0800 多线程[27767:6103231] write,<NSThread: 0x600002d42640>{number = 8, name = (null)}
2020-01-20 01:45:13.859167+0800 多线程[27767:6103231] read,<NSThread: 0x600002d42640>{number = 8, name = (null)}
2020-01-20 01:45:13.859167+0800 多线程[27767:6103230] read,<NSThread: 0x600002d42ac0>{number = 7, name = (null)}
2020-01-20 01:45:13.859167+0800 多线程[27767:6096149] read,<NSThread: 0x600002d8ed80>{number = 4, name = (null)}
2020-01-20 01:45:14.864153+0800 多线程[27767:6103231] write,<NSThread: 0x600002d42640>{number = 8, name = (null)}
2020-01-20 01:45:15.869272+0800 多线程[27767:6096149] read,<NSThread: 0x600002d8ed80>{number = 4, name = (null)}
 */
```

### 2.7.4 dispatch_barrier_sync 与 dispatch_barrier_async 的区别及注意点
**相同点：** 都会等待队列中在它们（栅栏）之前提交的任务都执行完毕，再执行它们的任务，接着执行它们后面的任务。

**不同点：**
* `dispatch_barrier_sync`：提交一个栅栏 block 以同步执行，并等待该 block 执行完；由于是同步，不会开启新的子线程，会阻塞当前线程。
* `dispatch_barrier_async`：提交一个栅栏 block 以异步执行，并直接返回，会继续往下执行代码，不会阻塞当前线程。

**注意点：**
* `dispatch_barrier_(a)sync`函数传入的的队列必须是自己手动创建的并发队列，如果传入的是全局并发队列或者串行队列，那么这个函数是没有栅栏的效果的，效果等同于`dispatch_(a)sync`函数。
* 只能栅栏`dispatch_barrier_(a)sync`函数中传入的`queue`。


## 2.8 Dispatch Semaphore 信号量
GCD 信号量`dispatch_semaphore`可以用来控制最大并发数量，可以用来实现 iOS 的线程同步方案。
* 信号量的初始值，可以用来控制线程并发访问的最大数量；
* 信号量的初始值为1，代表同时只允许 1 条线程访问资源，保证线程同步。

```objc
    //信号量的初始值
    int value = 1;
    //创建信号量
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(value);
    //如果信号量的值<=0,当前线程就会进入休眠等待（直到信号量的值>0）
    //如果信号量的值>0,就-1，然后继续往下执行代码
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    //让信号量的值+1
    dispatch_semaphore_signal(semaphore);
```
设置信号量初始值为 3，控制最大并发数为 3：
```objc
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(3);
    for (int i = 0; i < 10; i++) {
        [[[NSThread alloc]initWithBlock:^{
            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
            sleep(2);
            NSLog(@"执行任务%d,%@",i,[NSThread currentThread]);
            dispatch_semaphore_signal(semaphore);
        }] start];
    }
/*
2020-01-20 21:55:05.737842+0800 多线程[29977:6622607] 执行任务6,<NSThread: 0x6000007f58c0>{number = 13, name = (null)}
2020-01-20 21:55:05.737884+0800 多线程[29977:6622604] 执行任务3,<NSThread: 0x6000007f5e80>{number = 10, name = (null)}
2020-01-20 21:55:05.737958+0800 多线程[29977:6622601] 执行任务0,<NSThread: 0x6000007f6340>{number = 7, name = (null)}
2020-01-20 21:55:07.742784+0800 多线程[29977:6622609] 执行任务8,<NSThread: 0x6000007f6100>{number = 15, name = (null)}
2020-01-20 21:55:07.742816+0800 多线程[29977:6622602] 执行任务1,<NSThread: 0x6000007f62c0>{number = 8, name = (null)}
2020-01-20 21:55:07.742879+0800 多线程[29977:6622605] 执行任务4,<NSThread: 0x6000007f6300>{number = 11, name = (null)}
2020-01-20 21:55:09.748850+0800 多线程[29977:6622610] 执行任务9,<NSThread: 0x6000007f6200>{number = 16, name = (null)}
2020-01-20 21:55:09.748845+0800 多线程[29977:6622606] 执行任务5,<NSThread: 0x6000007f6180>{number = 12, name = (null)}
2020-01-20 21:55:09.748850+0800 多线程[29977:6622603] 执行任务2,<NSThread: 0x6000007f5e40>{number = 9, name = (null)}
2020-01-20 21:55:11.754020+0800 多线程[29977:6622608] 执行任务7,<NSThread: 0x6000007f6000>{number = 14, name = (null)}
*/
```

## 2.9 Dispatch Apply 多次执行
#### <font color=red>●</font> dispatch_apply
类似一个 for 循环。提交一个 block 到指定队列，并使该 block 执行指定的次数 n，等待 block 执行完 n 次才会返回。
```objc
/*!
 * @param iterations
 * 执行 block 的次数
 *
 * @param queue
 * 
 * @param block
 * 要提交的 block，此参数不能为空（NULL）
 * 该 block 没有返回值，并带有一个 size_t 类型的参数（iteration：当前迭代索引，即当前是第几次调用）
 */

void dispatch_apply(size_t iterations, dispatch_queue_t queue, void (^block)(size_t));
```
**注意点：**
* 该函数会等待 block 都执行完毕才会返回，所以是同步的，会阻塞；
* 如果指定的队列是全局并发队列`dispatch_get_global_queue`，则这些 block 可以并发执行，这里需要注意可重入性；<br> 
（可重入性相关的文章推荐：[可重入与线程安全](https://www.jianshu.com/p/8a88fbe3e1a9)）
* 如果指定的队列是手动创建的并发队列，在有些情况下不会并发执行，所以建议使用全局并发队列`dispatch_get_global_queue`；
* 当使用串行队列时，不会开启子线程，block 在主线程按串行执行；
* 当使用并发队列时，不一定会开启子线程，block 不一定都在子线程执行，也可能都在主线程执行，取决于任务的耗时程度。

**串行队列：**
```objc
- (void)test
{
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"开始执行");
    dispatch_apply(5, queue, ^(size_t i) {
        sleep(1);
        NSLog(@"%zu,%@",i,[NSThread currentThread]);
    });
    NSLog(@"执行完毕");
}
/*
2020-02-01 00:20:00.828203+0800 多线程[5402:867594] 开始执行
2020-02-01 00:20:01.829521+0800 多线程[5402:867594] 0,<NSThread: 0x600002182200>{number = 1, name = main}
2020-02-01 00:20:02.830285+0800 多线程[5402:867594] 1,<NSThread: 0x600002182200>{number = 1, name = main}
2020-02-01 00:20:03.831774+0800 多线程[5402:867594] 2,<NSThread: 0x600002182200>{number = 1, name = main}
2020-02-01 00:20:04.833280+0800 多线程[5402:867594] 3,<NSThread: 0x600002182200>{number = 1, name = main}
2020-02-01 00:20:05.834919+0800 多线程[5402:867594] 4,<NSThread: 0x600002182200>{number = 1, name = main}
2020-02-01 00:20:05.835200+0800 多线程[5402:867594] 执行完毕
 */
```
**并发队列：**
```objc
- (void)test
{
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    NSLog(@"开始执行");
    dispatch_apply(5, queue, ^(size_t i) {
        sleep(1);
        NSLog(@"%zu,%@",i,[NSThread currentThread]);
    });
    NSLog(@"执行完毕");
}
/*
2020-02-01 00:20:21.107718+0800 多线程[5402:867594] 开始执行
2020-02-01 00:20:22.109137+0800 多线程[5402:867736] 1,<NSThread: 0x6000021b0d80>{number = 3, name = (null)}
2020-02-01 00:20:22.109137+0800 多线程[5402:867594] 0,<NSThread: 0x600002182200>{number = 1, name = main}
2020-02-01 00:20:22.109186+0800 多线程[5402:868618] 2,<NSThread: 0x600002111400>{number = 8, name = (null)}
2020-02-01 00:20:22.109190+0800 多线程[5402:868619] 3,<NSThread: 0x600002108ec0>{number = 9, name = (null)}
2020-02-01 00:20:23.109931+0800 多线程[5402:867736] 4,<NSThread: 0x6000021b0d80>{number = 3, name = (null)}
2020-02-01 00:20:23.110251+0800 多线程[5402:867594] 执行完毕
 */
```
**使用场景：**

在某些场景下使用`dispatch_apply`会对性能有很大的提升，比如你的代码需要以每个像素为基准来处理计算 image 图片。同时`dispatch_apply`能够避免一些线程爆炸的情况发生（创建很多线程）[来自文章：关于GCD开发的一些事儿](https://www.jianshu.com/p/f9e01c69a46f)

```objc
    //危险，可能导致线程爆炸以及死锁
    for (int i = 0; i < 999; i++){
        dispatch_async(q, ^{...});
    }
    dispatch_barrier_sync(q, ^{});

    //较优选择， GCD 会管理并发
    dispatch_apply(999, q, ^(size_t i){...});
```

在异步中实现同步，并将任务并发执行：
```objc
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    dispatch_async(queue, ^{
        NSLog(@"开始执行");
        dispatch_apply(5, queue, ^(size_t i) {
            sleep(1);
            NSLog(@"%zu,%@",i,[NSThread currentThread]);
        });
        NSLog(@"执行完毕");
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"回到主线程");
        });
    });
/*
2020-02-01 00:12:00.243244+0800 多线程[5320:860216] 开始执行
2020-02-01 00:12:01.246097+0800 多线程[5320:860216] 0,<NSThread: 0x600000d769c0>{number = 5, name = (null)}
2020-02-01 00:12:01.249172+0800 多线程[5320:860352] 3,<NSThread: 0x600000dec600>{number = 8, name = (null)}
2020-02-01 00:12:01.249177+0800 多线程[5320:860351] 2,<NSThread: 0x600000dc2dc0>{number = 7, name = (null)}
2020-02-01 00:12:01.249172+0800 多线程[5320:860353] 1,<NSThread: 0x600000dc2ac0>{number = 9, name = (null)}
2020-02-01 00:12:02.246783+0800 多线程[5320:860216] 4,<NSThread: 0x600000d769c0>{number = 5, name = (null)}
2020-02-01 00:12:02.247289+0800 多线程[5320:860216] 执行完毕
2020-02-01 00:12:02.247622+0800 多线程[5320:860146] 回到主线程
 */
```
将上面代码的队列改为手动创建的并发队列，任务就不会并发执行：
```objc
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(queue, ^{
        NSLog(@"开始执行");
        dispatch_apply(5, queue, ^(size_t i) {
            sleep(1);
            NSLog(@"%zu,%@",i,[NSThread currentThread]);
        });
        NSLog(@"执行完毕");
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"回到主线程");
        });
    });
/*
2020-02-01 00:15:40.050698+0800 多线程[5364:863969] 开始执行
2020-02-01 00:15:41.056250+0800 多线程[5364:863969] 0,<NSThread: 0x600000824880>{number = 6, name = (null)}
2020-02-01 00:15:42.061385+0800 多线程[5364:863969] 1,<NSThread: 0x600000824880>{number = 6, name = (null)}
2020-02-01 00:15:43.066950+0800 多线程[5364:863969] 2,<NSThread: 0x600000824880>{number = 6, name = (null)}
2020-02-01 00:15:44.069207+0800 多线程[5364:863969] 3,<NSThread: 0x600000824880>{number = 6, name = (null)}
2020-02-01 00:15:45.071144+0800 多线程[5364:863969] 4,<NSThread: 0x600000824880>{number = 6, name = (null)}
2020-02-01 00:15:45.071491+0800 多线程[5364:863969] 执行完毕
2020-02-01 00:15:45.071912+0800 多线程[5364:863900] 回到主线程
 */
```

#### <font color=red>●</font> dispatch_apply_f
道理同 dispatch_sync_f，不再赘述。

## 2.10 Dispatch Source
### 2.10.1 dispatch_source
[来自文章：关于GCD开发的一些事儿](https://www.jianshu.com/p/f9e01c69a46f)<br>
dispatch 框架提供一套接口用于监听系统底层对象(如文件描述符、Mach 端口、信号量等)，当这些对象有事件产生时会自动把事件的处理 block 函数提交到 dispatch 队列中执行，这套接口就是 Dispatch Source API，Dispatch Source 其实就是对 kqueue 功能的封装，可以去查看 dispatch_source 的 c 源码实现(什么是 kqueue？Google，什么是 Mach 端口? Google Again)，Dispatch Source 主要处理以下几种事件：
```
DISPATCH_SOURCE_TYPE_DATA_ADD        变量增加
DISPATCH_SOURCE_TYPE_DATA_OR         变量OR
DISPATCH_SOURCE_TYPE_MACH_SEND       Mach端口发送
DISPATCH_SOURCE_TYPE_MACH_RECV       Mach端口接收
DISPATCH_SOURCE_TYPE_MEMORYPRESSURE  内存压力情况变化
DISPATCH_SOURCE_TYPE_PROC            与进程相关的事件
DISPATCH_SOURCE_TYPE_READ            可读取文件映像
DISPATCH_SOURCE_TYPE_SIGNAL          接收信号
DISPATCH_SOURCE_TYPE_TIMER           定时器事件
DISPATCH_SOURCE_TYPE_VNODE           文件系统变更
DISPATCH_SOURCE_TYPE_WRITE           可写入文件映像
```
当有事件发生时，dispatch source 自动将一个 block 放入一个 dispatch queue 执行。
#### <font color=red>●</font> dispatch_source_create
创建一个 dispatch source，需要指定事件源的类型，handler 的执行队列，dispatch source 创建完之后将处于挂起状态。此时 dispatch source 会接收事件，但是不会进行处理，你需要设置事件处理的 handler，并执行额外的配置；同时为了防止事件堆积到 dispatch queue 中，dispatch source 还会对事件进行合并，如果新事件在上一个事件处理 handler 执行之前到达，dispatch source 会根据事件的类型替换或者合并新旧事件。
#### <font color=red>●</font> dispatch_source_set_event_handler
给指定的 dispatch source 设置事件发生的处理 handler
#### <font color=red>●</font> dispatch_source_set_cancel_handler
给指定的 dispatch source 设置一个取消处理 handler，取消处理 handler 会在 dispatch soruce 释放之前做些清理工作，比如关闭文件描述符:
```objc
    dispatch_source_set_cancel_handler(mySource, ^{ 
        close(fd); //关闭文件秒速符 
    }); 
```
#### <font color=red>●</font> dispatch_source_cancel
异步地关闭 dispatch source，这样后续的事件发生时不去调用对应的事件处理 handler，但已经在执行的 handler 不会被取消。

### 2.10.2 GCD 定时器
在我的文章[《深入浅出 RunLoop（五）：RunLoop 与 NSTimer》](https://juejin.im/post/6844904073972416519)中提到过 NSTimer 和 CADisplayLink 定时器不准时的问题，解决办法就是使用 GCD 定时器。GCD 的定时器是直接跟系统内核挂钩的，而且它不依赖于`RunLoop`，所以它非常的准时。
```objc
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
    
    //创建定时器
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    //设置时间（start:几s后开始执行； interval:时间间隔）
    uint64_t start = 2.0; //2s后开始执行
    uint64_t interval = 1.0; //每隔1s执行
    dispatch_source_set_timer(timer, dispatch_time(DISPATCH_TIME_NOW, start * NSEC_PER_SEC), interval * NSEC_PER_SEC, 0);
    //设置回调
    dispatch_source_set_event_handler(timer, ^{
       NSLog(@"%@",[NSThread currentThread]);
    });
    //启动定时器
    dispatch_resume(timer);
    NSLog(@"%@",[NSThread currentThread]);
    
    self.timer = timer;
/*
2020-02-01 21:34:23.036474+0800 多线程[7309:1327653] <NSThread: 0x600001a5cfc0>{number = 1, name = main}
2020-02-01 21:34:25.036832+0800 多线程[7309:1327705] <NSThread: 0x600001acb600>{number = 7, name = (null)}
2020-02-01 21:34:26.036977+0800 多线程[7309:1327705] <NSThread: 0x600001acb600>{number = 7, name = (null)}
2020-02-01 21:34:27.036609+0800 多线程[7309:1327707] <NSThread: 0x600001a1e5c0>{number = 4, name = (null)}
 */
```



## 2.11 dispatch_queue_set_specific & dispatch_get_specific
[来自文章：关于GCD开发的一些事儿](https://www.jianshu.com/p/f9e01c69a46f)<br>
这两个 API 类似于`objc_setAssociatedObject`跟`objc_getAssociatedObject`，FMDB 里就用到这个来防止死锁，来看看 FMDB 的部分源码：
```objc
    static const void * const kDispatchQueueSpecificKey = &kDispatchQueueSpecificKey;
    //创建一个串行队列来执行数据库的所有操作
    _queue = dispatch_queue_create([[NSString stringWithFormat:@"fmdb.%@", self] UTF8String], NULL);

    //通过key标示队列，设置context为self
    dispatch_queue_set_specific(_queue, kDispatchQueueSpecificKey, (__bridge void *)self, NULL);
```
当要执行数据库操作时，如果在 queue 里面的 block 执行过程中，又调用了 indatabase 方法，需要检查是不是同一个 queue，因为同一个 queue 的话会产生死锁情况
```objc
- (void)inDatabase:(void (^)(FMDatabase *db))block {
    FMDatabaseQueue *currentSyncQueue = (__bridge id)dispatch_get_specific(kDispatchQueueSpecificKey);
    assert(currentSyncQueue != self && "inDatabase: was called reentrantly on the same queue, which would lead to a deadlock");
}
```

# 3. GCD 源码分析
待更新。
# 4. GCD 相关题目
#### Q1：以下打印的 a 的值为多少？
```objc
    __block int a = 0;
    while (a < 10) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            a++;
        });
    }
    NSLog(@"%d",a);
```
答：a >= 10。
```objc
/* 打印3次的结果
2020-01-29 02:35:42.070283+0800 多线程[49119:9919097] 12
2020-01-29 02:35:51.528086+0800 多线程[49119:9919097] 10
2020-01-29 02:35:52.285512+0800 多线程[49119:9919097] 15
 */

/*
 如果在打印 a 值之前，将线程睡眠一段时间，结果更明显。
 */
    __block int a = 0;
    while (a < 10) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            a++;
        });
    }
    sleep(2);
    NSLog(@"%d",a);

/* 打印3次的结果
2020-01-29 02:38:29.102872+0800 多线程[49139:9921222] 24
2020-01-29 02:38:31.663993+0800 多线程[49139:9921222] 19
2020-01-29 02:38:34.024043+0800 多线程[49139:9921222] 21
 */
```

解析：会发生资源抢夺。当执行`while`第一次判断`a`的值时`a=0`，条件成立，开启一条线程异步执行任务`a++`。由于异步不用等待当前语句执行完毕，就可以执行下一条语句，所以就会执行到第 6 行代码，再次执行`while`判断`a`的值，这时候可能任务`a++`还未执行，`a`的值还是 0，所以当`a=0`时可能会执行两次甚至多次任务`a++`。同理，当`a=1`时可能也会执行两次甚至多次任务`a++`。再由于`while`的成立条件为`a<10`，所以至少当`a=10`的时候才会结束`while`循环，而当`a=10`结束循环的时候可能还有`n`条线程还未执行完毕任务`a++`，所以打印的`a`的值`>=10`。

#### Q1扩展：如何输出 a 最终的值？
答：利用队列的 FIFO 规则，如下。需要注意的是，这种方法输出的 a 最终的值不是绝对的。
``` objc
    __block int a = 0;
    while (a < 10) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            a++;
        });
    }
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"%d",a);
    });
```
如果想输出 a 最终的绝对的值，使用 GCD 队列组：
```objc
    __block int a = 0;
    dispatch_group_t group = dispatch_group_create();
    while (a < 10) {
        dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
            a++;
        });
    }
    dispatch_group_notify(group, dispatch_get_global_queue(0, 0), ^{
        NSLog(@"%d",a);
    });
```
#### Q1扩展：如何让 a 最终的值为10？
答：可以使用 GCD 信号量，如下：
```objc
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    __block int a = 0;
    while (a < 10) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            a++;
            NSLog(@"a=%d,%@",a,[NSThread currentThread]);
            dispatch_semaphore_signal(semaphore);
        });
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    }
    NSLog(@"%d",a);
```

## 相关链接
GitHub：[https://github.com/dolphin1208/Thread](https://github.com/dolphin1208/Thread)


## 参考
[Dispatch（苹果官方文档）](https://developer.apple.com/documentation/dispatch?language=objc)<br>
[GCD 源码：https://opensource.apple.com/tarballs/libdispatch/](https://opensource.apple.com/tarballs/libdispatch/)<br>
[GCD 源码：https://apple.github.io/swift-corelibs-libdispatch/](https://apple.github.io/swift-corelibs-libdispatch/)<br>
[GCD官方文档解析（私人博客）](https://www.dazhuanlan.com/2019/10/20/5dac81a4d3c87/)<br>
[关于GCD开发的一些事儿（简书）](https://www.jianshu.com/p/f9e01c69a46f)
