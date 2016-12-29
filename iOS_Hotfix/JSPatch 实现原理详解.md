**原文链接：** [JSPatch 实现原理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3)

**Keywords:** `JSPatch`，`iOS`，`动态化`

# JSPatch 实现原理详解

JSPatch 是一个 iOS 动态更新框架，只需在项目中引入极小的引擎，就可以使用 JavaScript 调用任何 Objective-C 原生接口，获得脚本语言的优势：为项目动态添加模块，或替换项目原生代码动态修复 bug。

之前在博客上写过两篇 JSPatch 原理解析文章([1](http://blog.cnbang.net/tech/2808/) [2](http://blog.cnbang.net/tech/2855/))，但随着 JSPatch 的改进，有些内容已经跟最新代码对不上，这里重新整理成一篇完整的文章，对原来的两篇做整合和修改，详细阐述 JSPatch 的实现原理和一些细节，以帮助使用者更好地了解和使用 JSPatch。

## 大纲

```
基础原理

方法调用
  1.require
  2.JS接口
    i.封装 JS 对象
    ii.`__c()`元函数
  3.消息传递
  4.对象持有/转换
  5.类型转换

方法替换
  1.基础原理
  2.va_list实现(32位)
  3.ForwardInvocation实现(64位)
  4.新增方法
    i.方案
    ii.Protocol
  5.Property实现
  6.self关键字
  7.super关键字
扩展
  1.Struct 支持
  2.C 函数支持

细节
  1.Special Struct
  2.内存问题
    i.Double Release
    ii.内存泄露
  3.‘_’的处理
  4.JPBoxing
  5.nil 的处理
    i.区分NSNull/nil
    ii.链式调用

总结

```

## 基础原理

JSPatch 能做到通过 JS 调用和改写 OC 方法最根本的原因是 Objective-C 是动态语言，OC 上所有方法的调用/类的生成都通过 Objective-C Runtime 在运行时进行，我们可以通过类名/方法名反射得到相应的类和方法：

```source-objc
Class class = NSClassFromString("UIViewController");
id viewController = [[class alloc] init];
SEL selector = NSSelectorFromString("viewDidLoad");
[viewController performSelector:selector];
```

也可以替换某个类的方法为新的实现：

```source-objc
static void newViewDidLoad(id slf, SEL sel) {}
class_replaceMethod(class, selector, newViewDidLoad, @"");
```

还可以新注册一个类，为类添加方法：

```source-objc
Class cls = objc_allocateClassPair(superCls, "JPObject", 0);
objc_registerClassPair(cls);
class_addMethod(cls, selector, implement, typedesc);
```

对于 Objective-C 对象模型和动态消息发送的原理已有很多文章阐述得很详细，这里就不详细阐述了。理论上你可以在运行时通过类名/方法名调用到任何 OC 方法，替换任何类的实现以及新增任意类。所以 JSPatch 的基本原理就是：JS 传递字符串给 OC，OC 通过 Runtime 接口调用和替换 OC 方法。这是最基础的原理，实际实现过程还有很多怪要打，接下来看看具体是怎样实现的。

## 方法调用

```source-js
require('UIView')
var view = UIView.alloc().init()
view.setBackgroundColor(require('UIColor').grayColor())
view.setAlpha(0.5)
```

引入 JSPatch 后，可以通过以上 JS 代码创建了一个 UIView 实例，并设置背景颜色和透明度，涵盖了 require 引入类，JS 调用接口，消息传递，对象持有和转换，参数转换这五个方面，接下来逐一看看具体实现。

### 1.require

调用 `require('UIView')` 后，就可以直接使用 `UIView` 这个变量去调用相应的类方法了，require 做的事很简单，就是在JS全局作用域上创建一个同名变量，变量指向一个对象，对象属性 `__clsName` 保存类名，同时表明这个对象是一个 OC Class。

```source-js
var _require = function(clsName) {
  if (!global[clsName]) {
    global[clsName] = {
      __clsName: clsName
    }
  }
  return global[clsName]
}
```

所以调用 `require('UIView')` 后，就在全局作用域生成了 `UIView` 这个变量，指向一个这样一个对象：

```source-js
{
  __clsName: "UIView"
}
```

### 2.JS接口

接下来看看 `UIView.alloc()` 是怎样调用的。

#### i.封装 JS 对象

对于这个调用的实现，一开始我的想法是，根据JS特性，若要让 `UIView.alloc()` 这句调用不出错，唯一的方法就是给 `UIView` 这个对象添加 `alloc` 方法，不然是不可能调用成功的，JS 对于调用没定义的属性/变量，只会马上抛出异常，而不像 OC/Lua/ruby 那样会有转发机制。所以做了一个复杂的事，就是在require生成类对象时，把类名传入OC，OC 通过runtime方法找出这个类所有的方法返回给 JS，JS 类对象为每个方法名都生成一个函数，函数内容就是拿着方法名去 OC 调用相应方法。生成的 `UIView` 对象大致是这样的：

```source-js
{
    __clsName: "UIView",
    alloc: function() {…},
    beginAnimations_context: function() {…},
    setAnimationsEnabled: function(){…},
    ...
}
```

实际上不仅要遍历当前类的所有方法，还要循环找父类的方法直到顶层，整个继承链上的所有方法都要加到JS对象上，一个类就有几百个方法，这样把方法全部加到 JS 对象上，碰到了挺严重的问题，引入几个类就内存暴涨，无法使用。后来为了优化内存问题还在 JS 搞了继承关系，不把继承链上所有方法都添加到一个JS对象，避免像基类 NSObject 的几百个方法反复添加在每个 JS 对象上，每个方法只存在一份，JS 对象复制了 OC 对象的继承关系，找方法时沿着继承链往上找，结果内存消耗是小了一些，但还是大到难以接受。

#### ii.`__c()`元函数

当时继续苦苦寻找解决方案，若按 JS 语法，这是唯一的方法，但若不按 JS 语法呢？突然脑洞开了下，CoffieScript/JSX 都可以用 JS 实现一个解释器实现自己的语法，我也可以通过类似的方式做到，再进一步想到其实我想要的效果很简单，就是调用一个不存在方法时，能转发到一个指定函数去执行，就能解决一切问题了，这其实可以用简单的字符串替换，把 JS 脚本里的方法调用都替换掉。最后的解决方案是，在 OC 执行 JS 脚本前，通过正则把所有方法调用都改成调用 `__c()` 函数，再执行这个 JS 脚本，做到了类似 OC/Lua/Ruby 等的消息转发机制：

```source-js
UIView.alloc().init()
->
UIView.__c('alloc')().__c('init')()
```

给 JS 对象基类 Object 加上 `__c` 成员，这样所有对象都可以调用到 `__c`，根据当前对象类型判断进行不同操作：

```source-js
Object.defineProperty(Object.prototype, '__c', {value: function(methodName) {
  if (!this.__obj && !this.__clsName) return this[methodName].bind(this);
  var self = this
  return function(){
    var args = Array.prototype.slice.call(arguments)
    return _methodFunc(self.__obj, self.__clsName, methodName, args, self.__isSuper)
  }
}})
```

`_methodFunc()` 就是把相关信息传给OC，OC用 Runtime 接口调用相应方法，返回结果值，这个调用就结束了。

这样做不用去 OC 遍历对象方法，不用在 JS 对象保存这些方法，内存消耗直降 99%，这一步是做这个项目最爽的时候，用一个非常简单的方法解决了严重的问题，替换之前又复杂效果又差的实现。

### 3.消息传递

解决了 JS 接口问题，接下来看看 JS 和 OC 是怎样互传消息的。这里用到了 JavaScriptCore 的接口，OC 端在启动 JSPatch 引擎时会创建一个 `JSContext` 实例，`JSContext` 是 JS 代码的执行环境，可以给 `JSContext` 添加方法，JS 就可以直接调用这个方法：

```
JSContext *context = [[JSContext alloc] init];
context[@"hello"] = ^(NSString *msg) {
    NSLog(@"hello %@", msg);
};
[_context evaluateScript:@"hello('word')"];     //output hello word

```

JS 通过调用 `JSContext` 定义的方法把数据传给 OC，OC 通过返回值传会给 JS。调用这种方法，它的参数/返回值 JavaScriptCore 都会自动转换，OC 里的 NSArray, NSDictionary, NSString, NSNumber, NSBlock 会分别转为JS端的数组/对象/字符串/数字/函数类型。上述 `_methodFunc` 方法就是这样把要调用的类名和方法名传递给 OC 的。

### 4.对象持有/转换

结合上述几点，可以知道 `UIView.alloc()` 这个类方法调用语句是怎样执行的：

a.`require('UIView')` 这句话在 JS 全局作用域生成了 `UIView` 这个对象，它有个属性叫 `__isCls`，表示这代表一个 OC 类。 b.调用 `UIView` 这个对象的 `alloc()` 方法，会去到 `__c()`函数，在这个函数里判断到调用者 `__isCls` 属性，知道它是代表 OC 类，把方法名和类名传递给 OC 完成调用。

调用类方法过程是这样，那实例方法呢？`UIView.alloc()` 会返回一个 UIView 实例对象给 JS，这个 OC 实例对象在 JS 是怎样表示的？怎样可以在 JS 拿到这个实例对象后可以直接调用它的实例方法 `UIView.alloc().init()`？

对于一个自定义id对象，JavaScriptCore 会把这个自定义对象的指针传给 JS，这个对象在 JS 无法使用，但在回传给 OC 时 OC 可以找到这个对象。对于这个对象生命周期的管理，按我的理解如果JS有变量引用时，这个 OC 对象引用计数就加1 ，JS 变量的引用释放了就减1，如果 OC 上没别的持有者，这个OC对象的生命周期就跟着 JS 走了，会在 JS 进行垃圾回收时释放。

传回给 JS 的变量是这个 OC 对象的指针，这个指针也可以重新传回 OC，要在 JS 调用这个对象的某个实例方法，根据第2点 JS 接口的描述，只需在 `__c()` 函数里把这个对象指针以及它要调用的方法名传回给 OC 就行了，现在问题只剩下：怎样在 `__c()` 函数里判断调用者是一个 OC 对象指针？

目前没找到方法判断一个 JS 对象是否表示 OC 指针，这里的解决方法是在 OC 把对象返回给 JS 之前，先把它包装成一个 NSDictionary：

```source-objc
static NSDictionary *_wrapObj(id obj) {
    return @{@"__obj": obj};
}
```

让 OC 对象作为这个 NSDictionary 的一个值，这样在 JS 里这个对象就变成：

```source-objc
{__obj: [OC Object 对象指针]}
```

这样就可以通过判断对象是否有 `__obj` 属性得知这个对象是否表示 OC 对象指针，在 `__c` 函数里若判断到调用者有 `__obj` 属性，取出这个属性，跟调用的实例方法一起传回给 OC，就完成了实例方法的调用。

### 5.类型转换

JS 把要调用的类名/方法名/对象传给 OC 后，OC 调用类/对象相应的方法是通过 NSInvocation 实现，要能顺利调用到方法并取得返回值，要做两件事：

a.取得要调用的 OC 方法各参数类型，把 JS 传来的对象转为要求的类型进行调用。 b.根据返回值类型取出返回值，包装为对象传回给 JS。

例如开头例子的 `view.setAlpha(0.5)`， JS传递给OC的是一个 NSNumber，OC 需要通过要调用 OC 方法的 `NSMethodSignature` 得知这里参数要的是一个 float 类型值，于是把 NSNumber 转为 float 值再作为参数进行 OC 方法调用。这里主要处理了 int/float/bool 等数值类型，并对 CGRect/CGRange 等类型进行了特殊转换处理，剩下的就是实现细节了。

## 方法替换

JSPatch 可以用 `defineClass` 接口任意替换一个类的方法，方法替换的实现过程也是颇为曲折，一开始是用 `va_list` 的方式获取参数，结果发现 arm64 下不可用，只能转而用另一种 hack 方式绕道实现。另外在给类新增方法、实现property、支持self/super关键字上也费了些功夫，下面逐个说明。

### 1.基础原理

OC上，每个类都是这样一个结构体：

```source-objc
struct objc_class {
  struct objc_class * isa;
  const char *name;
  ….
  struct objc_method_list **methodLists; /*方法链表*/
};
```

其中 methodList 方法链表里存储的是 Method 类型：

```source-objc
typedef struct objc_method *Method;
typedef struct objc_ method {
  SEL method_name;
  char *method_types;
  IMP method_imp;
};
```

Method 保存了一个方法的全部信息，包括 SEL 方法名，type 各参数和返回值类型，IMP 该方法具体实现的函数指针。

通过 Selector 调用方法时，会从 methodList 链表里找到对应Method进行调用，这个 methodList 上的的元素是可以动态替换的，可以把某个 Selector 对应的函数指针IMP替换成新的，也可以拿到已有的某个 Selector 对应的函数指针IMP，让另一个 Selector 跟它对应，Runtime 提供了一些接口做这些事，以替换 UIViewController 的 `-viewDidLoad:` 方法为例：

```source-objc
static void viewDidLoadIMP (id slf, SEL sel) {
   JSValue *jsFunction = …;
   [jsFunction callWithArguments:nil];
}

Class cls = NSClassFromString(@"UIViewController");
SEL selector = @selector(viewDidLoad);
Method method = class_getInstanceMethod(cls, selector);

//获得viewDidLoad方法的函数指针
IMP imp = method_getImplementation(method)

//获得viewDidLoad方法的参数类型
char *typeDescription = (char *)method_getTypeEncoding(method);

//新增一个ORIGViewDidLoad方法，指向原来的viewDidLoad实现
class_addMethod(cls, @selector(ORIGViewDidLoad), imp, typeDescription);

//把viewDidLoad IMP指向自定义新的实现
class_replaceMethod(cls, selector, viewDidLoadIMP, typeDescription);
```

这样就把 UIViewController 的 `-viewDidLoad` 方法给替换成我们自定义的方法，APP里调用 UIViewController 的 `viewDidLoad` 方法都会去到上述 viewDidLoadIMP 函数里，在这个新的IMP函数里调用 JS 传进来的方法，就实现了替换 viewDidLoad 方法为JS代码里的实现，同时为 UIViewController 新增了个方法 `-ORIGViewDidLoad` 指向原来 viewDidLoad 的 IMP，JS 可以通过这个方法调用到原来的实现。

方法替换就这样很简单的实现了，但这么简单的前提是，这个方法没有参数。如果这个方法有参数，怎样把参数值传给我们新的 IMP 函数呢？例如 UIViewController 的 `-viewDidAppear:` 方法，调用者会传一个 Bool 值，我们需要在自己实现的IMP（上述的 viewDidLoadIMP）上拿到这个值，怎样能拿到？如果只是针对一个方法写 IMP，是可以直接拿到这个参数值的：

```source-objc
static void viewDidAppear (id slf, SEL sel, BOOL animated) {
   [function callWithArguments:@(animated)];
}
```

但我们要的是实现一个通用的IMP，任意方法任意参数都可以通过这个IMP中转，拿到方法的所有参数回调JS的实现。

### 2.va_list实现(32位)

最初我是用可变参数 `va_list` 实现：

```source-objc
static void commonIMP(id slf, ...)
  va_list args;
  va_start(args, slf);
  NSMutableArray *list = [[NSMutableArray alloc] init];
  NSMethodSignature *methodSignature = [cls instanceMethodSignatureForSelector:selector];
  NSUInteger numberOfArguments = methodSignature.numberOfArguments;
  id obj;
  for (NSUInteger i = 2; i < numberOfArguments; i++) {
      const char *argumentType = [methodSignature getArgumentTypeAtIndex:i];
      switch(argumentType[0]) {
          case 'i':
              obj = @(va_arg(args, int));
              break;
          case 'B':
              obj = @(va_arg(args, BOOL));
              break;
          case 'f':
          case 'd':
              obj = @(va_arg(args, double));
              break;
          …… //其他数值类型
          default: {
              obj = va_arg(args, id);
              break;
          }
      }
      [list addObject:obj];
  }
  va_end(args);
  [function callWithArguments:list];
}
```

这样无论方法参数是什么，有多少个，都可以通过 `va_list`的一组方法一个个取出来，组成 NSArray 在调用 JS 方法时传回。很完美地解决了参数的问题，一直运行正常，直到我跑在 arm64 的机子上测试，一调用就 crash。查了资料，才发现 arm64 下 `va_list` 的结构改变了，导致无法上述这样取参数。详见[这篇文章](https://blog.nelhage.com/2010/10/amd64-and-va_arg)。

### 3.ForwardInvocation实现(64位)

继续寻找解决方案，最后找到另一种非常 hack 的方法解决参数获取的问题，利用了 OC 消息转发机制。

当调用一个 NSObject 对象不存在的方法时，并不会马上抛出异常，而是会经过多层转发，层层调用对象的 `-resolveInstanceMethod:`, `-forwardingTargetForSelector:`, `-methodSignatureForSelector:`, `-forwardInvocation:` 等方法，其中最后 `-forwardInvocation:` 是会有一个 NSInvocation 对象，这个 NSInvocation 对象保存了这个方法调用的所有信息，包括 Selector 名，参数和返回值类型，最重要的是有所有参数值，可以从这个 NSInvocation 对象里拿到调用的所有参数值。我们可以想办法让每个需要被 JS 替换的方法调用最后都调到 `-forwardInvocation:`，就可以解决无法拿到参数值的问题了。

具体实现，以替换 UIViewController 的 -viewWillAppear: 方法为例：

1. 把UIViewController的 `-viewWillAppear:` 方法通过 `class_replaceMethod()` 接口指向 `_objc_msgForward`，这是一个全局 IMP，OC 调用方法不存在时都会转发到这个 IMP 上，这里直接把方法替换成这个 IMP，这样调用这个方法时就会走到 `-forwardInvocation:`。

2. 为UIViewController添加 `-ORIGviewWillAppear:` 和 `-_JPviewWillAppear:` 两个方法，前者指向原来的IMP实现，后者是新的实现，稍后会在这个实现里回调JS函数。

3. 改写UIViewController的 `-forwardInvocation:` 方法为自定义实现。一旦OC里调用 UIViewController 的 `-viewWillAppear:` 方法，经过上面的处理会把这个调用转发到 `-forwardInvocation:` ，这时已经组装好了一个 NSInvocation，包含了这个调用的参数。在这里把参数从 NSInvocation 反解出来，带着参数调用上述新增加的方法 `-_JPviewWillAppear:`，在这个新方法里取到参数传给JS，调用JS的实现函数。整个调用过程就结束了，整个过程图示如下：

![JSPatch方法替换](https://camo.githubusercontent.com/48cbbd8ee1c8af0ef8f18a2e0ab0d50a085afab1/687474703a2f2f626c6f672e636e62616e672e6e65742f77702d636f6e74656e742f75706c6f6164732f323031352f30362f4a535061746368322e706e67)

最后一个问题，我们把 UIViewController 的 `-forwardInvocation:` 方法的实现给替换掉了，如果程序里真有用到这个方法对消息进行转发，原来的逻辑怎么办？首先我们在替换 `-forwardInvocation:` 方法前会新建一个方法 `-ORIGforwardInvocation:`，保存原来的实现IMP，在新的 `-forwardInvocation:` 实现里做了个判断，如果转发的方法是我们想改写的，就走我们的逻辑，若不是，就调 `-ORIGforwardInvocation:` 走原来的流程。

其他就是实现上的细节了，例如需要根据不同的返回值类型生成不同的 IMP，要在各处处理参数转换等。

### 4.新增方法

#### i.方案

在 JSPatch 刚开源时，是不支持为一个类新增方法的，因为觉得能替换原生方法就够了，新的方法纯粹添加在 JS 对象上，只在 JS 端跑就行了。另外 OC 为类新增方法需要知道各个参数和返回值的类型，需要在 JS 定一种方式把这些类型传给 OC 才能完成新增方法，比较麻烦。后来挺多人比较关注这个问题，不能新增方法导致 action-target 模式无法用，我也开始想有没有更好的方法实现添加方法。后来的解决方案是所有类型都用 id 表示，因为反正新增的方法都是 JS 在用(Protocol定义的方法除外)，不如新增的方法返回值和参数全统一成 id 类型，这样就不用传类型了。

现在 `defineClass` 定义的方法会经过 JS 包装，变成一个包含参数个数和方法实体的数组传给OC，OC会判断如果方法已存在，就执行替换的操作，若不存在，就调用 `class_addMethod()` 新增一个方法，通过传过来的参数个数和方法实体生成新的 Method，把 Method 的参数和返回值类型都设为id。这里 JS 调用新增方法走的流程还是 `forwardInvocation` 这一套。

#### ii.Protocol

对于新增的方法还有个问题，若某个类实现了某 protocol，protocol方法里有可选的方法，它的参数不全是 id 类型，例如 `UITableViewDataSource` 的一个方法：

```source-objc
- (NSInteger)tableView:(UITableView *)tableView sectionForSectionIndexTitle:(NSString *)title atIndex:(NSInteger)index;
```

若原类没有实现这个方法，在 JS 里实现了，会走到新增方法的逻辑，每个参数类型都变成 id，与这个 protocol 方法不匹配，产生错误。

这里就需要在 JS 定义类时给出实现的 protocol，这样在新增 Protocol 里已定义的方法时，参数类型会按照 Protocol 里的定义去实现，Protocol 的定义方式跟 OC 上的写法一致：

```source-js
defineClass("JPViewController: UIViewController <UIAlertViewDelegate>", {
  alertView_clickedButtonAtIndex: function(alertView, buttonIndex) {
    console.log('clicked index ' + buttonIndex)
  }
})
```

实现方式比较简单，先把 Protocol 名解析出来，当 JS 定义的方法在原有类上找不到时，再通过 `objc_getProtocol` 和 `protocol_copyMethodDescriptionList` runtime 接口把 Protocol 对应的方法取出来，若匹配上，则按其方法的定义走方法替换的流程。

### 5.Property实现

若要在 JS 操作 OC 对象上已定义好的 property，只需要像调用普通 OC 方法一样，调用这个对象的 get/set 方法就行了：

```source-objc
//OC
@property (nonatomic) NSString *data;
@property (nonatomic) BOOL *succ;
```

```
//JS
self.setSucc(1);
var str = self.data();

```

若要动态给 OC 对象新增 property，则要另辟蹊径：

```source-js
defineClass('JPTableViewController : UITableViewController', {
  dataSource: function() {
    var data = self.getProp('data')
    if (data) return data;
    data = [1,2,3]
    self.setProp_forKey(data, 'data')
    return data;
  }
}
```

JSPatch可以通过 `-getProp:`， `-setProp:forKey:` 这两个方法给对象动态添加成员变量。实现上用了运行时关联接口 `objc_getAssociatedObject()` 和 `objc_setAssociatedObject()` 模拟，相当于把一个对象跟当前对象self关联起来，以后可以通过当前对象self找到这个对象，跟成员的效果一样，只是一定得是id对象类型。

本来OC有 `class_addIvar()` 可以为类添加成员，但必须在类注册之前添加完，注册完成后无法添加，这意味着可以为在JS新增的类添加成员，但不能为OC上已存在的类添加，所以只能用上述方法模拟。

### 6.self关键字

```source-js
defineClass("JPViewController: UIViewController", {
  viewDidLoad: function() {
    var view = self.view()
    ...
  },
}
```

JSPatch支持直接在defineClass里的实例方法里直接使用 self 关键字，跟OC一样 self 是指当前对象，这个 self 关键字是怎样实现的呢？实际上这个self是个全局变量，在 defineClass 里对实例方法进行了包装，在调用实例方法之前，会把全局变量 self 设为当前对象，调用完后设回空，就可以在执行实例方法的过程中使用 self 变量了。这是一个小小的trick。

### 7.super关键字

```source-js
defineClass("JPViewController: UIViewController", {
  viewDidLoad: function() {
    self.super().viewDidLoad()
  },
}
```

OC 里 super 是一个关键字，无法通过动态方法拿到 super，那么 JSPatch 的 super 是怎么实现的？实际上调用 super 的方法，OC 做的事是调用父类的某个方法，并把当前对象当成 self 传入父类方法，我们只要模拟它这个过程就行了。

首先 JS 端需要告诉OC想调用的是当前对象的 super 方法，做法是调用 `self.super()`时，`__c`函数会做特殊处理，返回一个新的对象，这个对象同样保存了 OC 对象的引用，同时标识 `__isSuper = 1`。

```source-js
...
if (methodName == 'super') {
  return function() {
    return {__obj: self.__obj, __clsName: self.__clsName, __isSuper: 1}
  }
}
...
```

再用这个返回的对象去调用方法时，`__c` 函数会把 `__isSuper` 这个标识位传给 OC，告诉 OC 要调 super 的方法。OC 做的事情是，如果是调用 super 方法，找到 superClass 这个方法的 IMP 实现，为当前类新增一个方法指向 super 的 IMP 实现，那么调用这个类的新方法就相当于调用 super 方法。把要调用的方法替换成这个新方法，就完成 super 方法的调用了。

```source-objc
static id callSelector(NSString *className, NSString *selectorName, NSArray *arguments, id instance, BOOL isSuper) {
    ...
    if (isSuper) {
        NSString *superSelectorName = [NSString stringWithFormat:@"SUPER_%@", selectorName];
        SEL superSelector = NSSelectorFromString(superSelectorName);

        Class superCls = [cls superclass];
        Method superMethod = class_getInstanceMethod(superCls, selector);
        IMP superIMP = method_getImplementation(superMethod);

        class_addMethod(cls, superSelector, superIMP, method_getTypeEncoding(superMethod));
        selector = superSelector;
    }
    ...
}
```

## 扩展

### 1.Struct 支持

struct 类型在 JS 与 OC 间传递需要做转换处理，一开始 JSPatch 只处理了原生的 NSRange / CGRect / CGSize / CGPoint 这四个，其他 struct 类型无法在 OC / JS 间传递。对于其他类型的 struct 支持，我是采用扩展的方式，让写扩展的人手动处理每个要支持的 struct 进行类型转换，这种做法没问题，但需要在 OC 代码写好这些扩展，无法动态添加，转换的实现也比较繁琐。于是转为另一种实现：

```source-js
/*
struct JPDemoStruct {
  CGFloat a;
  long b;
  double c;
  BOOL d;
}
*/
require('JPEngine').defineStruct({
  "name": "JPDemoStruct",
  "types": "FldB",
  "keys": ["a", "b", "c", "d"]
})
```

可以在 JS 动态定义一个新的 struct，只需提供 struct 名，每个字段的类型以及每个字段对应的在 JS 的键值，就可以支持这个 struct 类型在 JS 和 OC 间传递了：

```
//OC
@implementation JPObject
+ (void)passStruct:(JPDemoStruct)s;
+ (JPDemoStruct)returnStruct;
@end

```

```
//JS
require('JPObject').passStruct({a:1, b:2, c:4.2, d:1})
var s = require('JPObject').returnStruct();

```

这里的实现原理是顺序去取 struct 里每个字段的值，再根据 key 重新包装成 NSDictionary 传给 JS，怎样顺序取 struct 每字段的值呢？可以根据传进来的 struct 字段的变量类型，拿到类型对应的长度，顺序拷贝出 struct 对应位置和长度的值，具体实现：

```source-objc
for (int i = 0; i < types.count; i ++) {
  size_t size = sizeof(types[i]);  //types[i] 是 float double int 等类型
  void *val = malloc(size);
  memcpy(val, structData + position, size);
  position += size;
}
```

struct 从 JS 到 OC 的转换同理，只是反过来，先生成整个 struct 大小的内存地址（通过 struct 所有字段类型大小累加），再逐渐取出 JS 传过来的值进行类型转换拷贝到这端内存里。

这种做法效果很好，JS 端要用一个新的 struct 类型，不需要 OC 事先定义好，可以直接动态添加新的 struct 类型支持，但这种方法依赖 struct 各个字段在内存空间上的严格排列，如果某些机器在底层实现上对 struct 的字段进行一些字节对齐之类的处理，这种方式没法用了，不过目前在 iOS 上还没碰到这样的问题。

### 2.C 函数支持

C 函数没法通过反射去调用，所以只能通过手动转接的方式让 JS 调 C 方法，具体就是通过 JavaScriptCore 的方法在 JS 当前作用域上定义一个 C 函数同名方法，在这个方法实现里调用 C 函数，以支持 `memcpy()` 为例：

```source-objc
context[@"memcpy"] = ^(JSValue *des, JSValue *src, size_t n) {
    memcpy(des, src, n);
};
```

这样就可以在 JS 调用 `memcpy()` 函数了。实际上这里还有参数 JS  OC 转换问题，这里先忽略。

这里有两个问题：

a.如果这些 C 函数的支持都写在 JSPatch 源文件里，源文件会非常庞大。 b.如果一开始就给 JS 加这些函数定义，若要支持的 C 函数量大时会影响性能。

对此设计了一种扩展的方式去解决这两个问题，JSPatch 需要做的就是为外部提供 JS 运行上下文 JSContext，以及参数转换的方法，最终设计出来的扩展接口是这样：

```source-objc
@interface JPExtension : NSObject
+ (void)main:(JSContext *)context;

+ (void *)formatPointerJSToOC:(JSValue *)val;
+ (id)formatPointerOCToJS:(void *)pointer;
+ (id)formatJSToOC:(JSValue *)val;
+ (id)formatOCToJS:(id)obj;

@end
```

`+main` 方法暴露了 JSPatch 的运行环境 JSContext 给外部，可以自由在这个 JSContext 上加函数。另外四个 `formatXXX` 方法都是参数转换方法。上述的 `memcpy()` 完整的扩展定义如下：

```source-objc
@implementation JPMemory
+ (void)main:(JSContext *)context
{
    context[@"memcpy"] = ^id(JSValue *des, JSValue *src, size_t n) {
        void *ret = memcpy([self formatPointerJSToOC:des], [self formatPointerJSToOC:src], n);
        return [self formatPointerOCToJS:ret];
    };
}
@end
```

同时 JSPatch 提供了 `+addExtensions:` 接口，让 JS 端可以动态加载某个扩展，在需要的时候再给 JS 上下文添加这些 C 函数：

```source-js
require('JPEngine').addExtensions(['JPMemory'])
```

实际上还有另一种方法添加 C 函数的支持，就是定义 OC 方法转接：

```source-objc
@implementation JPCFunctions
+ (void)memcpy:(void *)des src:(void *)src n:(size_t)n {
  memcpy(des, src, n);
}
@end
```

然后直接在 JS 上这样调：

```source-js
require('JPFunctions').memcpy_src_n(des, src, n);
```

这样的做法不需要扩展机制，也不需要在实现时进行参数转换，但因为它走的是 OC runtime 那一套，相比扩展直接调用的方式，速度慢了一倍，为了更好的性能，还是提供一套扩展接口。

## 细节

整个 JSPatch 的基础原理上面大致阐述完了，接下来在看看一些实现上碰到的坑和的细节问题。

### 1.Special Struct

上文提到会把要覆盖的方法指向`_objc_msgForward`，进行转发操作，这里出现一个问题，如果替换方法的返回值是某些 struct，使用 `_objc_msgForward`（或者之前的 `@selector(__JPNONImplementSelector)`）会 crash。几经辗转，找到了解决方法：对于某些架构某些 struct，必须使用 `_objc_msgForward_stret` 代替 `_objc_msgForward`。为什么要用 `_objc_msgForward_stret` 呢，找到一篇说明 `objc_msgSend_stret` 和 `objc_msgSend` 区别的[文章](http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html)：说得比较清楚，原理是一样的，是C的一些底层机制的原因，简单复述一下：

大多数 CPU 在执行 C 函数时会把前几个参数放进寄存器里，对 `obj_msgSend` 来说前两个参数固定是 self / _cmd，它们会放在寄存器上，在最后执行完后返回值也会保存在寄存器上，取这个寄存器的值就是返回值：

```source-objc
-(int) method:(id)arg;
    r3 = self
    r4 = _cmd, @selector(method:)
    r5 = arg
    (on exit) r3 = returned int
```

普通的返回值(int/pointer)很小，放在寄存器上没问题，但有些 struct 是很大的，寄存器放不下，所以要用另一种方式，在一开始申请一段内存，把指针保存在寄存器上，返回值往这个指针指向的内存写数据，所以寄存器要腾出一个位置放这个指针，self / _cmd 在寄存器的位置就变了：

```source-objc
 -(struct st) method:(id)arg;
    r3 = &struct_var (in caller's stack frame)
    r4 = self
    r5 = _cmd, @selector(method:)
    r6 = arg
    (on exit) return value written into struct_var
```

`objc_msgSend` 不知道 self / _cmd 的位置变了，所以要用另一个方法 `objc_msgSend_stret` 代替。原理大概就是这样。

上面说某些架构某些 struct 有问题，那具体是哪些呢？iOS 架构中非 arm64 的都有这问题，而怎样的 struct 需要走上述流程用 `xxx_stret` 代替原方法则没有明确的规则，OC 也没有提供接口，只有在一个奇葩的接口上透露了这个天机，于是有这样一个神奇的判断：

```source-objc
if ([methodSignature.debugDescription rangeOfString:@"is special struct return? YES"].location != NSNotFound)
```

在 `NSMethodSignature` 的 `debugDescription` 上打出了是否 special struct，只能通过这字符串判断。所以最终的处理是，在非 arm64 下，是 special struct 就走 `_objc_msgForward_stret`，否则走 `_objc_msgForward`。

### 2.内存问题

#### i.Double Release

实现过程中碰到一些内存问题，首先是 Double Release 问题。从 `-forwardInvocation:` 里的 NSInvocation 对象取参数值时，若参数值是id类型，我们会这样取:

```source-objc
id arg;
[invocation getArgument:&arg atIndex:i];
```

但这样的写法会导致 crash，这是因为 `id arg` 在ARC下相当于 `__strong id arg`，若这时在代码显式为 arg 赋值，根据 ARC 的机制，会自动插入一条 retain 语句，然后在退出作用域时插入 release 语句：

```source-objc
- (void)method {
id arg = [SomeClass getSomething];
// [arg retain]
...
// [arg release]  退出作用域前release
}
```

但我们这里不是显式对 arg 进行赋值，而是传入 `-getArgument:atIndex:` 方法，在这里面赋值后 ARC 没有自动给这个变量插入 retain 语句，但退出作用域时还是自动插入了 release 语句，导致这个变量多释放了一次，导致 crash。解决方法是把 arg 变量设成 __unsafe_unretained 或 __weak，让 ARC 不在它退出作用域时插入 release 语句即可：

```source-objc
__unsafe_unretained id arg;
[invocation getReturnValue:&arg];
```

还可以通过 `__bridge` 转换让局部变量持有返回对象，这样做也是没问题的：

```source-objc
id returnValue;
void *result;
[invocation getReturnValue:&result];
returnValue = (__bridge id)result;
```

#### ii.内存泄露

Double Release 的问题解决了，又碰到内存泄露的坑。某天 github issue 上有人提对象生成后没有释放，几经排查，定位到还是这里 `NSInvocation getReturnValue` 的问题，当 `NSInvocation`调用的是 `alloc` 时，返回的对象并不会释放，造成内存泄露，只有把返回对象的内存管理权移交出来，让外部对象帮它释放才行：

```source-objc
id returnValue;
void *result;
[invocation getReturnValue:&result];
if ([selectorName isEqualToString:@"alloc"] || [selectorName isEqualToString:@"new"]) {
    returnValue = (__bridge_transfer id)result;
} else {
    returnValue = (__bridge id)result;
}
```

这是因为 ARC 对方法名有约定，当方法名开头是 alloc / new / copy / mutableCopy 时，返回的对象是 retainCount = 1 的，除此之外，方法返回的对象都是 autorelease 的，按上一节的说法，对于普通方法返回值，ARC 会在赋给 strong 变量时自动插入 retain 语句，但对于 alloc 等这些方法，不会再自动插入 retain 语句：

```source-objc
id obj = [SomeObject alloc];
//alloc 方法返回的对象 retainCount 已 +1，这里不需要retain

id obj2 = [SomeObj someMethod];
//方法返回的对象是 autorelease，ARC 会再这里自动插入 [obj2 retain] 语句
```

而 ARC 并没有处理非显式调用时的情况，这里动态调用这些方法时，ARC 都不会自动插入 retain，这种情况下，alloc / new 等这类方法返回值的 retainCount 是会比其他方法返回值多1的，所以需要特殊处理这类方法。

### 3.‘_’的处理

JSPatch 用下划线’_’连接OC方法多个参数间的间隔：

```source-objc
- (void)setObject:(id)anObject forKey:(id)aKey;
<==>
setObject_forKey()
```

那如果OC方法名里含有’_’，那就出现歧义了：

```source-objc
- (void)set_object:(id)anObject forKey:(id)aKey;
<==>
set_object_forKey()
```

没法知道 `set_object_forKey` 对应的 selector 是 `set_object:forKey:` 还是 `set:object:forKey:`。

对此需要定个规则，在 JS 用其他字符代替 OC 方法名里的 `_`。JS 命名规则除了字母和数字，就只有 `$` 和 `_`，看起来只能用 `$` 代替了，但效果很丑：

```source-objc
- (void)set_object:(id)anObject forKey:(id)aKey;
- (void)_privateMethod();
<==>
set$object_forKey()
$privateMethod()
```

于是尝试另一种方法，用两个下划线 `__` 代替：

```
set__object_forKey()
__privateMethod()

```

但用两个下划线代替有个问题，OC 方法名参数后面加下划线会匹配不到

```
- (void)setObject_:(id)anObject forKey:(id)aKey;
<==>
setObject___forKey()

```

实际上 `setObject___forKey()` 匹配到对应的 selector 是 `setObject:_forKey:`。虽然有这个坑，但因为很少见到这种奇葩的命名方式，感觉问题不大，使用$也会导致替换不了 OC 方法名包含 `$` 字符的，最终为了代码颜值，使用了双下划线 `__` 表示。

### 4.JPBoxing

在使用 JSPatch 过程中发现JS无法调用 `NSMutableArray` / `NSMutableDictionary` / `NSMutableString` 的方法去修改这些对象的数据，因为这三者都在从 OC 返回到 JS 时 JavaScriptCore 把它们转成了 JS 的 `Array` / `Object` / `String`，在返回的时候就脱离了跟原对象的联系，这个转换在 JavaScriptCore 里是强制进行的，无法选择。

若想要在对象返回 JS 后，回到 OC 还能调用这个对象的方法，就要阻止 JavaScriptCore 的转换，唯一的方法就是不直接返回这个对象，而是对这个对象进行封装，JPBoxing 就是做这个事情的：

```source-objc
@interface JPBoxing : NSObject
@property (nonatomic) id obj;
@end

@implementation JPBoxing
+ (instancetype)boxObj:(id)obj
{
   JPBoxing *boxing = [[JPBoxing alloc] init];
    boxing.obj = obj;
    return boxing;
}
```

把 `NSMutableArray` / `NSMutableDictionary` / `NSMutableString` 对象作为 `JPBoxing` 的成员保存在 `JPBoxing` 实例对象上返回给 JS，JS 拿到的是 `JPBoxing` 对象的指针，再传回给 OC 时就可以通过对象成员取到原来的 `NSMutableArray` / `NSMutableDictionary` / `NSMutableString` 对象，类似于装箱/拆箱操作，这样就避免了这些对象被 JavaScriptCore 转换。

实际上只有可变的 `NSMutableArray` / `NSMutableDictionary` / `NSMutableString` 这三个类有必要调用它的方法去修改对象里的数据，不可变的 `NSArray` / `NSDictionary` / `NSString` 是没必要这样做的，直接转为 JS 对应的类型使用起来会更方便，但为了规则简单，JSPatch 让 `NSArray` / `NSDictionary` / `NSString` 也同样以封装的方式返回，避免在调用 OC 方法返回对象时还需要关心它返回的是可变还是不可变对象。最后整个规则还是挺清晰：`NSArray` / `NSDictionary` / `NSString` 及其子类与其他 `NSObject` 对象的行为一样，在 JS 上拿到的都只是其对象指针，可以调用它们的 OC 方法，若要把这三种对象转为对应的 JS 类型，使用额外的 `.toJS()` 的接口去转换。

对于参数和返回值是C指针和 Class 类型的支持同样是用 JPBoxing 封装的方式，把指针和 Class 作为成员保存在 JPBoxing 对象上返回给 JS，传回 OC 时再解出来拿到原来的指针和 Class，这样 JSPatch 就支持所有数据类型 OCJS 的互传了。

### 5.nil的处理

#### i.区分NSNull/nil

对于"空"的表示，JS 有 `null` / `undefined`，OC 有 `nil` / `NSNull`，JavaScriptCore 对这些参数传递处理是这样的：

* 从 JS 到 OC，直接传递 `null` / `undefined` 到 `OC` 都会转为 `nil`，若传递包含 `null` / `undefined` 的 `Array` 给 OC，会转为 `NSNull`。
* 从 OC 到 JS，`nil` 会转为 `null`，`NSNull` 与普通 `NSObject` 一样返回指针。

JSPatch 的流程上都是通过数组的方式把参数从 JS 传入 OC，这样所有的 `null` / `undefined` 到 OC 就都变成了 `NSNull`，而真正的 `NSNull` 对象传进来也是 `NSNull`，无法分辨从 JS 过来实际传的是什么，需要有种方式区分这两者。

考虑过在 JS 用一个特殊的对象代表 `nil`，`null` / `undefined` 只用来表示 `NSNull`，后来觉得 `NSNull` 是很少手动传递的变量，而 `null` / `undefined` 以及 OC 的 `nil` 却很常见，这样做会给日常开发带来很大不便。于是反过来，在 JS 用一个特殊变量 `nsnull` 表示 `NSNull`，其他 `null` / `undefined` 表示 `nil`，这样传入 OC 就可以分辨出 `nil` 和 `NSNull`，具体使用方式：

```source-objc
@implementation JPObject
+ (void)testNil:(id)obj
{
     NSLog(@"%@", obj);
}
@end

require("JPObject").testNil(null)      //output: nil
require("JPObject").testNil(nsnull)      //output: NSNull
```

这样做有个小坑，就是显式使用 `NSNull.null()` 作为参数调用时，到 OC 后会变成 `nil`：

```source-js
require("JPObject").testNil(require("NSNull").null())     //output: nil
```

这个只需注意下用 `nsnull` 代替就行，从 OC 返回的 `NSNull` 再回传回去还是可以识别到 `NSNull`。

#### ii.链式调用

第二个问题，`nil` 在 JS 里用 `null` / `undefined` 表示，造成的后果是无法用 `nil` 调用方法，也就无法保证链式调用的安全：

```source-objc
@implementation JPObject
+ (void)returnNil
{
     return nil;
}
@end

[[JPObject returnNil] hash]     //it’s OK

require("JPObject").returnNil().hash()     //crash
```

原因是在 JS 里 `null` / `undefined` 不是对象，无法调用任何方法，包括我们给所有对象加的 `__c()` 方法。解决方式一度觉得只有回到上面说的，用一个特殊的对象表示 `nil`，才能解决这个问题了。但使用特殊的对象表示 `nil`，后果就是在 js 判断是否为 `nil` 时就要很啰嗦：

```source-js
//假设用一个_nil对象变量表示OC返回的nil
var obj = require("JPObject").returnNil()
obj.hash()     //经过特殊处理没问题
if (!obj || obj == _nil) {
     //判断对象是否为nil就得附加判断是否等于_nil
}
```

这样的使用方式难以接受，继续寻找解决方案，发现 `true` / false 在 JS 是个对象，是可以调用方法的，如果用 `false` 表示 `nil`，即可以做到调用方法，又可以直接通过 `if (!obj)` 判断是否为 `nil`，于是沿着这个方向，解决了用 `false` 表示 `nil` 带来的各种坑，几乎完美地解决了这个问题。实现上的细节就不多说了，说"几乎完美"，是因为还有一个小坑，传递 `false` 给 OC 上参数类型是 `NSNumber*` 的方法，OC 会得到 nil 而不是 NSNumber 对象：

```source-objc
@implementation JPObject
+ (void)passNSNumber:(NSNumber *)num {
     NSLog(@"%@", num);
}
@end

require("JPObject").passNSNumber(false) //output: nil
```

如果 OC 方法的参数类型是 `BOOL`，或者传入的是 `true` / `0`，都是没问题的，这小坑无伤大雅。

题外话，神奇的 JS 里 `false` 的 `this` 竟然不再是原来的 `false`，而是另一个 `Boolean` 对象，太特殊了：

```
Object.prototype.c = function(){console.log(this === false)};
false.c() //output false

```

有人做出了解释 [#351](https://github.com/bang590/JSPatch/issues/351)

## 总结

JSPatch 的原理以及一些实现细节就阐述到这里，希望这篇文章对大家了解和使用 JSPatch 有帮助。


