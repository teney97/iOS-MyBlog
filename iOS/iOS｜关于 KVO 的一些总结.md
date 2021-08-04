

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c88b93019d54ae19836dd4cb09cd1ad~tplv-k3u1fbpfcp-zoom-1.image)


## 1. 什么是 KVO
* `KVO`的全称是`Key-Value Observing`，俗称“键值观察/监听”，是苹果提供的一套事件通知机制，允许一个对象观察/监听另一个对象指定属性值的改变。当被观察对象属性值发生改变时，会触发`KVO`的监听方法来通知观察者。`KVO`是在`MVC`应用程序中的各层之间进行通信的一种特别有用的技术。
* `KVO`和`NSNotification`都是`iOS`中观察者模式的一种实现。
* `KVO`可以监听单个属性的变化，也可以监听集合对象的变化。监听集合对象变化时，需要通过`KVC`的`mutableArrayValueForKey:`等可变代理方法获得集合代理对象，并使用代理对象进行操作，当代理对象的内部对象发生改变时，会触发`KVO`的监听方法。集合对象包含`NSArray`和`NSSet`。
* `KVO`和`KVC`有着密切的关系，如果想要深入了解`KVO`，建议先学习`KVC`。
<br>[传送门：iOS - 关于 KVC 的一些总结](https://juejin.im/post/6844904082415550477)


## 2. KVO 的基本使用
`KVO`使用三部曲：添加/注册`KVO`监听、实现监听方法以接收属性改变通知、 移除`KVO`监听。
1. 调用方法`addObserver:forKeyPath:options:context: `给被观察对象添加观察者；
2. 在观察者类中实现`observeValueForKeyPath:ofObject:change:context:`方法以接收属性改变的通知消息；
3. 当观察者不需要再监听时，调用`removeObserver:forKeyPath:`方法将观察者移除。需要注意的是，至少需要在观察者销毁之前，调用此方法，否则可能会导致`Crash`。

### 2.1 注册方法
```objc
/*
 ** target：  被观察对象
 ** observer：观察者对象
 ** keyPath： 被观察对象的属性的关键路径，不能为nil
 ** options： 观察的配置选项，包括观察的内容（枚举类型）：
           NSKeyValueObservingOptionNew：观察新值
           NSKeyValueObservingOptionOld：观察旧值
           NSKeyValueObservingOptionInitial：观察初始值，如果想在注册观察者后，立即接收一次回调，可以加入该枚举值
           NSKeyValueObservingOptionPrior：分别在值改变前后触发方法（即一次修改有两次触发）
 ** context： 可以传入任意数据（任意类型的对象或者C指针），在监听方法中可以接收到这个数据，是KVO中的一种传值方式
             如果传的是一个对象，必须在移除观察之前持有它的强引用，否则在监听方法中访问context就可能导致Crash
 */
- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath
 options:(NSKeyValueObservingOptions)options context:(nullable void *)context;
```

### 2.2 监听方法
如果对象被注册成为观察者，则该对象必须能响应以下监听方法，即该对象所属类中必须实现监听方法。当被观察对象属性发生改变时就会调用监听方法。如果没有实现就会导致`Crash`。
```objc
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
/*
 ** keyPath：被观察对象的属性的关键路径
 ** object： 被观察对象
 ** change： 字典 NSDictionary<NSKeyValueChangeKey, id>，属性值更改的详细信息，根据注册方法中options参数传入的枚举来返回
             key为 NSKeyValueChangeKey 枚举类型
             {
                 1.NSKeyValueChangeKindKey：存储本次改变的信息（change字典中默认包含这个key）
                 {
                     对应枚举类型 NSKeyValueChange
                     typedef NS_ENUM(NSUInteger, NSKeyValueChange) {
                         NSKeyValueChangeSetting     = 1,
                         NSKeyValueChangeInsertion   = 2,
                         NSKeyValueChangeRemoval     = 3,
                         NSKeyValueChangeReplacement = 4,
                     };
                     如果是对被观察对象属性（包括集合）进行赋值操作，kind 字段的值为 NSKeyValueChangeSetting
                     如果被观察的是集合对象，且进行的是（插入、删除、替换）操作，则会根据集合对象的操作方式来设置 kind 字段的值
                         插入：NSKeyValueChangeInsertion
                         删除：NSKeyValueChangeRemoval
                         替换：NSKeyValueChangeReplacement
                 }    
                 2.NSKeyValueChangeNewKey：存储新值（如果options中传入NSKeyValueObservingOptionNew，change字典中就会包含这个key）
                 3.NSKeyValueChangeOldKey：存储旧值（如果options中传入NSKeyValueObservingOptionOld，change字典中就会包含这个key）
                 4.NSKeyValueChangeIndexesKey：如果被观察的是集合对象，且进行的是（插入、删除、替换）操作，则change字典中就会包含这个key，
                     这个key的value是一个NSIndexSet对象，包含更改关系中的索引
                 5.NSKeyValueChangeNotificationIsPriorKey：如果options中传入NSKeyValueObservingOptionPrior，则在改变前通知的change字典中会包含这个key。
                     这个key对应的value是NSNumber包装的YES，我们可以这样来判断是不是在改变前的通知[change[NSKeyValueChangeNotificationIsPriorKey] boolValue] == YES]
             }
 ** context：注册方法中传入的context
 */
}
```

### 2.3 移除方法
在调用注册方法后，`KVO`并不会对观察者进行强引用，所以需要注意观察者的生命周期。至少需要在观察者销毁之前，调用以下方法移除观察者，否则如果在观察者被释放后，再次触发`KVO`监听方法就会导致`Crash`。
```objc
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath;
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath context:(nullable void *)context;
```

### 2.4 使用示例

以下使用`KVO`为`person`对象添加观察者为当前`viewController`，监听`person`对象的`name`属性值的改变。当`name`值改变时，触发`KVO`的监听方法。

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.person = [HTPerson new];
    [self.person addObserver:self forKeyPath:@"name" options:(NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld) context:NULL];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    self.person.name= @"张三";
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"keyPath:%@",keyPath);
    NSLog(@"object:%@",object);
    NSLog(@"change:%@",change);
    NSLog(@"context:%@",context);
}

