**原文链接：** [Xcode 8 Instruments 学习（一）](http://www.jianshu.com/p/92cd90e65d4c)

**Keywords:** `iOS`，`Xcode`，`Instruments`

# Xcode 8 Instruments 学习（一）

最近的几天在看一些Instruments想关的知识，总结分享一下希望对大家有所帮助：

###### 本文章主要介绍的是Instruments的相关知识，以及 一、使用Instruments的 Leaks 工具。 Instruments 其它的工具会在下篇文章一一介绍.

##### 前言介绍：

1.或许很多朋友对Instruments应用不太了解，但可能很多老的iOS开发者都应该用过Instruments工具来检测iOS应用内存泄漏情况。特别 是在iOS 5.0之前，即苹果在iOS平台上面还没支持ARC的时候，写iOS应用就类似C语言那样，容易忘记释放内存，而内存对移动设备而言是非常可贵的。即使目 前iPhone设备内存已经基本都满足512MB了，但是因为苹果的后台模式是把整个应用封 装起来等待下次启用，所以该应用所占用的内存同样被占据了。也就是即使应用进入后台模式，它还是仍然占用原先的内存的，所以你打开的应用越多，内存耗用自 然也很多。对很多普通用户而言，往往他们打开的应用都是进入后台模式的，很少有用户清理后台的应用，所以也就造成很多应用其实可用内存还是非常有限地 （题外话：如果苹果原生支持一键清理后台程序就好了，貌似越狱的工具里面有这样的支持的）。
2.还有做过iOS应用自动化测试的开发者，应该对UIAutomation很熟悉吧。它就是通过JS脚本来写界面自动化测试用例。而Instruments应用对UIAutomation支持很完善，你可以通过它查看很多代码潜在的问题，并测试性能。
3.其实Instruments应用还有很多强大的功能，它原生支持很多instrument工具，帮助你分析你的代码，不仅包括内存检测和自动化测试，它还可以监测文件读写操作等等待。所以一个好的iOS开发者是应该掌握Instrument应用的使用。因为Instruments应用本身功能太强大的，所以完全掌握机会不可能，但是因为它们内置的很多工具具有相似性，所以基本掌握自己常用的即可。同时了解一下内部有哪些功能，这样在你需要用到的时候再查查文档，就可以很快上手了。
（ 以上段落来源： [http://www.cnblogs.com/theManOfGod/p/4609992.html](http://www.cnblogs.com/theManOfGod/p/4609992.html) ）

##### 正文 Instruments的介绍：

> Instruments 一个很灵活的、强大的工具，是性能分析、动态跟踪 和分析OS X以及iOS代码的测试工具，用它可以极为方便收集关于一个或多个系统进程的性能和行为的数据，并能及时随着时间跟踪而产生的数据，并检查所收集的数据，还可以广泛收集不同类型的数据.也可以追踪程序运行的过程，这样instrument就可以帮助我们了解用户的应用程序和操作系统的行为。
> 总结一下instrument能做的事情：
> 1.Instruments是用于动态调追踪和分析OS X和iOS的代码的性能分析和测试工具；
> 2.Instruments支持多线程的调试；
> 3.可以用Instruments去录制和回放，图形用户界面的操作过程
> 4.可将录制的图形界面操作和Instruments保存为模板，供以后访问使用。
> instrument还可以：
> 1.追踪代码中的（甚至是那些难以复制的）问题；
> 2.分析程序的性能；
> 3.实现程序的自动化测试；
> 4.部分实现程序的压力测试；
> 5.执行系统级别的通用问题追踪调试；
> 6.使你对程序的内部运行过程更加了解。
> （以上部分来源 [http://www.jianshu.com/p/f87455e47da1](http://www.jianshu.com/p/f87455e47da1) ）

###### instrument模板虽多，但常用的就那几个（这里只介绍前几个的操作）：

> Leaks（泄漏）：一般的查看内存使用情况，检查泄漏的内存，并提供了所有活动的分配和泄漏模块的类对象分配统计信息以及内存地址历史记录；
> Time Profiler（时间探查）：执行对系统的CPU上运行的进程低负载时间为基础采样。
> Allocations（内存分配）：跟踪过程的匿名虚拟内存和堆的对象提供类名和可选保留/释放历史；
> Activity Monitor（活动监视器）：显示器处理的CPU、内存和网络使用情况统计；
> Blank（空模板）：创建一个空的模板，可以从Library库中添加其他模板；
> Automation（自动化）：这个模板执行它模拟用户界面交互为IOS机应用从instrument启动的脚本；
> Core Data：监测读取、缓存未命中、保存等操作，能直观显示是否保存次数远超实际需要。
> Cocoa Layout：观察约束变化，找出布局代码的问题所在。
> Network：跟踪 TCP / IP和 UDP / IP 连接。
> Automations：创建和编辑测试脚本来自动化 iOS 应用的用户界面测试。

```
 //-------------------------------------------------------
 Instruments最常用的三大类（主要介绍下面这三个的操作）：
 Leaks：找到引发内存泄漏的起点
 Time Profiler：分析代码的执行时间，找出导致程序变慢的原因。
 Allocations：监测内存使用/分配情况
 //----------------------------------------------------
```

> 迅速膨胀的内存可以很快让程序毙命，所以要多加防范。即使有 ARC（自动引用计数）内存管理机制，但在现实中对象之间引用复杂，循环引用导致的内存泄漏仍然难以避免，所以关键时刻还要自力更生。
> 
> 一、使用Instruments的 Leaks 工具（command + control + i）
> （ ˈɪnstrʊm(ə)nt 调试解决iOS内存泄漏的工具 liks 漏洞）
> 分析内存泄露不能把所有的内存泄露查出来，有的内存泄露是在运行时，用户操作时才产生的。那就需要用到Instruments的leaks了。

###### 操作步骤：

> 1.首先我们选中Xcode先把模拟器（command + R）运行起来
> 2.然后我们再选中Xcode，按快捷键（command + control + i）运行起来,
> 此时Leaks已经跑起来了，我们可以狠明显的看到，
> （或者你可以 点击Xcode的“调试导航”（如图一）
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-0677c1ac87892b08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图一.png
> 
> 
> 
> 然后选中“Memory”，再点击右侧的 “Profile in Instruments”，（如图二）
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-a04cdcb517db4a22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图二.png
> 
> 
> 
> 会自动打开Instruments。这时候会弹出来一个对话框，（如图三）
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-a59cee216ff8e080.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图三.png
> 
> 
> 
> 选择“Transfer” 这种方式打开）
> （在或者你可以通过Xcode --> Open Developer Tool -->instruments --leaks 的方式来打开）（如图0）
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-707b2b386c217c3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图0.png
> 
> 
> 
> (再或者你可以 按着control+空格键，输入instruments打开)

3.打开后，这时界面如图四：

![](http://upload-images.jianshu.io/upload_images/1738027-b1d4f152a304fbff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图四.png



4、由于Leaks是动态监测，所以我们需要手动操作APP，进行测试，一边操作APP，一边观察Leaks的变化，通过暂停右边的选择我们可以选择正在运行的程序,选中设备--app,之后点击红点Record（红色圆圈按钮）运行。
观察，我们可以发现在Leaks里面有一个红色X，这说明了我们的APP存在内存泄露。
点击暂停，点击其中一个，然后我们开始分析。（也可继续检测，当多个时暂停，一次处理了多个），
5.下面就是定位修改了,此时选中有红色叉的Leaks，下面有个"田"字方格，点开，选中Call Tree。
6.下面就是最关键的一步，在这个界面的右下角有若干选框，选中Invert Call Tree 和Hide System Libraries,（红圈范围内）（如果不知道在那个位置请接着往下看）
7.定位
在详情面板选中显示的若干条中的一条，双击，会自动跳到内存泄露代码处，然后点击右上角 Xcode 图标进行修改。

###### Leaks界面讲解:

> Leaks启动后会开始录制，随着对模拟器运行的App的操作，可以在Leaks中查看内存占用的情况。
> Leaks顶部分为两栏：Allocations(aləˈkeɪʃ(ə)n,分配 )和Leaks，右侧的曲线代表内存分配和内存泄漏曲线。
> 点击第二栏Leaks，（图七）
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-3a7d3dde631da84f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图七.png
> 
> 
> 
> 进行内存泄漏分析，右下角会出现Leaks调试的选项：
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-5ae2fc81c98a19ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 如图.png
> 
> 
> 
> 1、Record Settings (ˈrɛkɔːd 记录设置)
> 2、Display Settings 选项面板
> 3、Extended Detail 扩展面板（ɛkˈstɛnd,延展 diːteɪl,细节详情），在时间探查仪器的情况下，它是用来跟踪显示堆栈。
> 
> ##### 内存泄漏动态分析技巧：
> 
> 1.在Display Settings 界面建议把 Snapshot Interval （snapʃɒt, 数据快照）间隔时间设置为10秒，勾选Automatic Snapshotting，Leaks会自动进行内存捕捉分析。
> 2.熟练使用Leaks后会对内存泄漏判断更准确，在可能导致泄漏的操作里，在你怀疑有内存泄漏的操作前和操作后，可以点击Snapshot Now进行手动捕捉。
> 3.开始时如果设备性能较好，可以把自动捕捉间隔设置为5秒钟。
> 4.使用ARC的项目，一般内存泄漏都是malloc、自定义结构、资源引起的，多注意这些地方进行分析。
> 5.开启ARC后，内存泄漏的原因
> 开启了ARC并不是就不会存在内存问题，苹果有句名言：ARC is only for NSObject。
> 注：如果你的项目使用了ARC，随着你的操作，不断开启或关闭视图，内存可能持续上升，但这不一定表示存在内存泄漏，ARC释放的时机是不固定的。
> 
> ###### 做一下演示:
> 
> 1.运行项目，切换到iOS模拟器，点击那个测试按钮多点几次“button”，
> (这里我我选用的是MRC,在Main.storyboard里拖了一个button 并且为button的“touchUpInset”事件绑定buttonClick：事件处理方法，
> 代码如下：
> 
> ```
>  - (IBAction)buttonClick:(id)sender {
>  People * people = [[People alloc]init];
>  [people retain];
>  people.str = @"1324567";
>  } ）
> ```
> 
> 2.切换到Instruments会发现 如果没有泄漏 如图五
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-60ec3c71138dff25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图五.png
> 
> 
> 
> 当然这里是泄漏，在“Leaks”一栏里有红色的 X。如图六这就是内存泄露了。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-79ebf58b423a8f8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图六.png
> 
> 
> 
> 3.点击暂停，然后点击“Leaks”一栏
> 4.然后点击“导航栏”切换到“call tree”模式下(图八)
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-e85f11eb3e78141b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图八.png
> 
> 
> 
> 5.看到列表里列出了内存泄露的调用逻辑：(图九)
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-a0fb31a7db8e510c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图九.png
> 
> 
> 
> 10.勾选右边的详细窗口 Display Settings中的 Call Tree中 Separate by Thread和 Hide System Libraries两个选项,Hide System Libraries作用是隐藏系统函数。如果不点击 这里显示的是执行代码完整路径，其中系统和应用本身一些调用路径完全揉捏在一起.完全看不到我们关心的应用程序中实际代码执行耗时和代码路径实际所在位置 （如图十）
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-ced2cf5fe4856a13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十.png
> 
> 
> 
> （不勾选图十一）
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-1b6c1575a4791919.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十一.png
> 
> 
> 
> (勾选图十二）
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-8255408274433d1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十二.png
> 
> 
> 
> 11.勾选之后，双击一下就会来到内存泄漏的地方如图十三
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-781d3ca01398c22c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十三.png
> 
> 
> 
> 这里对右侧Display Settings中的Call tree选项有必要做一下说明[官方user guide翻译]:
> Separate By Thread:线程分离,只有这样才能在调用路径中能够清晰看到占用CPU最大的线程.每个线程应该分开考虑。只有这样你才能揪出那些大量占用CPU的"重"线程，按线程分开做分析，这样更容易揪出那些吃资源的问题线程。特别是对于主线程，它要处理和渲染所有的接口数据，一旦受到阻塞，程序必然卡顿或停止响应。
> Invert Call Tree:从上到下跟踪堆栈信息.这个选项可以快捷的看到方法调用路径最深方法占用CPU耗时（这意味着你看到的表中的方法,将已从第0帧开始取样,这通常你是想要的,只有这样你才能看到CPU中话费时间最深的方法）,比如FuncA{FunB{FunC}},勾选后堆栈以C->B->A把调用层级最深的C显示最外面.反向输出调用树。把调用层级最深的方法显示在最上面，更容易找到最耗时的操作。
> 
> Hide Missing Symbols:如果dSYM无法找到你的APP或者调用系统框架的话，那么表中将看到调用方法名只能看到16进制的数值,勾选这个选项则可以隐藏这些符号，便于简化分析数据.
> Hide System Libraries:表示隐藏系统的函数，调用这个就更有用了,勾选后耗时调用路径只会显示app耗时的代码,性能分析普遍我们都比较关系自己代码的耗时而不是系统的.基本是必选项.注意有些代码耗时也会纳入系统层级，可以进行勾选前后前后对执行路径进行比对会非常有用.因为通常你只关心cpu花在自己代码上的时间不是系统上的，隐藏系统库文件。过滤掉各种系统调用，只显示自己的代码调用。隐藏缺失符号。如果 dSYM 文件或其他系统架构缺失，列表中会出现很多奇怪的十六进制的数值，用此选项把这些干扰元素屏蔽掉，让列表回归清爽。
> Show Obj-C Only: 只显示oc代码 ,如果你的程序是像OpenGl这样的程序,不要勾选侧向因为他有可能是C++的
> Flatten Recursion: 递归函数, 每个堆栈跟踪一个条目，拼合递归。将同一递归函数产生的多条堆栈（因为递归函数会调用自己）合并为一条。
> Top Functions:找到最耗时的函数或方法。 一个函数花费的时间直接在该函数中的总和，以及在函数调用该函数所花费的时间的总时间。因此，如果函数A调用B，那么A的时间报告在A花费的时间加上B.花费的时间,这非常真有用，因为它可以让你每次下到调用堆栈时挑最大的时间数字，归零在你最耗时的方法。
> 需要添加其他工具的话：如图十六
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-fd2a96d211804bf4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十六.jpg
> 
> 
> 
> 关于界面的一些其他的补充：如 图十四
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-b86bc8e435fc2389.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十四.png
> 
> 
> 
> 左边的Show the CPU Data 可以查看每个CPU的消耗情况
> 中间的Show the Instruments Data 显示整体的消耗情况
> 右边的Show the Thread Data 可以查看每个线程对CPU的消耗情况
> 
> 图十五
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-a125396d5302b0de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十五.png
> 
> 
> 
> 选择 Detail -> Call Tree
> 表示查看整个调用过程有了上面的基础知识就可以对App的CPU消耗情况进行实时检测了。



