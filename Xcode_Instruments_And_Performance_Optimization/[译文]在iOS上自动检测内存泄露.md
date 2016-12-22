**原文链接：** [(译文)在iOS上自动检测内存泄露](http://ifujun.com/yi-wen-zai-iosshang-zi-dong-jian-ce-nei-cun-xie-lu/)

**Keywords:** `iOS`，`内存泄漏检测`，`性能优化`

## [译文]在iOS上自动检测内存泄露

原文链接：[https://code.facebook.com/posts/583946315094347/automatic-memory-leak-detection-on-ios/](https://code.facebook.com/posts/583946315094347/automatic-memory-leak-detection-on-ios/)

手机设备的内存是一个共享资源。应用程序可能会不当的耗尽内存、崩溃，或者遭遇大幅度的性能降低。

Facebook iOS客户端有很多功能，并且它们共享同一块内存空间。如果任何特定的功能消耗过多的内存，就会影响到整个应用程序。这是可能发生的，比如，这个功能导致了内存泄露。

当我们分配了一块内存，并设置了对象之后，如果在使用完了之后忘记释放，这就会发生内存泄露。这意味着系统是无法回收内存并交予他人使用，这也最终意味着我们的内存将会逐渐耗尽。

在Facebook，我们有很多工程师在代码库的不同部分上工作。这不可避免的会发生内存泄露。当发生内存泄露之后，我们需要尽快找到并修复它们。

一些工具已经可以找到内存泄露，但是它们需要大量的人工干预：

1. 打开Xcode，给性能分析(profiling)编译。
2. 载入Instruments。
3. 使用应用程序，尝试尽可能多的重现场景和行为。
4. 查看内存和泄露。
5. 追踪内存泄露的根源。
6. 修复这个问题。

这意味着每次都需要重复大量的手动操作。为此，在我们的开发周期上，我们可能无法尽可能早的定位和修复内存泄露问题。

自动化可以在不需要更多开发者的情况下，更快的找到内存泄露。为了解决这个问题，我们做了一套工具来自动化的处理和修复我们代码库中的一些问题。今天，我们很兴奋的发布这些工具：[FBRetainCycleDetector](https://github.com/facebook/FBRetainCycleDetector)、[FBAllocationTracker](https://github.com/facebook/FBAllocationTracker)、[FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)。

## 循环引用（Retain cycles）

Objective-C 使用引用计数去管理内存和释放不使用的对象。内存中的任何一个对象都可以持有(`retain`)其他的对象，只要前面的对象需要它，对象就会一直保持在内存中。查看这个的一个方法是这个对象持有其他的对象。

在大部分时间内，这都工作的很好，但当两个对象互相持有的时候，这就会陷入一个僵局。直接，或者更常见的，通过间接对象连接它们。这种持有引用的环我们叫做循环引用（Retain cycles）。

![](http://7i7i81.com1.z0.glb.clouddn.com/blogimage_memoryleak_1.png)

循环引用会导致一些列的问题。最好的情况下，对象只会在内存中占有一点点位置。如果这个被泄露的对象正积极地做个一些不平凡的事情，应用程序的其他部分就只会有更少的内存。最坏的情况下，如果泄露导致使用超出可用内存的容量，那么，应用程序会崩溃。

在手动性能分析期间，我们发现，我们往往有一些循环引用。我们很容易引起内存泄露，但是很难找到它们。循环引用检测器可以很容易的找到它们。

## 在运行时检测循环引用

在 Objective-C 中找循环引用类似于在一个有向无环图(directed acyclic graph)中找环, 而节点就是对象，边就是对象之间的引用(如果对象A持有对象B，那么，A到B之间就存在着引用)。我们的 Objective-C 对象已经在我们的图中，我们要做的就是用深度优先搜索遍历它。

<iframe height=300 width="100%" src="http://7i7i81.com1.z0.glb.clouddn.com/blogvideo_memoryleak_1.mp4" allowfullscreen="true"></iframe>

这有点抽象，但效果很好。我们必须确保我们可以像节点一样使用对象，对于每个对象，我们都可以获取到它引用的所有对象。这些引用可能是`weak`，也可能是`strong`。只有强引用才会导致循环引用。对于每个对象来说，我们需要知道如何找出这些引用。

幸运的是，Objective-C提供了一个强有力的、内省的运行时库。这让我们在图中可以有足够的数据去挖掘。

图中的节点可以是对象，也可以是Block。让我们来分别讨论一下。

### 对象

运行时有很多工具允许我们对对象进行内省。

我们要做的第一件事是获取对象的实例变量的布局（ivar layout）。

```
const char *class_getIvarLayout(Class cls);
const char *class_getWeakIvarLayout(Class cls);

```

对于对象，实例变量的布局描述了我们在哪儿可以找到其他对象的引用。它会提供给我们一个索引（index），这代表我们需要在对象地址上添加一个偏移量(offset)，就可以得到它所引用的对象的地址。运行时也允许我们获取“弱引用实例变量布局(weak ivar layout)”。

这也部分支持Objective-C++。在Objective-C++中，我们可以在结构体中定义对象，但是这不会在实例变量布局中获取到。运行时提供了“类型编码(type encoding)”来处理这个问题。对于每一个实例变量来说，类型编码描述了变量是如何结构化的。如果这是一个结构体，它会描述它包含了哪些字段和类型。我们计算出它们的偏移量，在图中，找出它们所指向的对象。

也有一些边缘条件我们不会深入。大部分是一些不同的集合，我们不得不列举它们去获得它们持有的对象，这可能会导致一些副作用。

## Block

Block和对象有一点不一样。运行时不会让我们很轻易的看到它们的布局，但是我们仍然可以猜测。

在处理Block的时候，我们可以使用 Mike Ash 在他的项目[Circle](https://github.com/mikeash/Circle)(第一时间启发[FBRetainCycleDetector](https://github.com/facebook/FBRetainCycleDetector)的项目)中提出的想法。

我们可以使用的是[ABI](http://clang.llvm.org/docs/Block-ABI-Apple.html)(application binary interface for blocks - 应用程序二进制Block接口)。它描述了Block在内存中的样子。如果我们知道我们在处理的引用是一个Block,我们可以把它丢在一个假的结构体中来模仿Block。在放到一个C语言的结构体之后，我们可以知道Block所持有的对象。不幸的是，我们不知道这些引用是强引用还是弱引用。

为了解决这个问题，我们使用了一个黑盒技术。我们创建一个对象来假扮我们想要调查的Block。因为我们知道Block的接口，我们知道在哪可以找到Block持有的引用。我们伪造的对象将会拥有“释放检测(release detectors)”来代替这些引用。释放检测器是一些很小的对象，它们会观察发送给它们的释放消息。当持有者想要放弃它的持有的时候，这些消息会发送给强引用对象。当我们释放我们伪造的对象的时候，我们可以检测哪些检测器接收到了这些消息。只要知道哪些索引在伪造的对象的检测器中，我们就可以找到原来Block中实际持有的对象。

![](http://7i7i81.com1.z0.glb.clouddn.com/blogimage_memoryleak_2.png)

## 自动化

让这工具真正闪光的是，在工程师内部构建的时候，它会连续的、自动的运行。

客户端部分自动化是简单的。我们在定时器上运行循环引用检测器，定期扫描内存去寻找循环引用，虽然这不是完全没有问题。当我们第一次运行分析器的时候，我们意识到它不足以很快的扫描整个内存空间。当它开始检测的时候，我们需要给它提供一组候选对象。

为了更有效的解决这个问题，我们开发了[FBAllocationTracker](https://github.com/facebook/FBAllocationTracker)。这个工具会主动跟踪`NSObject`子类的创建和释放。它可以以一个很小的性能开销来获取任何类的任何实例。

对于客户端的自动化，只要在`NSTimer`上使用[FBRetainCycleDetector](https://github.com/facebook/FBRetainCycleDetector)，再用[FBAllocationTracker](https://github.com/facebook/FBAllocationTracker)来抓取实例来配合跟踪就行。

现在，让我们来仔细看看后台会发生什么。

循环引用可以包含任何数量的对象。一个坏的连接会导致很多环的时候，这就复杂了。

![](http://7i7i81.com1.z0.glb.clouddn.com/blogimage_memoryleak_3.png)

在环中，A→B是一个坏连接，创建了两个环：A-B-C-D 和 A-B-C-E。

这有两个问题：

1. 我们不想给一个坏连接导致的两个循环引用分别标记。
2. 我们不想给可能代表两个问题的两个循环引用一起标记，即使它们共享一个连接。

所以我们需要给循环引用定义簇组(clusters)，鉴于这些启发，我们写了个算法来找到这些问题。

1. 在给定的时间收集所有的环。
2. 对于每一个环，提取Facebook特定的类名。
3. 对于每一个环，找到包含在环内的被报告的最小的环。
4. 依据上面的最小环，将环添加到组中。
5. 只报告最小环。

最后一部分是找出谁第一时间偶然引入了循环引用。我们可以通过环中的”git/hg责任”的部分代码来猜测最近的变化所导致的问题。最后一个接触这个代码的人将会收到修复代码的任务。

整个系统如下:

![](http://7i7i81.com1.z0.glb.clouddn.com/blogimage_memoryleak_4.png)

## 手动性能分析

虽然自动化有助于简化发现循环引用的过程，降低人员的消耗，手动性能分析依然有它的用武之地。我们创建的另一个工具允许任何人查看内存使用，甚至不需要把他的手机插到电脑上。

[FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)可以很容易的添加到任何应用程序，可以让你手动配置构建文件，可以让你在应用程序内运行循环应用检测。它会借用[FBAllocationTracker](https://github.com/facebook/FBAllocationTracker)和[FBRetainCycleDetector](https://github.com/facebook/FBRetainCycleDetector)来实现此功能。

## ~~生成~~代(Generations)(感谢纠正翻译错误)

[FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)的一个很伟大的特性是“~~生成~~代追踪(generation tracking)”，类似于苹果的Instruments的~~生成~~代追踪。~~生成~~代只是简单的在两次标记之间拍摄所有仍然活着的对象的快照。

使用[FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)的界面,我们可以标记~~生成~~代，例如，分配三个对象。然后我们标记另一个~~生成~~代，之后继续分配对象。第一个~~生成~~代包含我们一开始的三个对象。如果任意一个对象被释放了，它会从我们第二个~~生成~~代中移除。

![](http://7i7i81.com1.z0.glb.clouddn.com/blogimage_memoryleak_5.png)

当我们有一个重复的任务，我们认为可能会内存泄露的时候，~~生成~~代追踪是很有用的，例如，导航View Controller的进出。在每次开始我们的任务的时候，我们标记一个~~生成~~代，然后，对之后的每个~~生成~~代进行调查。如果一个对象不应该活这么长时间，我们可以在[FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)界面清楚地看到。

## Check Out

无论你的应用程序是大是小，功能是多是少，好的工程师都应有好的内存管理。在这些工具的帮助之下，我们可以更简单的找到并修复这些内存泄露，所以我们可以花费更少的时间去手动处理，这样就可以有更多的时间去编写更好的代码。我们也希望你可以发现它们是有用的。在Github上check out下来吧。[FBRetainCycleDetector](https://github.com/facebook/FBRetainCycleDetector), [FBAllocationTracker](https://github.com/facebook/FBAllocationTracker) 和 [FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)。

## 备注

使用方式可以参考[FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)上的 Usage,或者也可以参考我的另外一篇博客:[FBMemoryProfiler 基础教程](http://ifujun.com/fbmemoryprofiler-shi-yong-ji-chu-jiao-cheng/)。