- (void)dealloc
{
    [self.person removeObserver:self forKeyPath:@"name"];
}
```
>keyPath:name<br>
>object:<HTPerson: 0x600003ae4340><br>
>change:{ kind = 1; new = "\U70b9\U51fb"; old = "<null>"; }<br>
>context:(null)


### 2.5 实际应用
`KVO`主要用来做键值观察操作，想要一个值发生改变后通知另一个对象，则用`KVO`实现最为合适。斯坦福大学的`iOS`教程中有一个很经典的案例，通过`KVO`在`Model`和`Controller`之间进行通信。如图所示：
![斯坦福大学 KVO示例](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a34bd0dc3714465f98470f27cf0c513f~tplv-k3u1fbpfcp-zoom-1.image)


### 2.6 KVO 触发监听方法的方式
`KVO`触发分为自动触发和手动触发两种方式。
#### 2.6.1 自动触发
① 如果是监听对象特定属性值的改变，通过以下方式改变属性值会触发`KVO`：
* 使用点语法
* 使用`setter`方法
* 使用`KVC`的`setValue:forKey:`方法
* 使用`KVC`的`setValue:forKeyPath:`方法

② 如果是监听集合对象的改变，需要通过`KVC`的`mutableArrayValueForKey:`等方法获得代理对象，并使用代理对象进行操作，当代理对象的内部对象发生改变时，会触发`KVO`。集合对象包含`NSArray`和`NSSet`。
#### 2.6.2 手动触发
① 普通对象属性或是成员变量使用：
```objc
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
```
② `NSArray`对象使用：
```objc
- (void)willChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key;
- (void)didChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key;
```
③ `NSSet`对象使用：
```objc
- (void)willChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key;
- (void)didChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key;
```


## 3. KVO 的进阶使用
### 3.1 observationInfo 属性
* `observationInfo`属性是`NSKeyValueObserving.h`文件中系统通过分类给`NSObject`添加的属性，所以所有继承于`NSObject`的对象都含有该属性；
* 可以通过`observationInfo`属性查看被观察对象的全部观察信息，包括`observer`、`keyPath`、`options`、`context`等。
```objc
@property (nullable) void *observationInfo NS_RETURNS_INNER_POINTER;
```

### 3.2 context 的使用
注册方法`addObserver:forKeyPath:options:context:`中的`context`可以传入任意数据，并且可以在监听方法中接收到这个数据。
* `context`作用：标签-区分，可以更精确的确定被观察对象属性，用于继承、 多监听；也可以用来传值。<br>
&emsp;&emsp;`KVO`只有一个监听回调方法`observeValueForKeyPath:ofObject:change:context:`，我们通常情况下可以在注册方法中指定`context`为`NULL`，并在监听方法中通过`object`和`keyPath`来判断触发`KVO`的来源。<br>
&emsp;&emsp;但是如果存在继承的情况，比如现在有 Person 类和它的两个子类 Teacher 类和 Student 类，person、teacher 和 student 实例对象都对 account 对象的 balance 属性进行观察。问题：<br>
&emsp;&emsp;① 当 balance 发生改变时，应该由谁来处理呢？<br>
&emsp;&emsp;② 如果都由 person 来处理，那么在 Person 类的监听方法中又该怎么判断是自己的事务还是子类对象的事务呢？<br>
&emsp;&emsp;这时候通过使用`context`就可以很好地解决这个问题，在注册方法中为`context`设置一个独一无二的值，然后在监听方法中对`context`值进行检验即可。

* 苹果的推荐用法：用`context`来精确的确定被观察对象属性，使用唯一命名的静态变量的地址作为`context`的值。可以为整个类设置一个`context`，然后在监听方法中通过`object`和`keyPath `来确定被观察属性，这样存在继承的情况就可以通过`context`来判断；也可以为每个被观察对象属性设置不同的`context`，这样使用`context`就可以精确的确定被观察对象属性。
```objc
static void *PersonAccountBalanceContext = &PersonAccountBalanceContext;
static void *PersonAccountInterestRateContext = &PersonAccountInterestRateContext;
```
```objc
- (void)registerAsObserverForAccount:(Account*)account {
    [account addObserver:self
              forKeyPath:@"balance"
                 options:(NSKeyValueObservingOptionNew |
                          NSKeyValueObservingOptionOld)
                 context:PersonAccountBalanceContext];
 
    [account addObserver:self
              forKeyPath:@"interestRate"
                 options:(NSKeyValueObservingOptionNew |
                          NSKeyValueObservingOptionOld)
                  context:PersonAccountInterestRateContext];
}
```
```objc
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
 
    if (context == PersonAccountBalanceContext) {
        // Do something with the balance…
 
    } else if (context == PersonAccountInterestRateContext) {
        // Do something with the interest rate…
 
    } else {
        // Any unrecognized context must belong to super
        [super observeValueForKeyPath:keyPath
                             ofObject:object
                               change:change
                               context:context];
    }
}
```


* `context`优点：嵌套少、性能高、更安全、扩展性强。
* `context`注意点：<br>
① 如果传的是一个对象，必须在移除观察之前持有它的强引用，否则在监听方法中访问`context`就可能导致`Crash`；<br>
② 空传`NULL`而不应该传`nil`。


### 3.3 KVO 监听集合对象
`KVO`可以监听单个属性的变化，也可以监听集合对象的变化。监听集合对象变化时，需要通过`KVC`的`mutableArrayValueForKey:`等方法获得代理对象，并使用代理对象进行操作，当代理对象的内部对象发生改变时，会触发`KVO`的监听方法。集合对象包含`NSArray`和`NSSet`。
（注意：如果直接对集合对象进行操作改变，不会触发`KVO`。）

**示例代码及输出如下：**

观察者 viewController 对被观察对象 person 的 mArray 属性进行监听。
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.person = [HTPerson new];
    self.person.mArray = [NSMutableArray arrayWithCapacity:5];
    [self.person addObserver:self forKeyPath:@"mArray" options:(NSKeyValueObservingOptionNew| NSKeyValueObservingOptionOld) context:NULL];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
//    [self.person.mArray addObject:@"2"]; //如果直接对数组进行操作，不会触发KVO
    NSMutableArray *array = [self.person mutableArrayValueForKey:@"mArray"];
    [array addObject:@"1"];
    [array replaceObjectAtIndex:0 withObject:@"2"];
    [array removeObjectAtIndex:0];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    /*  change 字典的值为：
        {
            indexes：对应的值为数组操作的详细信息，包括索引等
            kind：   对应的值为数组操作的方式：
                     2：代表插入操作
                     3：代表删除操作
                     4：代表替换操作
                     typedef NS_ENUM(NSUInteger, NSKeyValueChange) {
                         NSKeyValueChangeSetting = 1,
                         NSKeyValueChangeInsertion = 2,
                         NSKeyValueChangeRemoval = 3,
                         NSKeyValueChangeReplacement = 4,
                     };
            new/old：如果是插入操作，则字典中只会有new字段，对应的值为插入的元素，前提条件是options中传入了（NSKeyValueObservingOptionNew）
                     如果是删除操作，则字典中只会有old字段，对应的值为删除的元素，前提条件是options中传入了（NSKeyValueObservingOptionOld）
                     如果是替换操作，则字典中new和old字段都可以存在，对应的值为替换后的元素和替换前的元素，前提条件是options中传入了（NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld）
            
        如: indexes = "<_NSCachedIndexSet: 0x600001d092e0>[number of indexes: 1 (in 1 ranges), indexes: (0)]";
            kind = 2; 
            new =     (
                1
            );
        }
     */  
    NSLog(@"%@",change);  
}

- (void)dealloc
{
    [self.person removeObserver:self forKeyPath:@"mArray"];
}
```
> { indexes = "<_NSCachedIndexSet: 0x6000030e5380>[number of indexes: 1 (in 1 ranges), indexes: (0)]"; kind = 2; new =  (1); }<br>
>{ indexes = "<_NSCachedIndexSet: 0x6000030e5380>[number of indexes: 1 (in 1 ranges), indexes: (0)]"; kind = 4; new = (2); old = (1); }<br>
>{ indexes = "<_NSCachedIndexSet: 0x6000030e5380>[number of indexes: 1 (in 1 ranges), indexes: (0)]"; kind = 3; old = (2); }



