**原文链接：** [iOS 性能优化：Instruments 工具的救命三招](https://blog.leancloud.cn/2835/)

**Keywords:** `iOS`，`性能优化`，`Instruments`

# iOS 性能优化：Instruments 工具的救命三招

对于每位 iOS 开发者来说，代码性能是个避不开的话题。随着项目的扩大和功能的增多，没经过认真调试和优化的代码，要么任性地卡顿运行，要么低调地崩溃了之……结果呢，大家用着不高兴，开发者也不开心。

其实要破这个局面并不难，只要在 Xcode 自带的监控调试工具 Instruments 上花点功夫，让大代码流畅运行也不是神话。Instruments 提供了很多功能，我会重点介绍一下我最常用的三大类：

* Time Profiler：分析代码的执行时间，找出导致程序变慢的原因。
* Allocations：监测内存使用 / 分配情况
    迅速膨胀的内存可以很快让程序毙命，所以要多加防范。
* Leaks：找到引发内存泄漏的起点

即使有 ARC（自动引用计数）内存管理机制，但在现实中对象之间引用复杂，循环引用导致的内存泄漏仍然难以避免，所以关键时刻还要自力更生。

针对这三方面的测试，我写了个演示应用，放在 [GitHub](https://github.com/mcgraw/dojo-instruments) 上，来帮助大家更直观地了解这些工具的使用方法。好，进入正题。
[![001](https://blog.leancloud.cn/wp-content/uploads/2015/02/001-397x625.png)](https://blog.leancloud.cn/wp-content/uploads/2015/02/001.png)

## Time Profiler

时间都去哪儿啦？ Time Profiler 可以回答。它会按照设定的时间间隔（默认 1 毫秒）来跟踪每一线程的堆栈信息（stack trace），并通过比较时间间隔之间的堆栈状态，来推算出某个方法执行了多久，给出一个近似值。
在演示应用头一项「Time Profiler: System Methods」中，我用插入排序（Insertion Sort）和冒泡排序（Bubble Sort）两种算法来做性能比较，下面是 Swift 代码：

```
/* 引用自：http://waynewbishop.com/swift/sorting-algorithms/ */

func insertionSort() {
    var x, y, key: Int

    for (x = 0; x < numberList.count; x++) {
        key = numberList[x]

        for (y = x; y > -1; y--) {
            if key < numberList[y] {
                numberList.removeAtIndex(y + 1)
                numberList.insert(key, atIndex: y)
            }
        }
    }
}

func bubbleSort() {
    var x, y, z, passes, key : Int

    for (x = 0; x < numberList.count; ++x) {
        passes = (numberList.count - 1) - x;

        for (y = 0; y < passes; y++) {
            key = numberList[y]

            if (key > numberList[y + 1]) {
                z = numberList[y + 1]
             numberList[y + 1] = key
                numberList[y] = z
            }
        }
    }
}
```

这段代码主要是对数组的添加和删除，两种方法执行起来耗时不多，但后台发生的系统动作却多得让人眼晕。
[![004](https://blog.leancloud.cn/wp-content/uploads/2015/02/004-625x352.jpg)](https://blog.leancloud.cn/wp-content/uploads/2015/02/004.jpg)
可以发现，代码用到了很多间接依赖，这些都是支撑代码运行的系统库文件。因为处理大数据集比较消耗系统资源，所以要尽可能地把繁重的操作放到后台去做，上面的代码就走的后台线程。在上图的 Call Tree 中可以看到，被调用的堆栈名是 dispatch_worker_thread3。如果把它放到主线程去执行，程序肯定会挂起。不信你注释掉 dispatch_async 调用看一下。
再来个图片加载的例子。
[![003](https://blog.leancloud.cn/wp-content/uploads/2015/02/003-625x352.jpg)](https://blog.leancloud.cn/wp-content/uploads/2015/02/003.jpg)
这儿有三种图片加载方法：

* loadSlowImage1：从指定 URL 下载一张图片（加载速度慢）
* loadImage2：从本地资源库加载一张图片（注意：没用系统缓存）
* loadFastImage3：从系统缓存中加载一张图片（加载速度快）

我们来看看 Time Profiler 算出的结果是不是跟预想的一样。

进入演示应用第二项「Time Profiler: Our Methods」，点击「Reload」十次来重复加载图片，这样能产生足够的数据来分析。然后在 Time Profiler 图表中通过拖拉鼠标选中要放大查看的区域，从 Call Tree 中双击调用了 .reload 方法那一行（上图中加亮选中那一行），就会跳转到对应的代码行，所用时间也标注出来了。
[![004](https://blog.leancloud.cn/wp-content/uploads/2015/02/004-625x352.jpg)](https://blog.leancloud.cn/wp-content/uploads/2015/02/004.jpg)
看到谁最花时间了吧。虽然代码没什么可优化的地方，但大家应该认识到缓存能发挥的作用。所以即使有时还得调用 loadSlowImage，多数情况下把图片缓存下来，还是能省些资源占用。

此外，我想再说说 Call Tree 的选项设置。
[![005](https://blog.leancloud.cn/wp-content/uploads/2015/02/005.jpg)](https://blog.leancloud.cn/wp-content/uploads/2015/02/005.jpg)
这些选项默认是不选的，但把它们勾选上可以帮你更快定位到关键的代码上，往往这也是问题的源头。

* Separate by Thread：按线程分开做分析，这样更容易揪出那些吃资源的问题线程。特别是对于主线程，它要处理和渲染所有的接口数据，一旦受到阻塞，程序必然卡顿或停止响应。
* Invert Call Tree：反向输出调用树。把调用层级最深的方法显示在最上面，更容易找到最耗时的操作。
* Hide Missing Symbols：隐藏缺失符号。如果 dSYM 文件或其他系统架构缺失，列表中会出现很多奇怪的十六进制的数值，用此选项把这些干扰元素屏蔽掉，让列表回归清爽。
* Hide System Libraries：隐藏系统库文件。过滤掉各种系统调用，只显示自己的代码调用。
* Flattern Recursion：拼合递归。将同一递归函数产生的多条堆栈（因为递归函数会调用自己）合并为一条。
* Top Functions：找到最耗时的函数或方法。

需要添加其他工具的话：
[![007](https://blog.leancloud.cn/wp-content/uploads/2015/02/007.jpg)](https://blog.leancloud.cn/wp-content/uploads/2015/02/007.jpg)

## Allocations

我们经常需要从服务器下载大量图片，特别是开发照片类的应用。但往往稍不注意，内存使用就会暴增，所以得保证把这些图片缓存下来以便重复使用。下面来看看演示程序中内存分配的例子。
[![008](https://blog.leancloud.cn/wp-content/uploads/2015/02/008-625x403.jpg)](https://blog.leancloud.cn/wp-content/uploads/2015/02/008.jpg)
从图中可以看到，每次点击「Reload」重新载入图片时，内存都会出现使用峰值。应用先分配大量内存来替换原有图片，然后再释放掉这部分内存，可想而知这样的操作效率高不了，而且如果要下载更大的文件，呃，局面大概会失控吧。

看一下堆栈列表第四行，ImageIO_PNG_Data 里有 9 张处于活动状态的图片，占用了 12.38 MB 内存，这些都是没被系统释放或缓存的内存，所以导致堆内存分配升高。接下来再看看使用缓存后的效果。
[![009](https://blog.leancloud.cn/wp-content/uploads/2015/02/009-625x417.jpg)](https://blog.leancloud.cn/wp-content/uploads/2015/02/009.jpg)
使用了缓存库（[Swift](https://github.com/Haneke/HanekeSwift) [Haneke](https://github.com/Haneke/HanekeSwift)）后，点「Reload」五次，这回在 Allocations 列表中却看不到 ImageIO_PNG_Data 对象了，这说明它是空的，没有任何图像数据。同时，All Heap Allocations 的大小已从刚才的 14.61 MB 降到了 2.51 MB。Anonymous VM（匿名虚拟内存）是系统为程序预留的、可能会立即被重复使用的一部分可用内存。要防止程序崩溃，就别让堆的尺寸增长太快。

还有就是，例子用的是异步方式来加载图片，这样用不着等到所有图片下载完才能在界面中显示。大多数图像缓存库都会把加载工作放到后台，以避免延长主线程的响应周期。

## Leaks

尽管 Apple 推出的 ARC 可以有效防范内存泄漏，但出问题的机率还是会有，Swift 也不例外。鉴于篇幅有限，本文就不涉及内存和 ARC 的工作原理了，具体可以参考 [官方文档](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AutomaticReferenceCounting.html) 。我会用代码来触发内存泄漏。
首先从最底层上说，当两个对象相互建立了强引用（strong reference），当一个对象被释放，另一个对象由于是强引用的关系不允许被释放，此时 ARC 无法确定没被释放的对象到底还有没有用，于是就导致了内存泄漏。
[![010](https://blog.leancloud.cn/wp-content/uploads/2015/02/010-625x424.jpg)](https://blog.leancloud.cn/wp-content/uploads/2015/02/010.jpg)
要解决这个问题，可以将其中的一个对象中变量设为 weak，不让它出现在保留周期中。很多开发者在管理 view controller 时常会在内存泄漏上中招，以为换了新的 controller，老的 controller 就被释放回收了，其实还没。这样代码一多，就会造成很多对象都没被释放。所以用这个工具把整个应用跑一遍，把那些断链的强引用清理干净，会大有裨益。

除了上述这三类工具，Instruments 还有很多实用的工具，推荐大家根据自己的关注点，花些时间去学学。比如：

* Core Data：监测读取、缓存未命中、保存等操作，能直观显示是否保存次数远超实际需要。
* Cocoa Layout：观察约束变化，找出布局代码的问题所在。
* Network：跟踪 TCP / IP 和 UDP / IP 连接。
* Automations：创建和编辑测试脚本来自动化 iOS 应用的用户界面测试。

最后小总结下。我倒不想一味夸大 Instruments 的作用，如果应用跑得挺痛快，没出现啥调皮行为，大可把它忽略，等到问题来了再做优化。对于新手来说，花些时间了解 Instruments 的功能，多调试多积累经验，这样做出来的应用在用户体验上肯定错不了。

你最常用的 Instruments 工具都有哪些？欢迎与我们分享。

原文：[How To Use The 3 Instruments You Should Be Using](http://www.xmcgraw.com/how-to-use-the-3-instruments-you-should-be-using/)
引言：
你的 iOS 应用，运行速度靠谱吗？中枪的同学莫要愁，性能优化咱有妙招。用 Xcode 自家的调试工具 Instruments，揪出那些堵线程、占内存、耗资源的问题代码，彻底破掉迷局，让应用扬眉吐气喽~~

本条目发布于 [2015/02/27](https://blog.leancloud.cn/2835/ "17:30")。属于[技术分享](https://blog.leancloud.cn/category/%e6%8a%80%e6%9c%af%e5%88%86%e4%ba%ab/)分类。作者是[真·袁滚滚](https://blog.leancloud.cn/author/yyuan/ "查看所有由真·袁滚滚发布的文章")。

