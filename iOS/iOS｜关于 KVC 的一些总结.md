
![](https://user-gold-cdn.xitu.io/2020/4/21/17198b2eec822648?w=4093&h=2707&f=png&s=1327881)


## 1. 什么是 KVC
* `KVC`的全称是`Key-Value Coding`（键值编码），是由`NSKeyValueCoding`非正式协议启用的一种机制，对象采用这种机制来提供对其属性的间接访问，可以通过字符串来访问一个对象的成员变量或其关联的存取方法（`getter or setter`）。
* 通常，我们可以直接通过存取方法或变量名来访问对象的属性。我们也可以使用`KVC`间接访问对象的属性，并且`KVC`还可以访问私有变量。某些情况下，`KVC`还可以帮助简化代码。
* `KVC`是许多其他 Cocoa 技术的基础概念，比如 [KVO](https://juejin.im/post/6844903972528979976)、Cocoa bindings、Core Data、AppleScript-ability 等等。


## 2. 访问对象属性
### 常用 API
```objc
- (nullable id)valueForKey:(NSString *)key;         // 通过 key 来取值
- (nullable id)valueForKeyPath:(NSString *)keyPath; // 通过 keyPath 来取值

- (void)setValue:(nullable id)value forKey:(NSString *)key;         // 通过 key 来赋值
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath; // 通过 keyPath 来赋值
```
### 基础操作

如下是 BankAccount 类的声明：
```objc
@interface BankAccount : NSObject
@property (nonatomic) NSNumber* currentBalance;              // An attribute
@property (nonatomic) Person* owner;                         // A to-one relation
@property (nonatomic) NSArray< Transaction* >* transactions; // A to-many relation
@end
```
对于 BankAccount 的实例对象`myAccount`。
我们可以使用`setter`方法为`currentBalance`属性赋值，这是直接的，但缺乏灵活性。
```objc
[myAccount setCurrentBalance:@(100.0)];
```
我们也可以通过`KVC`间接为`currentBalance`属性赋值，通过其键`Key`设置值。
```objc
[myAccount setValue:@(100.0) forKey:@"currentBalance"];
```
### KeyPath
`KVC`还支持多级访问，`KeyPath`用法跟点语法相同。
例如：我们想对`myAccount`的`owner`属性的`address`属性的`street`属性赋值，其`KeyPath`为`owner.address.street`。
```objc
[myAccount setValue:@"地址" forKeyPath:@"owner.address.street"];
```

### 多值操作


给定一组`Key`，获得一组`value`，以字典的形式返回。该方法为数组中的每个`Key`调用`valueForKey:`方法。
```objc
- (NSDictionary<NSString *,id> *)dictionaryWithValuesForKeys:(NSArray<NSString *> *)keys;
```
将指定字典中的值设置到消息接收者的属性中，使用字典的`Key`标识属性。默认实现是为每个键值对调用`setValue:forKey:`方法 ，会根据需要用`nil`替换`NSNull`对象。
```objc
- (void)setValuesForKeysWithDictionary:(NSDictionary<NSString *,id> *)keyedValues;
```

## 3. 访问集合属性
我们可以像访问其它对象一样使用`valueForKey:`或`setValue:forKey:`方法来获取或设置集合对象（主要指`NSArray`和`NSSet`）。但是，当我们要操作集合对象的内容，比如添加或者删除元素时，通过`KVC`的可变代理方法获取集合代理对象是最有效的。<br>
根据`KVO`的实现原理，是在运行时动态生成子类并重写`setter`方法来达到可以通知所有观察者对象的目的，因此我们对集合对象进行操作是不会触发`KVO`的。当我们要使用`KVO`监听集合对象变化时，需要通过`KVC`的可变代理方法获取集合代理对象，然后对代理对象进行操作。当代理对象的内部对象发生改变时，会触发`KVO`的监听方法。<br>
[传送门：iOS - 关于 KVO 的一些总结](https://juejin.im/post/6844903972528979976)

`KVC`提供了三种不同的代理对象访问的代理方法，每种都有`Key`和`KeyPath`两种方法。
*   `mutableArrayValueForKey: `和 `mutableArrayValueForKeyPath:`

    返回`NSMutableArray`对象的代理对象。

*   `mutableSetValueForKey:` 和 `mutableSetValueForKeyPath:`

    返回`NSMutableSet`对象的代理对象。

* `mutableOrderedSetValueForKey:` 和 `mutableOrderedSetValueForKeyPath:`

    返回`NSMutableOrderedSet`对象的代理对象。


## 4. 使用集合运算符
`KVC`的`valueForKeyPath:`方法除了可以取出属性值以外，还可以在`KeyPath`中嵌套集合运算符，来对集合对象进行操作。

以下是`KeyPath`集合运算符的格式，主要分为 3 个部分。
* Left key path：左键路径，要操作的集合对象，如果消息接收者就是集合对象，则可以省略 Left 部分；
* Collection operator：集合运算符；
* Right key path：右键路径，要进行运算的集合中的属性。

![图 4-1 KeyPath 集合运算符格式.png](https://user-gold-cdn.xitu.io/2020/3/5/170aad31383f117f?w=503&h=49&f=png&s=5528)

**集合运算符主要分为三类：**
* ① 聚合运算符：以某种方式合并集合中的对象，并返回右键路径中指定的属性的数据类型匹配的一个对象，一般返回`NSNumber`实例。
* ② 数组运算符：根据运算符的条件，将符合条件的对象以一个`NSArray`实例返回。
* ③ 嵌套运算符：处理集合对象中嵌套其他集合对象的情况，并根据运算符返回一个`NSArray`或`NSSet`实例。

### 示例
如下是 BankAccount 类和 Transaction 类的声明。BankAccount 中有一个 transactions 数组属性，其元素为 Transaction 类型。Transaction 类中定义了 3 个属性，分别为收款人、金额、日期。
```objc
@interface BankAccount : NSObject
@property (nonatomic) NSArray< Transaction* >* transactions; // A to-many 
@end

@interface Transaction : NSObject
@property (nonatomic) NSString* payee;   // To whom
@property (nonatomic) NSNumber* amount;  // How much
@property (nonatomic) NSDate* date;      // When
@end
```
下表是为了演示集合运算符使用而给出的 transactions 数组的数据。
payee|    amount|    date
:--|:--|:--
Green Power|            $120.00|        Dec 1, 2015
Green Power|            $150.00|        Jan 1, 2016
Green Power|            $170.00|        Feb 1, 2016
Car Loan|            $250.00|        Jan 15, 2016
Car Loan|            $250.00|        Feb 15, 2016
Car Loan|            $250.00|        Mar 15, 2016
General Cable|    $120.00|        Dec 1, 2015
General Cable|    $155.00|        Jan 1, 2016
General Cable|    $120.00|        Feb 1, 2016
Mortgage|            $1,250.00|    Jan 15, 2016
Mortgage|            $1,250.00|    Feb 15, 2016
Mortgage|            $1,250.00|    Mar 15, 2016
Animal Hospital|    $600.00|        Jul 15, 2016

### 聚合运算符
以某种方式合并集合中的对象，并返回右键路径中指定的属性的数据类型匹配的一个对象，一般返回`NSNumber`实例。

#### @avg
读取集合中每个元素的右键路径指定的属性，将其转换为`double`类型 (`nil`用 0 替代)，并计算这些值的算术平均值。然后将结果以`NSNumber`实例返回。
```objc
// 计算上表中 amount 的平均值。
NSNumber *transactionAverage = [self.transactions valueForKeyPath:@"@avg.amount"];
// transactionAverage 格式化的结果为 $ 456.54。
```
#### @count
计算集合中的元素个数，以`NSNumber`实例返回。
```objc
// 计算 transactions 集合中的元素个数。
NSNumber *numberOfTransactions = [self.transactions valueForKeyPath:@"@count"];
// numberOfTransactions 的值为 13。
```
>**备注：**`@count`运算符比较特别，它不需要写右键路径，即使写了也会被忽略。
#### @sum
读取集合中每个元素的右键路径指定的属性，将其转换为`double`类型 (`nil`用 0 替代)，并计算这些值的总和。然后将结果以`NSNumber`实例返回。
```objc
// 计算上表中 amount 的总和。
NSNumber *amountSum = [self.transactions valueForKeyPath:@"@sum.amount"];
// amountSum 的结果为 $ 5935.00。
```
#### @max
返回集合中右键路径指定的属性的最大值。
```objc
// 获取日期的最大值。
NSDate *latestDate = [self.transactions valueForKeyPath:@"@max.date"];
// latestDate 的值为 Jul 15, 2016.
```
#### @min
返回集合中右键路径指定的属性的最小值。
```objc
// 获取日期的最小值。
NSDate *earliestDate = [self.transactions valueForKeyPath:@"@min.date"];
// earliestDate 的值为 Dec 1, 2015.
```
>**备注：**`@max`和`@min`根据右键路径指定的属性在集合中搜索，搜索使用`compare:`方法进行比较，许多基础类 (如`NSNumber类`) 中都有定义。因此，右键路径指定的属性必须能响应`compare:`消息。搜索忽略值为`nil`的集合项。可以通过重写`compare:`方法对搜索过程进行控制。


### 数组运算符
根据运算符的条件，将符合条件的对象以一个`NSArray`实例返回。

#### @unionOfObjects
读取数组中每个元素的右键路径指定的属性，放在一个`NSArray`实例中并返回。
```objc
// 获取集合中的所有 payee 对象。
NSArray *payees = [self.transactions valueForKeyPath:@"@unionOfObjects.payee"];
// payees 数组包含以下字符串：Green Power, Green Power, Green Power, Car Loan, Car Loan, Car Loan, General Cable, General Cable, General Cable, Mortgage, Mortgage, Mortgage, Animal Hospital。
```
#### @distinctUnionOfObjects
读取数组中每个元素的右键路径指定的属性，放在一个`NSArray`实例中，将数组进行去重后返回。
```objc
// 获取集合中的所有不同的 payee 对象。
NSArray *distinctPayees = [self.transactions valueForKeyPath:@"@distinctUnionOfObjects.payee"];
// distinctPayees 数组包含以下字符串：Car Loan, General Cable, Animal Hospital, Green Power, Mortgage。
```
>**注意：** 在使用数组运算符时，如果有任何操作的对象为`nil`，则`valueForKeyPath:`方法将引发异常。

### 嵌套运算符
处理集合对象中嵌套其他集合对象的情况，并根据运算符返回一个`NSArray`或`NSSet`实例。

如下 moreTransactions 是装着 transaction 对象的数组，arrayOfArrays 数组中嵌套了 self.transactions 和 moreTransactions 两个数组。
```objc
NSArray* moreTransactions = @[<# transaction data #>];
NSArray* arrayOfArrays = @[self.transactions, moreTransactions];
```
下表是 moreTransactions 数组的数据。

payee|    amount|    date|
:--|:--|:--
General Cable - Cottage|    $120.00|            Dec 18, 2015
General Cable - Cottage|    $155.00|            Jan 9, 2016
General Cable - Cottage|    $120.00|             Dec 1, 2016
Second Mortgage|                    $1,250.00|    Nov 15, 2016
Second Mortgage|                    $1,250.00|    Sep 20, 2016
Second Mortgage|                    $1,250.00|    Feb 12, 2016
Hobby Shop|                            $600.00|         Jun 14, 2016

#### @unionOfArrays
读取集合中的每个集合中的每个元素的右键路径指定的属性，放在一个`NSArray`实例中并返回。
```objc
// 获取 arrayOfArrays 集合中的每个集合中的所有 payee 对象。
NSArray *collectedPayees = [arrayOfArrays valueForKeyPath:@"@unionOfArrays.payee"];
// collectedPayees 数组包含以下字符串：Green Power, Green Power, Green Power, Car Loan, Car Loan, Car Loan, General Cable, General Cable, General Cable, Mortgage, Mortgage, Mortgage, Animal Hospital, General Cable - Cottage, General Cable - Cottage, General Cable - Cottage, Second Mortgage, Second Mortgage, Second Mortgage, Hobby Shop.
```
#### @distinctUnionOfArrays
读取集合中的每个集合中的每个元素的右键路径指定的属性，放在一个`NSArray`实例中，将数组进行去重后返回。
```objc
// 获取 arrayOfArrays 集合中的每个集合中的所有不同的 payee 对象。
NSArray *collectedDistinctPayees = [arrayOfArrays valueForKeyPath:@"@distinctUnionOfArrays.payee"];
// collectedDistinctPayees 数组包含以下字符串：Hobby Shop, Mortgage, Animal Hospital, Second Mortgage, Car Loan, General Cable - Cottage, General Cable, Green Power.
```
#### @distinctUnionOfSets
读取集合中的每个集合中的每个元素的右键路径指定的属性，放在一个`NSSet`实例中，去重后返回。
```objc
NSSet *collectedDistinctPayees = [setOfSets valueForKeyPath:@"@distinctUnionOfSets.payee"];
```

>**注意：**
>* 在使用嵌套运算符时，`valueForKeyPath:`内部会根据运算符创建一个`NSMutableArray`或`NSMutableSet`对象，将集合中的`array`和`set`添加进去再进行操作。如果集合中有非集合元素，会导致`Crash`。
>* 使用`unionOfArrays`或`distinctUnionOfArrays`运算符，消息接收者应该是`arrayOfArrays`类型，即`NSArray< NSArray* >* arrayOfArrays;`；使用`distinctUnionOfSets`运算符，消息接收者应该是`setOfSets`或者`arrayOfSets`类型。否则会发生异常。
>* 在使用嵌套运算符时，如果有任何操作的对象为`nil`， 则`valueForKeyPath:`方法将引发异常。

### 拓展
如果集合中的对象都是`NSNumber`，右键路径可以用`self`。
```objc
    NSArray *array = @[@1, @2, @3, @4, @5];
    NSNumber *sum = [array valueForKeyPath:@"@sum.self"];
    NSLog(@"%d",[sum intValue]); 
```

## 5. 自定义集合运算符
上面介绍了`KVC`为我们提供的集合运算符，我们能不能自定义呢？

我们使用`Runtime`打印`NSArray`类的方法列表：
```objc
- (void)printNSArrayMethods
{
    u_int count;
    Method *methods = class_copyMethodList([NSArray class], &count);
    for (int i = 0; i < count ; i++)
    {
        Method method = methods[i];
        SEL sel = method_getName(method);
        NSLog(@"%d---%@", i, NSStringFromSelector(sel));
    }
    free(methods);
}
```
```
0---mr_isEqualToOutputDevicesArray:
1---mr_containsAnyOf:
2---mr_map:
3---sg_enumerateChunksOfSize:usingBlock:
4---_pas_mappedArrayWithTransform:
5---_pas_shuffledArrayUsingRng:
......
```
方法很多，我们搜索关键字`avg`、`count`、`sum`等`KVC`为我们提供的集合运算符，发现都有对应的方法`_<operatorKey>ForKeyPath:`。
```
267---_avgForKeyPath:
268---_countForKeyPath:
264---_sumForKeyPath:
269---_maxForKeyPath:
270---_minForKeyPath:
266---_unionOfObjectsForKeyPath:
273---_distinctUnionOfObjectsForKeyPath:
265---_unionOfArraysForKeyPath:
272---_distinctUnionOfArraysForKeyPath:
274---_distinctUnionOfSetsForKeyPath:
```

>**注意：** 我们再来看一下`NSSet`类支持哪些集合运算符：
>```
>50---_sumForKeyPath:
>51---_avgForKeyPath:
>52---_countForKeyPath:
>53---_maxForKeyPath:
>54---_minForKeyPath:
>55---_distinctUnionOfArraysForKeyPath:
>56---_distinctUnionOfObjectsForKeyPath:
>57---_distinctUnionOfSetsForKeyPath:
>```
>可见`NSSet`类不支持`@unionOfObjects`和`@unionOfArrays`运算符，如果使用了就会抛出异常`NSInvalidArgumentException`并导致程序崩溃，reason: `[<__NSSetI 0x6000017a12f0> valueForKeyPath:]: this class does not implement the unionOfArrays operation.`不支持该运算符。
>
>而`NSArray`类虽然支持`@distinctUnionOfSets`运算符，但其必须是`arrayOfSets`类型，即`NSArray< NSSet* >* arrayOfSets;`。因为`_distinctUnionOfSetsForKeyPath`方法中会创建一个`NSMutableSet`实例，并调用`unionSet:`方法将集合中的`set`的元素添加进去再进行操作。如果是`arrayOfArrays`类型就会抛出异常`NSInvalidArgumentException`并导致程序崩溃，reason: `'*** -[NSMutableSet unionSet:]: set argument is not an NSSet'`即集合中有非`NSSet`元素。



我们尝试为`NSArray`添加一个分类，并定义一个`_medianForKeyPath:`方法，用来获取`NSArray`中的中位数。
```objc
#import <Foundation/Foundation.h>
@interface NSArray (HTOperator)
- (NSNumber *)_medianForKeyPath:(NSString *)keyPath;
@end

#import "NSArray+HTOperator.h"
@implementation NSArray (HTOperator)
- (NSNumber *)_medianForKeyPath:(NSString *)keyPath {
    //排序
    NSArray *sortedArray = [self sortedArrayUsingSelector:@selector(compare:)];
    double median;
    if (self.count % 2 == 0) {
        NSInteger index1 = sortedArray.count * 0.5;
        NSInteger index2 = sortedArray.count * 0.5 - 1;
        median = ([[sortedArray objectAtIndex:index1] doubleValue] + [[sortedArray objectAtIndex:index2] doubleValue]) * 0.5;        
    } else {
        NSInteger index = (sortedArray.count-1) * 0.5;
        median = [[sortedArray objectAtIndex:index] doubleValue];
    }
    return [NSNumber numberWithDouble:median];
}
```
测试。
```objc
    NSArray *array = @[@9, @7, @8, @2, @6, @3];
    NSNumber *num = [array valueForKeyPath:@"@median.self"];
    NSLog(@"%f",[num doubleValue]);
    // 6.500000
```

## 6. 非对象值处理
`KVC`支持基础数据类型和结构体，在使用`KVC`进行赋值或取值的时候，会自动在非对象值和对象值之间进行转换。
* 当进行取值如`valueForKey:`时，如果返回值非对象，会使用该值初始化一个`NSNumber`（用于基础数据类型）或`NSValue`（用于结构体）实例，然后返回该实例。
* 当进行赋值如`setValue:forKey:`时，如果`key`的数据类型非对象，则会发送一条`<type>Value`消息给`value`对象以提取基础数据，然后赋值给`key`。

>**注意：**
>* 因为`Swift`中的所有属性都是对象，所以这里仅适用于`Objective-C`属性。
>* 当进行赋值如`setValue:forKey:`时，如果`key`的数据类型是非对象类型，则`value`就禁止传`nil`。否则会调用`setNilValueForKey:`方法，该方法的默认实现抛出异常`NSInvalidArgumentException`，并导致程序`Crash`。

下表是`KVC`对于基础数据类型和`NSNumber`对象之间的转换。

Data type|                    Creation method|                            Accessor method
:--|:--|:--
BOOL|                         numberWithBool:|                            boolValue (in iOS) <br>charValue (in macOS)*
char|                         numberWithChar:|    charValue
double|    numberWithDouble:|    doubleValue
float|     numberWithFloat:|    floatValue
int|    numberWithInt:|    intValue
long|    numberWithLong:|    longValue
long long|    numberWithLongLong:|    longLongValue
short|    numberWithShort:|    shortValue
unsigned char|    numberWithUnsignedChar:|    unsignedChar
unsigned int|    numberWithUnsignedInt:|    unsignedInt
unsigned long|    numberWithUnsignedLong:|    unsignedLong
unsigned long long|    numberWithUnsignedLongLong:|    unsignedLongLong
unsigned short|    numberWithUnsignedShort:|    unsignedShort




下表是`KVC`对于结构体类型和`NSValue`对象之间的转换。

Data type| Creation method| Accessor method
:--|:--|:--
CGPoint|     valueWithCGPoint:|   CGPointValue 
CGRect|      valueWithCGRect:|    CGRectValue 
CGSize|      valueWithCGSize:|    CGSizeValue
NSRange|   valueWithRange:|      rangeValue 

除了以上`CGPoint`、`CGRect`、`CGSize`和`NSRange`类型的结构体可以和`NSValue`对象之间进行转换，我们自定义的结构体也可以包装成`NSValue`对象，示例如下。
```objc
typedef struct {
    float x, y, z;
} ThreeFloats;
 
@interface MyClass
@property (nonatomic) ThreeFloats threeFloats;
@end
```
```objc
// 取值
NSValue* result = [myClass valueForKey:@"threeFloats"];
```
```objc
// 赋值
ThreeFloats floats = {1., 2., 3.};
NSValue* value = [NSValue valueWithBytes:&floats objCType:@encode(ThreeFloats)];
[myClass setValue:value forKey:@"threeFloats"];
```

## 7. 属性验证
`KVC`提供了属性验证的方法，如下。我们可以在使用`KVC`赋值前验证能否为这个`key`赋值指定`value`。
<br>`validateValue`方法的默认实现是查看消息接收者类中是否实现了遵循命名规则为`validate<Key>:error:`的方法，如果有的话就返回调用该方法的结果；如果没有的话，则默认验证成功并返回`YES`。我们可以在消息接收者类中实现`validate<Key>:error:`的方法来自定义逻辑返回`YES`或`NO`。

```objc
- (BOOL)validateValue:(id  _Nullable *)value 
               forKey:(NSString *)key 
                error:(NSError * _Nullable *)error;

- (BOOL)validateValue:(inout id  _Nullable *)ioValue 
           forKeyPath:(NSString *)inKeyPath 
                error:(out NSError * _Nullable *)outError;
```

**示例**

在`Person`类中实现了`validateName:error:`方法，验证给`name`赋的值是不是`jack`。
```objc
// ViewController.m
    Person *person = [[Person alloc] init];
    NSString *value = @"rose";
    NSString *key = @"name";
    NSError  *error;
    BOOL result = [person validateValue:&value forKey:key error:&error];
    
    if (error) {
        NSLog(@"error = %@", error);
        return;
    }
    NSLog(@"%d",result);

// Person.m
- (BOOL)validateName:(id *)value error:(out NSError * _Nullable __autoreleasing *)outError
{
    NSString *name = *value;
    BOOL result = NO;
    if ([name isEqualToString:@"jack"]) {
        result = YES;
    }
    return result;
}
// 打印：0
```
>**备注：** 默认情况下，`KVC`是不会自动验证属性的。


## 8. 搜索规则
除了了解`KVC`的使用，了解`KVC`取值和赋值过程的工作原理也是很有必要的。

### 基本的 Getter 搜索模式
以下是`valueForKey:`方法的默认实现，给定一个`key`作为输入参数，在消息接收者类中操作，执行以下过程。
* ① 按照`get<Key>`、`<key>`、`is<Key>`、`_<key>`顺序查找方法。
    <br>如果找到就调用取值并执行⑤，否则执行②；
* ② 查找`countOf<Key>`、`objectIn<Key>AtIndex:`、`<key>AtIndexes:`命名的方法。
    <br>如果找到第一个和后面两个中的至少一个，则创建一个能够响应所有`NSArray`的方法的集合代理对象（类型为`NSKeyValueArray`，继承自`NSArray`），并返回该对象。否则执行③；
    * 代理对象随后将其接收到的任何`NSArray`消息转换为`countOf<Key>`、`objectIn<Key>AtIndex:`、`<Key>AtIndexes:`消息的组合，并将其发送给`KVC`调用方。如果原始对象还实现了一个名为`get<Key>:range:`的可选方法，则代理对象也会在适当时使用该方法。
    * 当`KVC`调用方与代理对象一起工作时，允许底层属性的行为如同`NSArray`一样，即使它不是`NSArray`。
* ③ 查找`countOf<Key>`、`enumeratorOf<Key>`、`memberOf<Key>:`命名的方法。
    <br>如果三个方法都找到，则创建一个能够响应所有`NSSet`的方法的集合代理对象（类型为`NSKeyValueSet`，继承自`NSSet`），并返回该对象。否则执行④；
    * 代理对象随后将其接收到的任何`NSSet`消息转换为`countOf<Key>`、`enumeratorOf<Key>`、`memberOf<Key>:`消息的组合，并将其发送给`KVC`调用方。
    * 当`KVC`调用方与代理对象一起工作时，允许底层属性的行为如同`NSSet`一样，即使它不是`NSSet`。
* ④ 查看消息接收者类的`+accessInstanceVariablesDirectly`方法的返回值（默认返回`YES`）。如果返回`YES`，就按照`_<key>`、`_is<Key>`、`<key>`、`is<Key>`顺序查找成员变量。如果找到就直接取值并执行⑤，否则执行⑥。如果`+accessInstanceVariablesDirectly`方法返回`NO`也执行⑥。
* ⑤ 如果取到的值是一个对象指针，即获取的是对象，则直接将对象返回。
    * 如果取到的值是一个`NSNumber`支持的数据类型，则将其存储在`NSNumber`实例并返回。
    * 如果取到的值不是一个`NSNumber`支持的数据类型，则转换为`NSValue`对象, 然后返回。
* ⑥ 调用`valueForUndefinedKey:`方法，该方法抛出异常`NSUnknownKeyException`，并导致程序`Crash`。这是默认实现，我们可以重写该方法根据特定`key`做一些特殊处理。

### 基本的 Setter 搜索模式
以下是`setValue:forKey:`方法的默认实现，给定`key`和`value`作为输入参数，尝试将`KVC`调用方的属性名为`key`的值设置为`value`，执行以下过程。
* ① 按照`set<Key>:`、`_set<Key>:`顺序查找方法。
    <br>如果找到就调用并将`value`传进去（根据需要进行数据类型转换），否则执行②。
* ② 查看消息接收者类的`+accessInstanceVariablesDirectly`方法的返回值（默认返回`YES`）。如果返回`YES`，就按照`_<key>`、`_is<Key>`、`<key>`、`is<Key>`顺序查找成员变量（同 基本的 Getter 搜索模式）。如果找到就将`value`赋值给它（根据需要进行数据类型转换），否则执行③。如果`+accessInstanceVariablesDirectly`方法返回`NO`也执行③。
* ③ 调用`setValue:forUndefinedKey:`方法，该方法抛出异常`NSUnknownKeyException`，并导致程序`Crash`。这是默认实现，我们可以重写该方法根据特定`key`做一些特殊处理。


### NSMutableArray 搜索模式
以下是`mutableArrayValueForKey:`方法的默认实现，给定一个`key`作为输入参数，返回属性名为`key`的集合的代理对象（这里指`NSMutableArray`对象），在消息接收者类中操作，执行以下过程。

* ① 查找一对方法`insertObject:in<Key>AtIndex:`和`removeObjectFrom<Key>AtIndex:`
    <br>（相当于`NSMutableArray`的原始方法`insertObject:atIndex:`和`removeObjectAtIndex:`），
    <br>或者`insert<Key>:atIndexes:`和`remove<Key>AtIndexes:`
    <br>（相当于`NSMutableArray`的原始方法`insertObjects:atIndexes:`和`removeObjectsAtIndexes:`）。
    * 如果我们至少实现了一个`insertion`方法和一个`removal`方法，则返回一个代理对象，来响应发送给`NSMutableArray`的消息，通过发送`insertObject:in<Key>AtIndex:`、`removeObjectFrom<Key>AtIndex:`、`insert<Key>:atIndexes:`、`remove<Key>AtIndexes:`组合消息给`KVC`调用方。否则执行②。
    >该代理对象类型为`NSKeyValueFastMutableArray2`，继承链为`NSKeyValueFastMutableArray2`->`NSKeyValueFastMutableArray`->`NSKeyValueMutableArray`->`NSMutableArray`。
    * 如果我们也实现了一个可选的`replace object`方法，如`replaceObjectIn<Key>AtIndex:withObject:`或`replace<Key>AtIndexes:with<Key>:`，代理对象在适当的情况下也会使用它们，以获得最佳性能。
* ② 查找`set<Key>:`方法。
    <br>如果找到，就会向`KVC`调用方发送一个`set<Key>:`消息，来返回一个响应`NSMutableArray`消息的代理对象。否则执行③。
    >该代理对象类型为`NSKeyValueSlowMutableArray`，继承链为`NSKeyValueSlowMutableArray`->`NSKeyValueMutableArray`->`NSMutableArray`。
    
    >**注意：**
    >此步骤中描述的机制比上一步的效率低得多，因为它可能重复创建新的集合对象，而不是修改现有的集合对象。因此，在设计自己的键值编码兼容对象时，通常应该避免使用它。
    >
    >给代理对象发送`NSMutableArray`消息都会调用`set<Key>:`方法。即，对代理对象进行修改，都是调用`set<Key>:`来重新赋值，所以效率会低很多。

* ③ 查看消息接收者类的`+accessInstanceVariablesDirectly`方法的返回值（默认返回`YES`）。如果返回`YES`，就按照`_<key>`、`<key>`顺序查找成员变量。如果找到就返回一个代理对象，该代理对象将接收所有`NSMutableArray`消息，通常是`NSMutableArray`或其子类。否则执行④。如果`+accessInstanceVariablesDirectly`方法返回`NO`也执行④。
* ④ 返回一个可变的集合代理对象。当它接收到`NSMutableArray`消息时，发送一个`valueForUndefinedKey:`消息给`KVC`调用方，该方法抛出异常`NSUnknownKeyException`，并导致程序`Crash`。这是默认实现，我们可以重写该方法根据特定`key`做一些特殊处理。

### 其他搜索模式
除了以上三种，`KVC`还有`NSMutableSet`和`NSMutableOrderedSet`两种搜索模式，它们的搜索规则和`NSMutableArray`相同，只是搜索和调用的方法不同。具体可以查看`KVC`官方文档 [KVC - Accessor Search Patterns](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/SearchImplementation.html#//apple_ref/doc/uid/20000955-CJBBBFFA)。

## 9. 异常处理
* ① 根据`KVC`搜索规则，当没有搜索到对应的`key`或者`keyPath`相关方法或者变量时，会调用对应的异常方法`valueForUndefinedKey:`或`setValue:forUndefinedKey:`，这两个方法的默认实现是抛出异常`NSUnknownKeyException`，并导致程序`Crash`。我们可以重写这两个方法来处理异常。
```objc
- (nullable id)valueForUndefinedKey:(NSString *)key;
- (void)setValue:(nullable id)value forUndefinedKey:(NSString *)key;
```
* ② 当进行赋值如`setValue:forKey:`时，如果`key`的数据类型是非对象类型，则`value`就禁止传`nil`。否则会调用`setNilValueForKey:`方法，该方法的默认实现是抛出异常`NSInvalidArgumentException`，并导致程序`Crash`。我们可以重写这个方法来处理异常。
```objc
- (void)setNilValueForKey:(NSString *)key
{
    if ([key isEqualToString:@"hidden"]) {
        [self setValue:@(NO) forKey:@”hidden”];
    } else {
        [super setNilValueForKey:key];
    }
}
```
## 10. 相关面试题
#### Q：通过 KVC 修改属性会触发 KVO 吗？
会，通过`KVC`修改成员变量值也会触发`KVO`。
#### Q：通过 KVC 键值编码技术是否会破坏面向对象的编程方法，或者说违背面向对象的编程思想呢？
`valueForKey:`和`setValue:forKey:`这里面的`key`是没有任何限制的，当我们知道一个类或实例它内部的私有变量名称的情况下，我们在外界可以通过已知的`key`来对它的私有变量进行访问或者赋值的操作，从这个角度来讲`KVC`键值编码技术会违背面向对象的编程思想。

## 参考
[Key-Value Coding Programming Guide（苹果官方文档）](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding)