### 3.4 KVO 的自动触发控制
&emsp;&emsp;可以在被观察对象的类中重写`+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key`方法来控制`KVO`的自动触发。<br>
&emsp;&emsp;如果我们只允许外界观察 person 的 name 属性，可以在 Person 类如下操作。这样外界就只能观察 name 属性，即使外界注册了对 person 对象其它属性的监听，那么在属性发生改变时也不会触发`KVO`。
```objc
// 返回值代表允不允许触发 KVO
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
{
    BOOL automatic = NO;
    if ([key isEqualToString:@"name"]) {
        automatic = YES;
    } else {
        automatic = [super automaticallyNotifiesObserversForKey:key];
    }
    return automatic;
}
```
&emsp;&emsp;也可以实现遵循命名规则为`+ (BOOL)automaticallyNotifiesObserversOf<Key>`的方法来单一控制属性的`KVO`自动触发，`<Key>`为属性名（首字母大写）。
```objc
+ (BOOL)automaticallyNotifiesObserversOfName
{
    return NO;
}
```
>**注意：**
>* 第一个方法的优先级高于第二个方法。如果实现了`automaticallyNotifiesObserversForKey:`方法，并对`<Key>`做了处理，则系统就不会再调用该`<Key>`的`automaticallyNotifiesObserversOf<Key>`方法。
>* `options`指定的`NSKeyValueObservingOptionInitial`触发的`KVO`通知，是无法被`automaticallyNotifiesObserversForKey:`阻止的。
 
