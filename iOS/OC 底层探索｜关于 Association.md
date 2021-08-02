![网络配图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/82d72385f68a4d6da672726bcc3edb0b~tplv-k3u1fbpfcp-zoom-1.image)

## 1. 关联对象
### 1.1 使用场景
默认情况下，由于分类底层结构的限制，不能直接给 Category 添加成员变量，但是可以通过关联对象间接实现 Category 有成员变量的效果。
<br>[传送门：OC - Category 和 Extension](https://www.jianshu.com/p/1eca0353ceab)


### 1.2 使用方法
```objc
#import "Person.h"
@interface Person (Test)
@property (nonatomic, assign) int height;
@end

#import "Person+Test.h"
#import <objc/runtime.h>
@implementation Person (Test)
- (void)setHeight:(int)height
{
    objc_setAssociatedObject(self, @selector(height), [NSNumber numberWithInt:height], OBJC_ASSOCIATION_ASSIGN);
}
- (int)height
{
    return [objc_getAssociatedObject(self, @selector(height)) intValue];
}
@end
```

#### 1.2.1 相关 API

**objc_setAssociatedObject**

使用给定的`key`和关联策略为给定的对象设置关联的`value`。
```objc
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
```

**objc_getAssociatedObject**

返回给定`key`的给定对象关联的`value`。
```objc
id objc_getAssociatedObject(id object, const void *key);
```

**objc_removeAssociatedObjects**

删除给定对象的所有关联。 
```objc
void objc_removeAssociatedObjects(id object);
```
如果只想移除给定对象的某个`key`的关联，可以使用`objc_setAssociatedObject`给参数`value`传值`nil`。
```objc
objc_setAssociatedObject(self, @selector(height), nil, OBJC_ASSOCIATION_ASSIGN);
```


#### 1.2.2 objc_AssociationPolicy 关联策略
```objc
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```
objc_AssociationPolicy|对应的属性修饰符
:--:|:--:
OBJC_ASSOCIATION_ASSIGN|assign
OBJC_ASSOCIATION_RETAIN_NONATOMIC|strong, nonatomic
OBJC_ASSOCIATION_COPY_NONATOMIC|copy, nonatomic
OBJC_ASSOCIATION_RETAIN|strong, atomic
OBJC_ASSOCIATION_COPY|copy, atomic



#### 1.2.3 key 的常见用法
```objc
// ①
static const void *MyKey = &MyKey;
objc_setAssociatedObject(object, MyKey, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(object, MyKey];

// ② 
static const char MyKey;
objc_setAssociatedObject(object, &MyKey, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(object, &MyKey];

// ③ 使用属性名作为 key
#define MYHEIGHT @"height"
objc_setAssociatedObject(object, MYHEIGHT, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(object, MYHEIGHT];

// ④ 使用 getter 方法的 SEL 作为 key（可读性高，有智能提示）
objc_setAssociatedObject(object, @selector(getter), value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(object, @selector(getter)];
// 或使用隐式参数 _cmd
objc_getAssociatedObject(object, _cmd];
```

## 2. 关联对象的原理
在`Runtime`源码`objc4`中，有关关联对象的代码都在文件`objc-references.mm`中。

**实现关联对象技术的核心对象**
* AssociationsManager
* AssociationsHashMap
* ObjectAssociationMap
* ObjcAssociation
```objc
class AssociationsManager {
    static AssociationsHashMap *_map;
};
class AssociationsHashMap : public unordered_map<disguised_ptr_t, ObjectAssociationMap>
class ObjectAssociationMap : public std::map<void *, ObjcAssociation>
class ObjcAssociation {
    uintptr_t _policy;
    id _value;
};
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c4a14c2a52a40feb4b544422860ba12~tplv-k3u1fbpfcp-zoom-1.image)


### 数据结构

**AssociationsManager**
* 关联对象并不是存储在关联对象本身内存中，而是存储在全局统一的一个容器中；
* 由 AssociationsManager 管理并在它维护的一个单例 Hash 表 AssociationsHashMap 中存储；
* 使用 AssociationsManagerLock 自旋锁保证了线程安全。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a77399024c12426aacd33156159c91ed~tplv-k3u1fbpfcp-zoom-1.image)


**AssociationsHashMap**
* 一个单例的 Hash 表，存储 disguised_ptr_t 和 ObjectAssociationMap 之间的映射。
* disguised_ptr_t 是根据 object 生成，但不存在引用关系。
<br>`disguised_ptr_t disguised_object = DISGUISE(object);`


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5a06289571644d2a6f8dd78a269b4ae~tplv-k3u1fbpfcp-zoom-1.image)


**ObjectAssociationMap**
* 存储 key 和 ObjcAssociation 之间的映射。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fc8f0f53bec43a2b6d2ffe511acaa03~tplv-k3u1fbpfcp-zoom-1.image)

**ObjcAssociation**
* 存储着关联策略 policy 和关联对象的值 value。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae58beb55c974f97aee586660e8f6f57~tplv-k3u1fbpfcp-zoom-1.image)

### objc_setAssociatedObject
* ① 实例化一个  AssociationsManager 对象，它维护了一个单例 Hash 表 AssociationsHashMap 对象；
* ② 实例化一个 AssociationsHashMap 对象，它维护 disguised_ptr_t 和 ObjectAssociationMap 对象之间的关系；
* ③ 根据`object`生成一个 disguised_ptr_t 对象；
* ④ 根据 disguised_ptr_t 获取对应的 ObjectAssociationMap 对象，它存储`key`和 ObjcAssociation 之间的映射；
* ⑤ 根据`policy`和`value`创建一个 ObjcAssociation 对象，并存储在 ObjectAssociationMap 中；
* ⑥ 如果传进来的`value`为 nil，则在 ObjectAssociationMap 中删除该 ObjcAssociation 对象。






![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a62f0f1fea3242d8af7679f936ea5a40~tplv-k3u1fbpfcp-zoom-1.image)
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1dcc378517d48b88a122266a83dd9b3~tplv-k3u1fbpfcp-zoom-1.image)


### objc_getAssociatedObject
* ① 实例化一个  AssociationsManager 对象；
* ② 实例化一个 AssociationsHashMap 对象；
* ③ 根据`object`生成一个 disguised_ptr_t 对象；
* ④ 根据 disguised_ptr_t 获取对应的 ObjectAssociationMap 对象；
* ⑤ 根据 key 获取到它所对应的 ObjcAssociation 对象；
* ⑥ 返回 ObjcAssociation 中的 value。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/716e3b01f38949c989ce5cf392bc2ba6~tplv-k3u1fbpfcp-zoom-1.image)


### objc_removeAssociatedObjects
* ① 实例化一个  AssociationsManager 对象；
* ② 实例化一个 AssociationsHashMap 对象；
* ③ 根据`object`生成一个 disguised_ptr_t 对象；
* ④ 根据 disguised_ptr_t 获取对应的 ObjectAssociationMap 对象；
* ⑤ 删除该 ObjectAssociationMap 对象。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7556573b4674ed1a12da4771639a967~tplv-k3u1fbpfcp-zoom-1.image)

### acquireValue

根据`policy`来对`value`进行`retain`或者`copy`操作。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66aa88b9b94844bb93cf7252c43b81fa~tplv-k3u1fbpfcp-zoom-1.image)

## 3. 相关面试题

#### Q：如何移除关联对象？
* 移除一个`object`的某个`key`的关联对象：调用`objc_setAssociatedObject`设置关联对象`value`为`nil`。<br>`objc_setAssociatedObject`函数会调用`_object_set_associative_reference`函数，并在该函数中判断传进来的`value`是否为`nil`，是的话会调用`erase(j)`擦除函数，将`j`变量擦除。`j`即为`ObjectAssociationMap`对象里的一对【key: key&emsp;value: ObjcAssociation（_policy、_value）】。
* 移除一个`object`的所有关联对象：调用函数`objc_removeAssociatedObjects`。<br>`objc_removeAssociatedObjects`函数会调用`_object_remove_assocations`函数，并在该函数中调用对象的`erase(i)`擦除函数，将`i`变量擦除。`i`即为`AssociationsHashMap`对象中的一对【key: object&emsp;value: ObjectAssociationMap】。

#### Q：如果 object 被销毁，那它所对应的 ObjectAssociationMap 是否也会自动销毁？
会。

#### Q：如果没有关联对象，怎么实现 Category 有成员变量的效果？
使用字典。创建一个全局的字典，将`self`对象在内存中的地址作为`key`。<br>
缺点：① 内存泄漏问题：全局变量会一直存储在内存中；<br>
&emsp;&emsp;&emsp;② 线程安全问题：可能会有多个对象同时访问字典，加锁可以解决；<br>
&emsp;&emsp;&emsp;③ 每添加一个成员变量就要创建一个字典，很麻烦。

```objc
#import "Person.h"
@interface Person (Test)
@property (nonatomic, assign) int height;
@end

#import "Person+Test.h"
#import <objc/runtime.h>
@implementation Person (Test)
NSMutableDictionary *heights_;
+ (void)load {
    heights_ = [NSMutableDictionary dictionary];
}
- (void)setHeight:(int)height {
    NSString *key = [NSString stringWithFormat:@"%@",self];
    heights_[key] = @(height);
}
- (int)height {
    NSString *key = [NSString stringWithFormat:@"%@",self];
    return [heights_[key] intValue];
}
```
