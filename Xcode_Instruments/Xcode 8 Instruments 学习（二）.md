**原文链接：** [Xcode 8 Instruments 学习（二）](http://www.jianshu.com/p/9ac281228de2)

**Keywords:** `iOS`，`Xcode`，`Instruments`

# Xcode 8 Instruments 学习（二）

###### 这篇文章主要介绍使用Instruments的 Time Profiler 的使用

###### 前言

> 1.很多公司都恨不得把app压法周期压缩到最低,这就导致了开发中隐藏了很多问题,有点经验的工程师草率的优化下,更糟的情况那些没有经验的工程师甚至不会对app进行任何优化.
> 2.某种程度上来说,你开发过程中是可以忽略性能优化的. 十年前,移动设备的硬件资源是非常有限的.甚至连浮点数都是被禁止的.因为浮点数能导致代码变大计算的速度变慢.
> 3.科技发展如此迅速的今天,硬件很大程度上可以弥补软件的短板.现在的移动设备3D硬件处理的效率甚至媲美于PC机了,但是你不能总依赖于硬件和处理器速度来掩饰你APP做的多垃圾吧.(如果安卓系统跑在Iphone上还能够像iOS一样顺滑吗?其实是一个道理的)
> 4.性能这个概念很抽象,所以我们必须借助数据化图形化的输出方式.你可能花一周的时间去优化一个有趣的算法,但是这算法只占总执行时间的0.5%,不管你花多少精力去优化它,没人会注意到.相反一个for循环花费了90%的时间,你稍微修改下就能提高10%的效率,就是这个简单的修改可以得到大家很大的好感.因为.他们运行app时的第一感受就是比之前快了很多.没人会care你修改的是一个多牛逼的算法,还是一个简单的for循环.
> 这个说明了什么？与其花费时间在优化小细节上不如多点时间找到你改优化的地方.

###### 正文

###### 二、使用Instruments的 Time Profiler 工具（ˈprəʊfʌɪlə 脯肉fai乐 分析工具）

> * Time Profiler帮助我们分析代码的执行时间，找出导致程序变慢的原因，告诉我们“时间都去哪儿了？”。
> * Time Profiler分析原理：它按照固定的时间间隔来跟踪每一个线程的堆栈信息，通过统计比较时间间隔之间的堆栈状态，来推算某个方法执行了多久，并获得一个近似值。其实从根本上来说与我们的原始分析方法异曲同工，只不过其将各个方法消耗的时间统计起来。

下面讲解两种调试方式，一般来说，我们会用前者调试CPU的运行状况
第一种调试方式：使用Instruments的 Leaks 工具（这里不做讲解，请看我的第一篇文章）
第二种调试方式：Time Profiler 时间分析工具 用来检测应用CPU的使用情况.可以看到应用程序中各个方法正在消耗CPU时间.使用大量CPU不一定是个问题.类似我们客户端中不同场景的天气,动画就对CPU依赖就非常高，动画本身也是非常苛刻且耗费资源较多的任务.

> 下面引出“时间事件查看器”(Time Profiler),———他可以测量时间的间隔,中断程序执行,跟踪每个线程的堆栈.
> 操作步骤:
> 1.工具通过Xcode工具栏中Product->Profile可以启动,启动后界面如下:图一
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-acb1e5c573b1e2cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图一.png
> 
> 
> 
> 2.选择Time Profiler启动.弹出如 (图二)窗口
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-d2a7496c3c33b9e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图二.png
> 
> 
> 
> 当点击Time Profiler应用程序开始运行后. 就能获取到整个应用程序运行 消耗时间分布 和 百分比.
> 研究下面的截图和它下面的每个部分的解释：如图十一：
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-c5e87f054cdbf796.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十一.png
> 
> 

###### 下面是讲解：

> 1.这里控制记录过程，点击红色的"记录"按钮可以停止或开始当前正在分析的app（在记录和停止按钮之间切换），暂停键，如你所想，暂停当前正在运行的app。注意这实际上是停止和启动应用程序,而不是暂停它.
> 2.这里是执行计时器（run timer）,运行定时器和运行导航,定时器显示APP已经运行了多长时间,执行了多少次。箭头之间是可以移动的。如果停止，然后使用录制按钮重新启动应用程序，这将开始一个新的运行。显示屏便会显示“run2 of 2”，你可以回到第一次运行的数据，首先你停止当前运行，然后按下左箭头回去
> 3.运行轨道。这里被称作路径（track），就你选择的Time Profiler工具而言，因为只有一个工具，所以这里只有一条路径，关于这里显示的图标的详情，一会你就会在接下来的教程中了解更多。
> 4.扩展面板，在时间探查仪器的情况下，它是用来跟踪显示堆栈。
> 5.详细地面板。它显示了你正在使用的仪器的主要信息,这是使用频率最高的部门,可以从它这里看到cpu运行的时间.这里是详情面板，展示的是你正在使用的工具的主要信息。就现在而言，这里展示的是最"笨重（hottest）"的方法--换句话说，占用CPU时间最长的方法。点击上方的bar会看到Call Tree（左手边的那个）并选中Sample List，然后你会看到数据的不同视图，视图展示了每一个示例。点击其中几个，你会在Extended Detail inspector中看到被捕获的堆栈跟踪。
> 6.选项面板，也叫检查器（inspector）面板，一共有三个检查器：record setting（记录设置），display setting（展示设置），还有extends detail（扩展详情）。（在上一篇文章里有详细介绍）。

###### 原始的性能分析方法

> 这种分析方法估计是很多开发人员第一时间想到的，写个单元测试，在开始和结束的地方记录时间。
> 示例代码：
> NSDate *startDate = [NSDate date];
> for(int i = 0; i  // do something
> }
> NSDate* endDate = [NSDate date];
> NSLog(@"time:%f", [endDate timeIntervalSinceDate: startDate]);
> 这种方法的缺点有以下几点:
> 1、测试效率太低，很多性能瓶颈是很难预估到的，需要从上层到下层进行逐步排除；
> 2、无法对界面渲染的效率进行测试，找出界面性能瓶颈；
> 3、NSLog的分析不够精确，可能在模拟器上由于开发设备性能速度快，无法明显区分出性能瓶颈。
> 二.使用Time Profiler前须知
> 1.当点击Time Profiler应用程序开始运行后.就能获取到整个应用程序运行消耗时间分布和百分比.为了保证数据分析在统一使用场景真实行有如下点需要注意:
> 在开始进行应用程序性能分析的时候,一定要使用真机,模拟器运行在Mac上，然而Mac上的CPU往往比iOS设备要快。相反，Mac上的GPU和iOS设备的完全不一样，模拟器不得已要在软件层面（CPU）模拟设备的GPU，这意味着GPU相关的操作在模拟器上运行的更慢，尤其是使用CAEAGLLayer来写一些OpenGL的代码时候. 这就导致模拟器性能数据和用户真机使用性能数据相去甚运.
> 2.另外在开始性能分析前另外一件重要的事情是，应用程序运行一定要发布配置 而不是调试配置.
> 在发布环境打包的时候，编译器会引入一系列提高性能的优化，例如去掉调试符号或者移除并重新组织代码.另iOS引入一种"Watch Dog"[看门狗]机制.不同的场景下，“看门狗”会监测应用的性能。如果超出了该场景所规定的运行时间，“看门狗”就会强制终结这个应用的进程.开发者可以crashlog看到对应的日志.但Xcode在调试配置下会禁用"Watch Dog".
> 三.time profiler的主界面(几个需要关注的重点区域)
> 1.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-def1166d02149e13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 1.png
> 
> 
> 
> 视图开关:分别为三个视图的开关，全部选上可达到主界面截图效果。
> 搜索条:如果您需要快速查找具体的类或函数，可在些输入类名或者函数名，会有意想不到的效果。
> 数据可视化面板:可在此设置您需要关注的范围，以便将不相干的内容过滤掉。
> 详情面板:在time profiler下主要是看Call Tree和Sample List这两种视图：
> 2.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-15b89985b24e2e9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 2.png
> 
> 
> 
> 调用树（Call Tree 3.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-7ea9f4e64c72d21b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 3.png
> 
> 
> 
> Running Time：函数运行的时间，这个时间是累积时间
> Self：在栈顶次数
> Symbol Name：被调用函数的符号信息
> 从详情面板Call Tree与相关内容扩展详情面板对应的关系图：
> 4.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-f190a1d6433b2929.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 4.png
> 
> 
> 
> 详情面板更多的信息选项
> 5.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-52ca0b4761147ba3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 5.png
> 
> 
> 
> 样本列表（Sample List）6.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-835e7b8269656f84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 6.png
> 
> 
> 
> Timestamp：采样的开始时间
> Dep：堆栈深度
> CPU：线程运行在那一个CPU上
> Process：进程名称
> Thread：所在的线程名称
> Hot Frame：采样中调用最多的函数
> Responsible Library：调用该函数的库
> Responsible Caller：调用该函数的函数
> 选项视图参数设置 7.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-4bf85b9ccd264775.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 7.png
> 
> 
> 
> Separate byt Thread（建议选择）：通过线程分类来查看那些纯种占用CPU最多。
> Invert Call Tree（不建议选择）：调用树倒返过来，将习惯性的从根向下一级一级的显示，如选上就会返过来从最底层调用向一级一级的显示。如果想要查看那个方法调用为最深时使用会更方便些。
> Hide Missing Symbols（建议选择）：隐藏丢失的符号，比如应用或者系统的dSYM文件找不到的话，在详情面板上是看不到方法名的，只能看一些读不明的十六进值，所以对我们来说是没有意义的，去掉了会使阅读更清楚些。
> Hide System Libraries（建议选择）：选上它只会展示与应用有关的符号信息，一般情况下我们只关心自己写的代码所需的耗时，而不关心系统库的CPU耗时。
> Flatten Recursion（一般不选）：选上它会将调用栈里递归函数作为一个入口。
> Top Functions（可选）：选上它会将最耗时的函数降序排列，而这种耗时是累加的，比如A调用了B，那么A的耗时数是会包含B的耗时数。
> 四.使用技巧
> 1.图标为黑色头像的就是Time Profiler给我们的提示，有可能存在性能瓶颈的地方
> 2.按着option键在主界面6中通过拖动鼠标来选择需要过滤的时间段
> 3.Command+F查找过滤的函数名或者类名
> 4.关联代码 8.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-d33dcbed36993632.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 8.png
> 
> 
> 
> 5.Call Tree Constraints过滤 9.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-c02a889178e90a94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 9.png
> 
> 
> 
> Count：设置调用的次数(调用2次以上的方法)
> Time (ms)：设置耗时范围(耗时20毫秒以上的方法)
> 筛选结果示例图
> 10.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-7caae333320b1885.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 10.png
> 
> 

