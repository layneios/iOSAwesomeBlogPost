**原文链接：** [iOS/OS X内存管理(一)：基本概念与原理](http://www.jianshu.com/p/1928b54e1253)

**Keywords:** `iOS`，`内存管理`，`性能优化`

# iOS/OS X内存管理(一)：基本概念与原理

在Objective-C的内存管理中，其实就是`引用计数(reference count)`的管理。内存管理就是在程序需要时程序员分配一段内存空间，而当使用完之后将它释放。如果程序员对内存资源使用不当，有时不仅会造成内存资源浪费，甚至会导致程序crach。我们将会从引用计数和内存管理规则等基本概念开始，然后讲述有哪些内存管理方法，最后注意有哪些常见内存问题。

![memory management from apple document](http://upload-images.jianshu.io/upload_images/166109-6f9ffb5f0dad7d27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 基本概念

### 引用计数(Reference Count)

为了解释引用计数，我们做一个类比：员工在办公室使用灯的情景。

![引用Pro Multithreading and Memory Management for iOS and OS X的图](http://upload-images.jianshu.io/upload_images/166109-679cb7139c451968.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 当*第一个人*进入办公室时，他需要使用灯，于是开灯，引用计数为1
* 当另一个人进入办公室时，他也需要灯，引用计数为2；每当多一个人进入办公室时，引用计数加1
* 当有一个人离开办公室时，引用计数减1，当引用计数为0时，也就是*最后一个人*离开办公室时，他不再需要使用灯，关灯离开办公室。

### 内存管理规则

从上面员工在办公室使用灯的例子，我们对比一下*灯的动作*与*Objective-C对象的动作*有什么相似之处：

| 灯的动作 | Objective-C对象的动作 |
| :-- | :-- |
| 开灯 | 创建一个对象并获取它的**所有权(ownership)** |
| 使用灯 | 获取对象的所有权 |
| 不使用灯 | 放弃对象的所有权 |
| 关灯 | 释放对象 |

因为我们是通过引用计数来管理灯，那么我们也可以通过引用计数来管理使用Objective-C对象。

![引用Pro Multithreading and Memory Management for iOS and OS X的图](http://upload-images.jianshu.io/upload_images/166109-5b6b9fef51d9b80a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而Objective-C对象的动作对应有哪些方法以及这些方法对引用计数有什么影响？

| Objective-C对象的动作 | Objective-C对象的方法 |
| :-- | :-- |
| 1\. 创建一个对象并获取它的所有权 | alloc/new/copy/mutableCopy (RC = 1) |
| 2\. 获取对象的所有权 | retain (RC + 1) |
| 3\. 放弃对象的所有权 | release (RC - 1) |
| 4\. 释放对象 | dealloc (RC = 0 ，此时会调用该方法) |

当你`alloc`一个对象objc，此时RC=1；在某个地方你又`retain`这个对象objc，此时RC加1，也就是RC=2；由于调用`alloc/retain`一次，对应需要调用`release`一次来释放对象objc，所以你需要`release`对象objc两次，此时RC=0；而当RC=0时，系统会自动调用`dealloc`方法释放对象。

### Autorelease Pool

在开发中，我们常常都会使用到**局部变量**，局部变量一个特点就是当它超过作用域时，就会自动释放。而**autorelease pool**跟局部变量类似，当执行代码超过autorelease pool块时，所有放在autorelease pool的对象都会自动调用`release`。它的工作原理如下：

* 创建一个`NSAutoreleasePool`对象
* 在autorelease pool块的对象调用`autorelease`方法
* 释放`NSAutoreleasePool`对象

![引用Pro Multithreading and Memory Management for iOS and OS X的图](http://upload-images.jianshu.io/upload_images/166109-4f0ef3af1b674dfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**iOS 5/OS X Lion前的**(等下会介绍引入ARC的写法)实例代码如下：

```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

// put object into pool
id obj = [[NSObject alloc] init];
[obj autorelease];

[pool drain];

/* 超过autorelease pool作用域范围时，obj会自动调用release方法 */
```

由于放在autorelease pool的对象并不会马上释放，如果有大量图片数据放在这里的话，将会导致内存不足。

```
for (int i = 0; i < numberOfImages; i++)
{
      /*   处理图片，例如加载
       *   太多autoreleased objects存在
       *   由于NSAutoreleasePool对象没有被释放
       *   在某个时刻，会导致内存不足 
       */
}
```

# ARC管理方法

iOS/OS X内存管理方法有两种：手动引用计数(Manual Reference Counting)和自动引用计数(Automatic Reference Counting)。从OS X Lion和iOS 5开始，不再需要程序员手动调用`retain`和`release`方法来管理Objective-C对象的内存，而是引入一种新的内存管理机制*Automatic Reference Counting(ARC)*，简单来说，它让编译器来代替程序员来自动加入`retain`和`release`方法来持有和放弃对象的所有权。

在ARC内存管理机制中，`id`和其他对象类型变量必须是以下四个**ownership qualifiers**其中一个来修饰：

* __strong(默认，如果不指定其他，编译器就默认加入)
* __weak
* __unsafe_unretained
* __autoreleasing

所以在管理Objective-C对象内存的时候，你必须选择其中一个，下面会用一些列子来逐个解释它们的含义以及如何选择它们。

### __strong ownership qualifier

如果我想创建一个字符串，使用完之后将它释放调用，使用MRC管理内存的写法应该是这样：

```
{
    NSString *text = [[NSString alloc] initWithFormat:@"Hello, world"];   //@"Hello, world"对象的RC=1
    NSLog(@"%@", text);
    [text release];                      //@"Hello, world"对象的RC=0
}
```

而如果是使用ARC方式的话，就`text`对象无需调用`release`方法，而是当`text`变量超过作用域时，编译器来自动加入`[text release]`方法来释放内存

```
{
    NSString *text = [[NSString alloc] initWithFormat:@"Hello, world"];    //@"Hello, world"对象的RC=1
    NSLog(@"%@", text);
}
/*
 *  当text超过作用域时，@"Hello, world"对象会自动释放，RC=0
 */
```

而当你将`text`赋值给其他变量`anotherText`时，MRC需要`retain`一下来持有所有权，当`text`和`anotherText`使用完之后，各个调用`release`方法来释放。

```
{
    NSString *text = [[NSString alloc] initWithFormat:@"Hello, world"];    //@"Hello, world"对象的RC=1
    NSLog(@"%@", text);

    NSString *anotherText = text;        //@"Hello, world"对象的RC=1
    [anotherText retain];                //@"Hello, world"对象的RC=2
    NSLog(@"%@", anotherText);

    [text release];                      //@"Hello, world"对象的RC=1
    [anotherText release];               //@"Hello, world"对象的RC=0
}
```

而使用ARC的话，并不需要调用`retain`和`release`方法来持有跟释放对象。

```
{
    NSString *text = [[NSString alloc] initWithFormat:@"Hello, world"];   //@"Hello, world"对象的RC=1
    NSLog(@"%@", text);

    NSString *anotherText = text;        //@"Hello, world"对象的RC=2
    NSLog(@"%@", anotherText);
}
/*
 *  当text和anotherText超过作用域时，会自动调用[text release]和[anotherText release]方法， @"Hello, world"对象的RC=0
 */
```

除了当`__strong`变量超过作用域时，编译器会自动加入`release`语句来释放内存，如果你将`__strong`变量重新赋给它其他值，那么编译器也会自动加入`release`语句来释放变量指向之前的对象。例如：

```
{
    NSString *text = [[NSString alloc] initWithFormat:@"Hello, world"];    //@"Hello, world"对象的RC=1
    NSString *anotherText = text;        //@"Hello, world"对象的RC=2
    NSString *anotherText = [[NSString alloc] initWithFormat:@"Sam Lau"];  // 由于anotherText对象引用另一个对象@"Sam Lau"，那么就会自动调用[anotherText release]方法，使得@"Hello, world"对象的RC=1, @"Sam Lau"对象的RC=1
}
/*
 *  当text和anotherText超过作用域时，会自动调用[text release]和[anotherText release]方法，
 *  @"Hello, world"对象的RC=0和@"Sam Lau"对象的RC=0
 */
```

> 如果变量var被`__strong`修饰，当变量var指向某个对象objc，那么变量var持有某个对象objc的所有权

前面已经提过内存管理的**四条规则**：

| Objective-C对象的动作 | Objective-C对象的方法 |
| :-- | :-- |
| 1\. 创建一个对象并获取它的所有权 | alloc/new/copy/mutableCopy (RC = 1) |
| 2\. 获取对象的所有权 | retain (RC + 1) |
| 3\. 放弃对象的所有权 | release (RC - 1) |
| 4\. 释放对象 | dealloc (RC = 0 ，此时会调用该方法) |

我们总结一下编译器是按以下方法来实现的：

* 对于规则1和规则2，是通过`__strong`变量来实现，
* 对于规则3来说，当变量超过它的作用域或被赋值或成员变量被丢弃时就能实现
* 对于规则4，当RC=0时，系统就会自动调用

### __weak ownership qualifier

其实编译器根据`__strong`修饰符来管理对象内存。但是`__strong`并不能解决引用循环(Reference Cycle)问题：对象A持有对象B，反过来，对象B持有对象A；这样会导致不能释放内存造成[内存泄露](https://en.wikipedia.org/wiki/Memory_leak)问题。

![引用Pro Multithreading and Memory Management for iOS and OS X的图](http://upload-images.jianshu.io/upload_images/166109-bb0f3e53edb1308a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

举一个简单的例子，有一个类`Test`有个属性objc，有两个对象test1和test2的属性objc互相引用test1和test2：

```
@interface Test : NSObject

@property (strong, nonatomic) id objc;

@end
```

```
{
    Test *test1 = [Test new];        /* 对象a */
    /* test1有一个强引用到对象a */

    Test *test2 = [Test new];        /* 对象b */
    /* test2有一个强引用到对象b */

    test1.objc = test2;              /* 对象a的成员变量objc有一个强引用到对象b */
    test2.objc = test1;              /* 对象b的成员变量objc有一个强引用到对象a */
}
/*   当变量test1超过它作用域时，它指向a对象会自动release
 *   当变量test2超过它作用域时，它指向b对象会自动release
 *   
 *   此时，b对象的objc成员变量仍持有一个强引用到对象a
 *   此时，a对象的objc成员变量仍持有一个强引用到对象b
 *   于是发生内存泄露
 */
```

如何解决？于是我们引用一个`__weak`ownership qualifier，被它修饰的变量都不持有对象的所有权，而且当变量指向的对象的RC为0时，变量设置为nil。例如：

```
__weak NSString *text = [[NSString alloc] initWithFormat:@"Sam Lau"];
NSLog(@"%@", text);
```

由于text变量被`__weak`修饰，text并不持有`@"Sam Lau"`对象的所有权，`@"Sam Lau"`对象一创建就马上被释放，并且编译器给出警告⚠️，所以打印结果为`(null)`。

所以，针对刚才的引用循环问题，只需要将`Test`类的属性objc设置weak修饰符，那么就能解决。

```
@interface Test : NSObject

@property (weak, nonatomic) id objc;

@end
```

```
{
    Test *test1 = [Test new];        /* 对象a */
    /* test1有一个强引用到对象a */

    Test *test2 = [Test new];        /* 对象b */
    /* test2有一个强引用到对象b */

    test1.objc = test2;              /* 对象a的成员变量objc不持有对象b */
    test2.objc = test1;              /* 对象b的成员变量objc不持有对象a */
}
/*   当变量test1超过它作用域时，它指向a对象会自动release
 *   当变量test2超过它作用域时，它指向b对象会自动release
 */
```

### __unsafe_unretained ownership qualifier

`__unsafe_unretained` ownership qualifier，正如名字所示，它是**不安全**的。它跟`__weak`相似，被它修饰的变量都不持有对象的所有权，但当变量指向的对象的RC为0时，变量并不设置为nil，而是继续保存对象的地址；这样的话，对象有可能已经释放，但继续访问，就会造成**非法访问(Invalid Access)**。例子如下：

```
__unsafe_unretained id obj0 = nil;

{
    id obj1 = [[NSObject alloc] init];     // 对象A
    /* 由于obj1是强引用，所以obj1持有对象A的所有权，对象A的RC=1 */

    obj0 = obj1;
    /* 由于obj0是__unsafe_unretained，它不持有对象A的所有权，但能够引用它，对象A的RC=1 */

    NSLog(@"A: %@", obj0);
}
/* 当obj1超过它的作用域时，它指向的对象A将会自动释放 */

NSLog(@"B: %@", obj0);
/* 由于obj0是__unsafe_unretained，当它指向的对象RC=0时，它会继续保存对象的地址，所以两个地址相同 */
```

打印结果是**内存地址相同**：

![](http://upload-images.jianshu.io/upload_images/166109-8708e51bc5a5b117.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


如果将`__unsafe_unretained`改为`weak`的话，两个打印结果将不同

```
__weak id obj0 = nil;

{
    id obj1 = [[NSObject alloc] init];     // 对象A
    /* 由于obj1是强引用，所以obj1持有对象A的所有权，对象A的RC=1 */

    obj0 = obj1;
    /* 由于obj0是__unsafe_unretained，它不持有对象A的所有权，但能够引用它，对象A的RC=1 */

    NSLog(@"A: %@", obj0);
}
/* 当obj1超过它的作用域时，它指向的对象A将会自动释放 */

NSLog(@"B: %@", obj0);
/* 由于obj0是__weak， 当它指向的对象RC=0时，它会自动设置为nil，所以两个打印结果将不同*/
```

![](http://upload-images.jianshu.io/upload_images/166109-800c3458f2675a16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### __autoreleasing ownership qualifier

引入ARC之后，让我们看看autorelease pool有哪些变化。没有ARC之前的写法如下：

```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

// put object into pool
id obj = [[NSObject alloc] init];
[obj autorelease];

[pool drain];

/* 超过autorelease pool作用域范围时，obj会自动调用release方法 */
```

引入ARC之后，写法比之前更加简洁：

```
@autoreleasepool {
    id __autoreleasing obj = [[NSObject alloc] init];
}
```

相比之前的创建、使用和释放`NSAutoreleasePool`对象，现在你只需要将代码放在`@autoreleasepool`块即可。你也不需要调用`autorelease`方法了，只需要用`__autoreleasing`修饰变量即可。

![引用Pro Multithreading and Memory Management for iOS and OS X的图](http://upload-images.jianshu.io/upload_images/166109-afdd57710d0433e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是我们很少或基本上不使用autorelease pool。当我们使用XCode创建工程后，有一个app的入口文件`main.m`使用了它：

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

### Property(属性)

有了ARC之后，新的property modifier也被引入到Objective-C类的**property**，例如：

```
@property (strong, nonatomic) NSString *text;
```

下面有张表来展示property modifier与ownership qualifier的对应关系

| Property modifier | Ownership qualifier |
| :-- | :-- |
| strong | __strong |
| retain | __strong |
| copy | __strong |
| weak | __weak |
| assign | __unsafe_unretained |
| unsafe_unretained | __unsafe_unretained |

# 总结

要想掌握iOS/OS X的内存管理，首先要深入理解引用计数(Reference Count)这个概念以及内存管理的规则；在没引入ARC之前，我们都是通过`retain`和`release`方法来手动管理内存，但引入ARC之后，我们可以借助编译器来帮忙自动调用`retain`和`release`方法来简化内存管理和减低出错的可能性。虽然`__strong`修饰符能够执行大多数内存管理，但它不能解决引用循环(Reference Cycle)问题，于是又引入另一个修饰符`__weak`。被`__strong`修饰的变量都持有对象的所有权，而被`__weak`修饰的变量并不持有对象所有权。下篇我们介绍使用工具如何解决常见内存问题：**悬挂指针和内存泄露**。

# 参考资料

* [Pro Multithreading and Memory Management for iOS and OS X](http://book.douban.com/subject/10536953/)
* [Advanced Memory Management Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)
* [Memory Usage Performance Guidelines](https://developer.apple.com/library/mac/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html#//apple_ref/doc/uid/20001880-99100-TPXREF113)