### 3.5 KVO 的手动触发

**使用场景：**
* 使用`KVO`监听成员变量值的改变；
* 在某些需要控制监听过程的场景下。比如：为了尽量减少不必要的触发通知操作，或者当多个更改同时具备的时候才调用属性改变的监听方法。

&emsp;&emsp;由于`KVO`的本质，重写`setter`方法来达到可以通知所有观察者对象的目的，所以只有通过`setter`方法或`KVC`方法去修改属性变量值的时候，才会触发`KVO`，直接修改成员变量不会触发`KVO`。<br>
&emsp;&emsp;当我们要使用`KVO`监听成员变量值改变的时候，可以通过在为成员变量赋值的前后手动调用`willChangeValueForKey:`和`didChangeValueForKey:`两个方法来手动触发`KVO`，如：
```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [self.person willChangeValueForKey:@"age"];
    self.person->_age = 18;
    [self.person didChangeValueForKey:@"age"];
}
```
&emsp;&emsp;`NSKeyValueObservingOptionPrior`（分别在值改变前后触发方法，即一次修改有两次触发）的两次触发分别在`willChangeValueForKey:`和`didChangeValueForKey:`的时候进行的。<br>
&emsp;&emsp;如果注册方法中`options`传入`NSKeyValueObservingOptionPrior`，那么可以通过只调用`willChangeValueForKey:`来触发改变前的那次`KVO`，可以用于在属性值即将更改前做一些操作。



