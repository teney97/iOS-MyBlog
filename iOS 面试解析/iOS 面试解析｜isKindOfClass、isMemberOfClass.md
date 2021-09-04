## iOS 面试解析｜isKindOfClass、isMemberOfClass

本期通过一个 demo 讲解 `isMemberOfClass:`、`isKindOfClass:` 两个方法的相关知识点。

**以下打印结果是什么？**（严谨点就添加个说明吧：Person 类继承于 NSObject 类）

```objectivec
BOOL res1 = [[NSObject class] isKindOfClass:[NSObject class]];
BOOL res2 = [[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res3 = [[Person class] isKindOfClass:[Person class]];
BOOL res4 = [[Person class] isMemberOfClass:[Person class]];

NSLog(@"%d, %d, %d, %d", res1, res2, res3, res4);
```

打印结果：1, 0, 0, 0

**解释：**

以下是 objc4-723 中 `isMemberOfClass:`、`isKindOfClass:` 方法以及 `object_getClass()` 函数的实现。

```objectivec
+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
```

emmm 整理的时候发现后面的版本又做了小优化，具体就不展开了，不过原理不变，以下是 824 版本的：

```objectivec
+ (BOOL)isMemberOfClass:(Class)cls {
    return self->ISA() == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = self->ISA(); tcls; tcls = tcls->getSuperclass()) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->getSuperclass()) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

由此我们可以得出结论：

* `isMemberOfClass:` 方法是判断当前 `instance/class` 对象的 `isa` 指向是不是 `class/meta-class` 对象类型；
* `isKindOfClass:` 方法是判断当前 `instance/class` 对象的 `isa` 指向是不是 `class/meta-class` 对象或者它的子类类型。

显然 `isKindOfClass:` 的范围更大。如果方法调用者是 instance 对象，传参就应该是 class 对象。如果方法调用着是 class 对象，传参就应该是 meta-class 对象。所以 res2-res4 都为 0。那为什么 res1 为 1 呢？

因为 NSObject 的 class 的对象的 isa 指向它的 meta-class 对象，而它的 meta-class 的 superclass 指向它的 class 对象，所以 `[[NSObject class] isKindOfClass:[NSObject class]]` 成立 。

![](https://gitee.com/zhangferry/Images/raw/master/iOSWeeklyLearning/objc-isa-class-diagram.jpg)

总之，`[instance/class isKindOfClass:[NSObject class]]` 恒成立。（严谨点，需要是 NSObject 及其子类类型）