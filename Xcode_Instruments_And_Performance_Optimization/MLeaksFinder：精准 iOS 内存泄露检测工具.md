**原文链接：** [MLeaksFinder：精准 iOS 内存泄露检测工具](http://wereadteam.github.io/2016/02/22/MLeaksFinder/)

**Keywords:** `iOS`，`性能优化`，`内存泄漏`

# MLeaksFinder：精准 iOS 内存泄露检测工具

# 背景

平常我们都会用 Instrument 的 Leaks / Allocations 或其他一些开源库进行内存泄露的排查，但它们都存在各种问题和不便，我们逐个来看这些工具的使用和存在的问题。

### Leaks

先看看 Leaks，从苹果的开发者文档里可以看到，一个 app 的内存分三类：

* **Leaked memory**: Memory unreferenced by your application that cannot be used again or freed (also detectable by using the Leaks instrument).

* **Abandoned memory**: Memory still referenced by your application that has no useful purpose.

* **Cached memory**: Memory still referenced by your application that might be used again for better performance.

其中 Leaked memory 和 Abandoned memory 都属于应该释放而没释放的内存，都是内存泄露，而 Leaks 工具只负责检测 Leaked memory，而不管 Abandoned memory。在 MRC 时代 Leaked memory 很常见，因为很容易忘了调用 release，但在 ARC 时代更常见的内存泄露是循环引用导致的 Abandoned memory，Leaks 工具查不出这类内存泄露，应用有限。

### Allocations

对于 Abandoned memory，可以用 Instrument 的 Allocations 检测出来。检测方法是用 Mark Generation 的方式，当你每次点击 Mark Generation 时，Allocations 会生成当前 App 的内存快照，而且 Allocations 会记录从上回内存快照到这次内存快照这个时间段内，新分配的内存信息。举一个最简单的例子：

我们可以不断重复 push 和 pop 同一个 UIViewController，理论上来说，push 之前跟 pop 之后，app 会回到相同的状态。因此，在 push 过程中新分配的内存，在 pop 之后应该被 dealloc 掉，除了前几次 push 可能有预热数据和 cache 数据的情况。如果在数次 push 跟 pop 之后，内存还不断增长，则有内存泄露。因此，我们在每回 push 之前跟 pop 之后，都 Mark Generation 一下，以此观察内存是不是无限制增长。这个方法在 WWDC 的视频里：[Session 311 - Advanced Memory Analysis with Instruments](http://developer.apple.com/videos/wwdc/2010/)，以及苹果的开发者文档：[Finding Abandoned Memory](https://developer.apple.com/library/mac/recipes/Instruments_help_articles/FindingAbandonedMemory/FindingAbandonedMemory.html) 里有介绍。

用这种方法来发现内存泄露还是很不方便的：

* 首先，你得打开 Allocations
* 其次，你得一个个场景去重复的操作
* 无法及时得知泄露，得专门做一遍上述操作，十分繁琐

### 开源库

在 GitHub 上有一些内存泄露检测相关的项目，例如 [HeapInspector-for-iOS](https://github.com/tapwork/HeapInspector-for-iOS) 和 [MSLeakHunter](https://github.com/mindsnacks/MSLeakHunter)。

HeapInspector-for-iOS 可以说是 Allocations 的改进。它通过 hook 掉 alloc，dealloc，retain，release 等方法，来记录对象的生命周期。具体的检测内存泄露的方法和原理，与 Instrument 的 Allocations 一致。然而它跟 Allocations 一样，存在的问题是，你需要一个个场景去重复的操作，还有检测不及时。

MSLeakHunter 就简单得多，它只检测 UIViewController 和 UIView，通过 hook 掉 UIViewController 的 `-viewDidDisappear:` 方法，并认为 `-viewDidDisappear:` 后，UIViewController 将很快被释放，如果 UIViewController 没有被释放，则打个建议日志。这种做法其实不是很好，`-viewDidDisappear:` 被调用可能是因为又 push 进来一个新的 ViewController，把当前的 ViewController 挡住了，所以可能有很多错误的建议，需要结合你实际的操作去具体地分析日志。

# MLeaksFinder

[MLeaksFinder](https://github.com/Zepo/MLeaksFinder) 提供了内存泄露检测更好的解决方案。只需要引入 MLeaksFinder，就可以自动在 App 运行过程检测到内存泄露的对象并立即提醒，无需打开额外的工具，也无需为了检测内存泄露而一个个场景去重复地操作。MLeaksFinder 目前能自动检测 UIViewController 和 UIView 对象的内存泄露，而且也可以扩展以检测其它类型的对象。

MLeaksFinder 的使用很简单，参照 [https://github.com/Zepo/MLeaksFinder](https://github.com/Zepo/MLeaksFinder)，基本上就是把 MLeaksFinder 目录下的文件添加到你的项目中，就可以在运行时（debug 模式下）帮助你检测项目里的内存泄露了，无需修改任何业务逻辑代码，而且只在 debug 下开启，完全不影响你的 release 包。

当发生内存泄露时，MLeaksFinder 会中断言，并准确的告诉你哪个对象泄露了。这里设计为中断言而不是打日志让程序继续跑，是因为很多人不会去看日志，断言则能强制开发者注意到并去修改，而不是犯拖延症。

中断言时，控制台会有如下提示，View-ViewController stack 从上往下看，该 stack 告诉你，MyTableViewController 的 UITableView 的 subview UITableViewWrapperView 的 subview MyTableViewCell 没被释放。而且，这里我们可以肯定的是 MyTableViewController，UITableView，UITableViewWrapperView 这三个已经成功释放了。

```
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Possibly Memory Leak.
In case that MyTableViewCell should not be dealloced, override -willDealloc in MyTableViewCell by returning NO.
View-ViewController stack: (
 MyTableViewController,
 UITableView,
 UITableViewWrapperView,
 MyTableViewCell
)'
```

从 MLeaksFinder 的使用方法可以看出，MLeaksFinder 具备以下优点：

* 使用简单，不侵入业务逻辑代码，不用打开 Instrument
* 不需要额外的操作，你只需开发你的业务逻辑，在你运行调试时就能帮你检测
* 内存泄露发现及时，更改完代码后一运行即能发现（这点很重要，你马上就能意识到哪里写错了）
* 精准，能准确地告诉你哪个对象没被释放

# 原理

MLeaksFinder 一开始从 UIViewController 入手。我们知道，当一个 UIViewController 被 pop 或 dismiss 后，该 UIViewController 包括它的 view，view 的 subviews 等等将很快被释放（除非你把它设计成单例，或者持有它的强引用，但一般很少这样做）。于是，我们只需在一个 ViewController 被 pop 或 dismiss 一小段时间后，看看该 UIViewController，它的 view，view 的 subviews 等等是否还存在。

具体的方法是，为基类 NSObject 添加一个方法 `-willDealloc` 方法，该方法的作用是，先用一个弱指针指向 self，并在一小段时间(3秒)后，通过这个弱指针调用 `-assertNotDealloc`，而 `-assertNotDealloc` 主要作用是直接中断言。

```objc
- (BOOL)willDealloc {
 __weak id weakSelf = self;
 dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
 [weakSelf assertNotDealloc];
 });
 return YES;
}
- (void)assertNotDealloc {
 NSAssert(NO, @“”);
}
```

这样，当我们认为某个对象应该要被释放了，在释放前调用这个方法，如果3秒后它被释放成功，weakSelf 就指向 nil，不会调用到 `-assertNotDealloc` 方法，也就不会中断言，如果它没被释放（泄露了），`-assertNotDealloc` 就会被调用中断言。这样，当一个 UIViewController 被 pop 或 dismiss 时（我们认为它应该要被释放了），我们遍历该 UIViewController 上的所有 view，依次调 `-willDealloc`，若3秒后没被释放，就会中断言。

在这里，有几个问题需要解决：

1. 不入侵开发代码

    这里使用了 AOP 技术，hook 掉 UIViewController 和 UINavigationController 的 pop 跟 dismiss 方法，关于如何 hook，请参考 [Method Swizzling](http://nshipster.com/method-swizzling/)。

2. 遍历相关对象

    在实际项目中，我们发现有时候一个 UIViewController 被释放了，但它的 view 没被释放，或者一个 UIView 被释放了，但它的某个 subview 没被释放。这种内存泄露的情况很常见，因此，我们有必要遍历基于 UIViewController 的整棵 View-ViewController 树。我们通过 UIViewController 的 presentedViewController 和 view 属性，UIView 的 subviews 属性等递归遍历。对于某些 ViewController，如 UINavigationController，UISplitViewController 等，我们还需要遍历 viewControllers 属性。

3. 构建堆栈信息

    需要构建 View-ViewController stack 信息以告诉开发者是哪个对象没被释放。在递归遍历 View-ViewController 树时，子节点的 stack 信息由父节点的 stack 信息加上子结点信息即可。

4. 例外机制

    对于有些 ViewController，在被 pop 或 dismiss 后，不会被释放（比如单例），因此需要提供机制让开发者指定哪个对象不会被释放，这里可以通过重载上面的 `-willDealloc` 方法，直接 return NO 即可。

5. 特殊情况

    对于某些特殊情况，释放的时机不大一样（比如系统手势返回时，在划到一半时 hold 住，虽然已被 pop，但这时还不会被释放，ViewController 要等到完全 disappear 后才释放），需要做特殊处理，具体的特殊处理视具体情况而定。

6. 系统View

    某些系统的私有 View，不会被释放（可能是系统 bug 或者是系统出于某些原因故意这样做的，这里就不去深究了），因此需要建立白名单

7. 手动扩展

    MLeaksFinder目前只检测 ViewController 跟 View 对象。为此，MLeaksFinder 提供了一个手动扩展的机制，你可以从 UIViewController 跟 UIView 出发，去检测其它类型的对象的内存泄露。如下所示，我们可以检测 UIViewController 底下的 View Model：

```objc
- (BOOL)willDealloc {
 if (![super willDealloc]) {
 return NO;
 }
 MLCheck(self.viewModel);
 return YES;
}
```

这里的原理跟上面的是一样的，宏 MLCheck() 做的事就是为传进来的对象建立 View-ViewController stack 信息，并对传进来的对象调用 `-willDealloc` 方法。

# 未来

MLeaksFinder 目前还在起步阶段，它的内存泄露检测的想法是很简单，很直接的。虽然目前只能自动地检测 UIViewController 和 UIView 相关的对象，然而在我们几个大的项目中，已经起到很大的作用，帮助我们发现很多历史存在的内存泄露，而且确保新提交的 UI 相关代码不会引进新的问题。MLeaksFinder 会继续探索覆盖更广的情况，提供更全面的检测，包括网络层，数据存储层等等。