### 3.6 KVO 新旧值相等时不触发
&emsp;&emsp;有时候我们可能会有这样的需求，`KVO`监听的属性值修改前后相等的时候，不触发`KVO`的监听方法，可以结合`KVO`的自动触发控制和手动触发来实现。<br>
&emsp;&emsp;例如：对 person 对象的 name 属性注册了`KVO`监听，我们希望在对 name 属性赋值时做一个判断，如果新值和旧值相等，则不触发`KVO`，可以在 Person 类中如下这样实现，将 name 属性值改变的`KVO`触发方式由自动触发改为手动触发。
```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
{
    BOOL automatic = YES;
    if ([key isEqualToString:@"name"]) {
        automatic = NO;
    } else {
        automatic = [super automaticallyNotifiesObserversForKey:key];
    }
    return automatic;
}

- (void)setName:(NSString *)name
{
    if (![_name isEqualToString:name]) {
        [self willChangeValueForKey:@"name"];
        _name = name;
        [self didChangeValueForKey:@"name"];
    } 
}
```

### 3.7 KVO 手动观察集合属性
有些情况下我们想手动观察集合属性，下面以观察数组为例。<br>
关键方法：
```objc
- (void)willChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key;
- (void)didChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key;
```
需要注意的是，根据`KVC`的`NSMutableArray 搜索模式`：<br>
[传送门：iOS - 关于 KVC 的一些总结](https://juejin.im/post/6844904082415550477)
* 至少要实现一个插入和一个删除方法，否则不会触发`KVO`。如
	<br>插入方法：`insertObject:in<Key>AtIndex:`或`insert<Key>:atIndexes:`
	<br>删除方法：`removeObjectFrom<Key>AtIndex:`或`remove<Key>AtIndexes:`
* 可以不实现替换方法，但是如果不实现替换方法，执行替换操作时，`KVO`会把它当成先删除后添加，即会触发两次`KVO`。第一次触发的`KVO`中`change`字典的`old`键的值为替换前的元素，第二次触发的`KVO`中`change`字典的`new`键的值为替换后的元素，前提条件是注册方法中的`options`传入对应的枚举值。
* 如果实现替换方法，则执行替换操作只会触发一次`KVO`，并且`change`字典会同时包含`new`和`old`，前提条件是注册方法中的`options`传入对应的枚举值。
	<br>替换方法：`replaceObjectIn<Key>AtIndex:withObject:`或`replace<Key>AtIndexes:with<Key>:`
* 建议实现替换方法以提高性能。

**示例代码如下：**
```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
{
    BOOL automatic = NO;
    if ([key isEqualToString:@"mArray"]) {
        automatic = NO;
    } else {
        automatic = [super automaticallyNotifiesObserversForKey:key];
    }
    return automatic;
}

- (void)insertMArray:(NSArray *)array atIndexes:(NSIndexSet *)indexes
{
    [self willChange:NSKeyValueChangeInsertion valuesAtIndexes:indexes forKey:@"mArray"];

    [self.mArray insertObjects:array atIndexes:indexes];

    [self didChange:NSKeyValueChangeInsertion valuesAtIndexes:indexes forKey:@"mArray"];
}

- (void)removeMArrayAtIndexes:(NSIndexSet *)indexes
{
    [self willChange:NSKeyValueChangeRemoval valuesAtIndexes:indexes forKey:@"mArray"];

    [self.mArray removeObjectsAtIndexes:indexes];

    [self didChange:NSKeyValueChangeRemoval valuesAtIndexes:indexes forKey:@"mArray"];
}

- (void)replaceMArrayAtIndexes:(NSIndexSet *)indexes withMArray:(NSArray *)array
{
    [self willChange:NSKeyValueChangeReplacement valuesAtIndexes:indexes forKey:@"mArray"];

    [self.mArray replaceObjectsAtIndexes:indexes withObjects:array];

    [self didChange:NSKeyValueChangeReplacement valuesAtIndexes:indexes forKey:@"mArray"];
}
```

### 3.8 KVO 的依赖观察
#### 3.8.1 一对一关系
&emsp;&emsp;有些情况下，一个属性的改变依赖于别的一个或多个属性的改变，也就是说当别的属性改了，这个属性也会跟着改变。<br>
&emsp;&emsp;比如我们想要对 Download 类中的 downloadProgress 属性进行`KVO`监听，该属性的改变依赖于 writtenData 和 totalData 属性的改变。观察者监听了 downloadProgress ，当 writtenData 和 totalData 属性值改变时，观察者也应该被通知。以下有两种方法可以解决这个问题。
1. 重写以下方法来指明 downloadProgress 属性依赖于 writtenData 和 totalData：
```objc
+ (NSSet<NSString *> *)keyPathsForValuesAffectingValueForKey:(NSString *)key
{
    NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];
    if ([key isEqualToString:@"downloadProgress"]) {
        NSArray *affectingKeys = @[@"writtenData",@"totalData"];
        keyPaths = [keyPaths setByAddingObjectsFromArray:affectingKeys];
    }
    return keyPaths;
}
```
2. 实现一个遵循命名规则为`keyPathsForValuesAffecting<Key>`的类方法，`<Key>`是依赖于其他值的属性名（首字母大写）：
```objc
+ (NSSet<NSString *> *)keyPathsForValuesAffectingDownloadProgress
{
    return [NSSet setWithObjects:@"writtenData",@"totalData", nil];
}
```
>**注意：** 以上两个方法可以同时存在，且都会调用，但是最终结果会以`keyPathsForValuesAffectingValueForKey:`为准。


#### 3.8.2 一对多关系
&emsp;&emsp;以上方法在观察集合属性时就不管用了。例如，假如你有一个 Department 类，它有一个装有 Employee 类的实例对象的数组，Employee 类有 salary 属性。你希望 Department 类有一个 totalSalary 属性来计算所有员工的薪水，也就是在这个关系中 Department 的 totalSalary 依赖于所有 Employee 实例对象的 salary 属性。以下有两种方法可以解决这个问题。

1. 你可以用`KVO`将 parent（比如 Department ）作为所有 children（比如 Employee ）相关属性的观察者。你必须在把 child 添加或删除到 parent 时把 parent 作为 child 的观察者添加或删除。在`observeValueForKeyPath:ofObject:change:context:`方法中我们可以针对被依赖项的变更来更新依赖项的值：
```objc
#import "Department.h"

static void *totalSalaryContext = &totalSalaryContext;

@interface Department ()
@property (nonatomic,strong)NSArray<Employee *> *employees;
@property (nonatomic,strong)NSNumber *totalSalary;

@end


@implementation Department

- (instancetype)initWithEmployees:(NSArray *)employees
{
    self = [super init];
    if (self) {
        self.employees = [employees copy];
        for (Employee *em in self.employees) {
            [em addObserver:self forKeyPath:@"salary" options:NSKeyValueObservingOptionNew context:totalSalaryContext];
        }
    }
    return self;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
 
    if (context == totalSalaryContext) {
        [self setTotalSalary:[self valueForKeyPath:@"employees.@sum.salary"]];
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
    
}
 
- (void)setTotalSalary:(NSNumber *)totalSalary
{
    if (_totalSalary != totalSalary) {
        [self willChangeValueForKey:@"totalSalary"];
        _totalSalary = totalSalary;
        [self didChangeValueForKey:@"totalSalary"];
    }
}

- (void)dealloc
{
    for (Employee *em in self.employees) {
        [em removeObserver:self forKeyPath:@"salary" context:totalSalaryContext];
    }
}

@end
```
2. 使用`iOS`中观察者模式的另一种实现方式：通知 (`NSNotification`) 。


## 4. KVO 的使用注意
### 4.1 移除观察者的注意点
* 在调用`KVO`注册方法后，`KVO`并不会对观察者进行强引用，所以需要注意观察者的生命周期。至少需要在观察者销毁之前，调用`KVO`移除方法移除观察者，否则如果在观察者被释放后，再次触发`KVO`监听方法就会导致`Crash`。
* `KVO`的注册方法和移除方法应该是成对的，如果重复调用移除方法，就会抛出异常`NSRangeException`并导致程序`Crash`。
* 苹果官方推荐的方式是，在观察者初始化期间（`init`或者`viewDidLoad`的时候）注册为观察者，在释放过程中（`dealloc`时）调用移除方法，这样可以保证它们是成对出现的，是一种比较理想的使用方式。


### 4.2 防止多次注册和移除相同的 KVO
&emsp;&emsp;有时候我们难以避免多次注册和移除相同的`KVO`，或者移除了一个未注册的观察者，从而产生可能会导致`Crash`的风险。<br>
&emsp;&emsp;三种解决方案：[黑科技防止多次添加删除KVO出现的问题](https://www.jianshu.com/p/6c6f3a24b1ef)

* 利用 `@try @catch`（只能针对删除多次`KVO`的情况下）；
	<br>给`NSObject`增加一个分类，然后利用`Runtime API`交换系统的`removeObserver`方法，在里面添加`@try @catch`。
* 利用 模型数组 进行存储记录；
* 利用 `observationInfo` 里私有属性。

### 4.3 其它注意点
* 如果对象被注册成为观察者，则该对象必须能响应监听方法，即该对象所属类中必须实现监听方法。当被观察对象属性发生改变时就会调用监听方法。如果没有实现就会导致`Crash`。所以`KVO`三部曲缺一不可。
* `keyPath `传入的是一个字符串，为避免写错，可以使用`NSStringFromSelector(@selector(propertyName))`，将属性的`getter`方法`SEL`转换成字符串，在编译阶段对`keyPath`进行检验。
* 如果注册方法中`context`传的是一个对象，必须在移除观察之前持有它的强引用，否则在监听方法中访问`context`就可能导致`Crash`。
    * 可以使用`__bridge_retained`桥接剥夺对象的内存管理权，但必须记得在不需要该对象时释放它，否则内存泄露。关于桥接可以参阅[《iOS - 老生常谈内存管理（三）：ARC 面世  —— Toll-Free Bridging》](https://juejin.im/post/6844904130431942670)；
    * 或者使用全局变量。
* 如果是监听集合对象的改变，需要通过`KVC`的`mutableArrayValueForKey:`等方法获得代理对象，并使用代理对象进行操作，当代理对象的内部对象发生改变时，会触发`KVO`。如果直接对集合对象进行操作改变，不会触发`KVO`。
* 在观察者类的监听方法中，应该为无法识别的`context`或者`object`、`keyPath `调用父类的实现`[super observeValueForKeyPath:keyPath ofObject:object change:change context:context];`。



## 5. KVO 的实现原理

>**Key-Value Observing Implementation Details**
>* Automatic key-value observing is implemented using a technique called *isa-swizzling*.
>* The `isa` pointer, as the name suggests, points to the object's class which maintains a dispatch table. This dispatch table essentially contains pointers to the methods the class implements, among other data.
>* When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance.
>* You should never rely on the `isa` pointer to determine class membership. Instead, you should use the `class` method to determine the class of an object instance.

&emsp;&emsp;以上是苹果官方对`KVO`实现的解释，只说明了`KVO`是使用`isa-swizzling`技术来实现的，并没有做过多介绍。


### 5.1 isa-swizzling
&emsp;&emsp;苹果使用了`isa`混写技术（`isa-swizzling`）来实现`KVO`。当我们调用了`addObserver:forKeyPath:options:context:`方法，为`instance`被观察对象添加`KVO`监听后，系统会在运行时利用`Runtime API`动态创建`instance`对象所属类`A`的子类`NSKVONotifying_A`，并且让`instance`对象的`isa`指向这个全新的子类，并重写原类`A`的被观察属性的`setter`方法来达到可以通知所有观察者对象的目的。<br>
&emsp;&emsp;这个子类的`isa`指针指向它自己的`meta-class`对象，而不是原类的`meta-class`对象。<br>
&emsp;&emsp;重写的`setter`方法的`SEL`对应的`IMP`为`Foundation`中的`_NSSetXXXValueAndNotify`函数（`XXX`为`Key`的数据类型），当被观察对象的属性发送改变时，会调用`_NSSetXXXValueAndNotify`函数，这个函数中会调用：
* `willChangeValueForKey:`方法
* 父类原来的`setter`方法
* `didChangeValueForKey:`方法（内部会触发监听器即观察对象`observer`的监听方法：`observeValueForKeyPath:ofObject:change:context:`）

&emsp;&emsp;在移除`KVO`监听后，被观察对象的`isa`会指回原类`A`，但是`NSKVONotifying_A`类并没有销毁，还保存在内存中。


### 5.2 KVO 动态生成的子类都有哪些方法
&emsp;&emsp;`NSKVONotifying_A`除了重写了`setter`方法，还重写了`class`、`dealloc`、`_isKVOA`这三个方法（可以使用`runtime`的`class_copyMethodList`函数打印方法列表获得），其中：
* `class`：`class`方法中返回的是父类的`class`对象，目的是为了不让外界知道`KVO`动态生成类的存在；
* `dealloc`：释放`KVO`使用过程中产生的东西；
* `_isKVOA`：用来标志它是一个`KVO`的类。



## 6. FBKVOController
### 6.1 系统 KVO 的缺点
* 使用比较麻烦，需要三个步骤：添加/注册`KVO`监听、实现监听方法以接收属性改变通知、 移除`KVO`监听，缺一不可；
* 需要手动移除观察者，移除观察者的时机必须合适，还不能重复移除；
* 注册观察者的代码和事件发生处的代码上下文不同，传递上下文`context`是通过`void *`指针；
* 需要实现`-observeValueForKeyPath:ofObject:change:context:`方法，比较麻烦；
* 在复杂的业务逻辑中，准确判断被观察者相对比较麻烦，有多个被观测的对象和属性时，需要在方法中写大量的`if`进行判断。



### 6.2 FBKVOController 的介绍
`FBKVOController`是 Facebook 开源的一个基于系统`KVO`实现的框架。支持`Objective-C`和`Swift`语言。<br>
GitHub：[https://github.com/facebook/KVOController](https://github.com/facebook/KVOController)

### 6.3 FBKVOController 的优点
* 会自动移除观察者；
* 函数式编程，可以一行代码实现系统`KVO`的三个步骤；
* 实现`KVO`与事件发生处的代码上下文相同，不需要跨方法传参数；
* 增加了`block`和`SEL`自定义操作对`NSKeyValueObserving`回调的处理支持；
* 每一个`keyPath`会对应一个`block`或者`SEL`，不需要使用`if`判断`keyPath`；
* 可以同时对一个对象的多个属性进行监听，写法简洁；
* 线程安全。

### 6.4 FBKVOController 的使用
`FBKVOController`实现了观察者和被观察者的角色反转，系统的`KVO`是被观察者添加观察者，而`FBKVO`实现了观察者主动去添加被观察者，实现了角色上的反转，使用比较方便。
```objc
// create KVO controller with observer
FBKVOController *KVOController = [FBKVOController controllerWithObserver:self];
self.KVOController = KVOController;

// observe clock date property
// 使用 block
[self.KVOController observe:clock keyPath:@"date" options:NSKeyValueObservingOptionInitial|NSKeyValueObservingOptionNew block:^(ClockView *clockView, Clock *clock, NSDictionary *change) {

  // update clock view with new value
  clockView.date = change[NSKeyValueChangeNewKey];
}];

// 使用 SEL
[self.KVOController observe:clock keyPath:@"date" options:NSKeyValueObservingOptionInitial|NSKeyValueObservingOptionNew action:@selector(updateClockWithDateChange:)];
```

### 6.5 FBKVOController 的解析
[如何优雅地使用KVO（简书）](https://www.jianshu.com/p/4c0c36b88db6)
<br>[iOS - FBKVOController 实现原理（简书）](https://www.jianshu.com/p/77b1d627780e)


## 参考
[Key-Value Observing Programming Guide（苹果官方文档）](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177i)
<br>[iOS - 关于 KVC 的一些总结（掘金）](https://juejin.im/post/6844904082415550477)
<br>[KVO原理分析及使用进阶（简书）](https://www.jianshu.com/p/badf5cac0130)
<br>[iOS开发 - 黑科技防止多次添加删除KVO出现的问题（简书）](https://www.jianshu.com/p/6c6f3a24b1ef)
<br>[谈谈 KVO（简书）](https://www.jianshu.com/p/cfd553f250f9)
<br>[GitHub/facebook/KVOController（GitHub）](https://github.com/facebook/KVOController)
<br>[如何优雅地使用KVO（简书）](https://www.jianshu.com/p/4c0c36b88db6)
<br>[iOS - FBKVOController 实现原理（简书）](https://www.jianshu.com/p/77b1d627780e)



