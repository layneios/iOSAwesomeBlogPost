**原文链接：** [OCS——史上最疯狂的iOS动态化方案](http://www.jianshu.com/p/6c756ce76758)

**Keywords:** `OCS`，`iOS`，`动态化方案`

# OCS——史上最疯狂的iOS动态化方案

在iOS的发展历程上，涌现了很多动态化方案，有历史悠久的WaxPatch动态化方案，有远近闻名的JSPatch动态化方案。今天我们向大家介绍一款堪称“史上最疯狂”的iOS动态化方案——OCS.

# **初窥OCS**

OCS是全新设计的iOS动态化方案。我们定义了一套精确描述OC语义的字节码指令集(OCScript)，开发了一套全自动编译器（OCSCompiler），实现了一个高性能的虚拟机（OCSVM）以及一个可以跟底层无缝对接的桥接器(OCSBridge)。我们首先使用OCS编译器把OC源码转化成OCS字节码，然后通过OCS桥接器实现OCS虚拟机与Native运行时的互联，最后使用OCS虚拟机对OCS字节码进行解释运算，并驱动Native运行时完成逻辑的执行，以此达到Native代码动态化的效果。OCS被用于iOS APP安装包减包、功能插件化、HotPatch等方方面面动态化需求。

![](http://upload-images.jianshu.io/upload_images/4175145-76c157130ce69554?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


那么，我们为什么要实现OCS呢？

# OCS的发展背景

在手Q iPhone客户端的发展历史上，有两大众所周知的刚性技术需求一直是绕之不开的，一是安装包瘦身，二是打热补丁。为了进行减包和打热补丁，我们跟很多APP一样，先后使用WaxPatch和JSPatch这两大神器（本文后面对同类型的使用第三方脚本语言作为中间语言实现的动态化方案都简称为XPatch）进行了广泛的尝试，取得了很多成果，也暴露了不少问题：

**一、学习成本高。**

要进行补丁/插件编写，必须要学习并掌握对应的脚本语言，其次还需要学习补丁脚本的编写方式，并且必须熟悉各种约定和潜规则，否则，很容易掉到各种各样的坑里去。

**二、语义不一致。**

XPatch使用的脚本语言的语义与OC的语义天然不一致，所以机械地把Native代码转化成脚本，很容易引入未定义行为。由于缺乏高效的调试工具，Bug修复效率非常低下。

**三、内存管理不一致****。**

XPatch使用的解释引擎使用垃圾回收机制进行内存管理，而Native环境则使用MRC/ARC进行内存管理。两者并不兼容，很容易引入各种诡异的Crash。

**四、线程管理不一致****。**

XPatch使用的解释引擎本身只支持单线程，而Native运行环境则支持抢占式多线程。两者线程管理机制不可避免地相互冲突，非常容易引入隐秘的死锁和Crash风险。

**五、性能低下。**

XPatch对Native运行时进行消息发生的时候需要进行大量的字符串解析和数据转换操作，效率非常低下，很容易造成页面滑动卡顿和掉帧

![](http://upload-images.jianshu.io/upload_images/4175145-a60a9cfb74465cc9?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


OCS又是如何解决这些问题的呢？

# **四大一致性原则**

为了克服第三方脚本解释器动态方案存在的种种问题，我们从2015年下旬开始了OCS动态化方案的设计与实现。为了让上述问题得到有效解决， 我们提出了普适性的iOS动态化四大一致性原则：

**开发体验一致性；**

**线程管理一致性；**

**内存管理一致性；**

**语义定义一致性。**

![](http://upload-images.jianshu.io/upload_images/4175145-cfbecdbe864b3b05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这四大一致性原则贯穿于OCS整个设计实现过程。OCS从无到有，从只支持基本特性到功能日趋完善，从首发到迭代升级，OCS的架构从未发生重大更改。

OCS 1.0版本于2016年8月在手Q成功上线。到目前为止，OCS已经得到大规模代码转化与海量用户使用实战验证。从统计数据来看，我们的设计目标基本得到实现，传统脚本动态化方案存在的种种弊端几近得到完美的解决。

# OCS有哪些众不同之处

OCS是严格遵循四大一致性原则设计实现出来的，那么它有哪些鲜明的特点呢？

**一、语义与OC保持严格一致**

OCS字节码指令集语义与OC/的语义保持严格一一对应关系，支持的数据类型也完全等同，运行过程无需进行任何转换，因此效率非常高。

**二、自动化工具支持**

OCS拥有完善的自动化工具链，OCS转化器支持绝大多数OC/C语法的转化，包括C方法、任意自定义结构体、任意自定义block。对于少数不支持的的语法，可以准确报错，引导用户进行规避，避免产生未定义行为。OCS转化器还可以准确识别OC/C代码的内存管理语义，可以正确生成内存管理相关的代码。特别对于ARC来说，可以准确生成Retain、Release代码，正确插入到生成代码中去。

**三、原生OC内存管理机制**

OCS解释器完全支持Native的内存管理机制，可以在正确的时机执行正确的内存释放逻辑，绝无延迟操作，彻底杜绝了Crash和爆内存风险的引入。

**四、抢占式多线程**

OCS解释器支持Native的抢占式多线程线程管理支持，完全支持Pthread、 GCD、 NSOperationqueue、 NSThread等各种线程模型，完全避免了额外引入Crash、卡顿与死锁风险。

**五、高性能汇编ABI**

OCS桥接器根据过程调用约定实现自定义OCSABI，使得解释器与Native底层实现直接通信，保证Native代码动态化引入额外性能损耗最小化，使得OCS动态化不仅普遍适用于普通逻辑转化，对于对性能要求苛刻的有复杂排版的滑动列表，OCS也有极佳的表现，动态化后掉帧率增长量几乎可以忽略不计。

![](http://upload-images.jianshu.io/upload_images/4175145-09cff8d621951bfd?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


那么OCS框架的各个组成部分到底是怎样的呢？

OCS是一个四位一体的自研动态化方案，其中包含细节非常多。限于文章篇幅和读者的流量，我们下面仅对核心模块做简单的介绍。

# OCScript

OCScript与OC存在语义上的一一对应关系。跟OC一样，OCScript也是图灵完备性的语言。下面我们看看OCScript的指令集概况

![](http://upload-images.jianshu.io/upload_images/4175145-b461291b1390f664?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**基本运算指令**使得OCScript具有加、减、乘、除、移位等各种基本算术、逻辑运算能力。

**内存操作指令**使得OCScript具有在内存中的读写各种数据的能力；

**地址指跳转指令**使得OCScript具有分支判断与控制转移的能力；

**强类型转换指令**使得OCScript具有对数据类型进行强制转化能力；

**OC调用指令**使得OCScript具有指令级别的消息发生能力，根本性地提高了执行效率。

OCScript中每个指令编码占一个字节，所以储存空间非常紧凑。

# **OCSCompiler**

我们基于Clang构建了编译器

![](http://upload-images.jianshu.io/upload_images/4175145-89e3173840c9217a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在编译器中，我们根据AST来生成OCScript。举个例子

![](http://upload-images.jianshu.io/upload_images/4175145-e4caf7afab5c2a72?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图中左侧的源码使用Clang转成右侧的AST语法树。然后遍历每个AST节点，即可生成对应的OCScript字节码

![](http://upload-images.jianshu.io/upload_images/4175145-4a1fd7bc8a7ba15e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


它的具体含义可以根据对应的符号信息翻译得到

![](http://upload-images.jianshu.io/upload_images/4175145-0154c4391e9d2acc?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上面的汇编符号信息我们不难看出，OCScript的语义跟OC Native代码的语义完全一致。

OCScript字节码按如下文件格式进行存储

![](http://upload-images.jianshu.io/upload_images/4175145-1cde7b744954e5fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后由OCSVM加载执行。

# **OCSVM**

OCSVM是OCS动态化方案的核心，它不仅具备了解释执行OCScript的能力，它还承担了不可或缺的内存管理任务和线程管理任务

![](http://upload-images.jianshu.io/upload_images/4175145-03b2e34cb077b37b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**内存管理系统**

我们深入剖析了操作系统和OC运行时的内存管理相关的自动控制原理，构建了跟Native一致的内存管理机制，确保所有数据类型的内存得到妥善管理与正确释放。

**解释执行系统**

我们以图灵机作为计算模型，构造了具有强大通用计算能力、可以正确识别OC语义的专有计算系统

![](http://upload-images.jianshu.io/upload_images/4175145-70d14b12cf6d6b45?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**线程管理系统**

为了支持跟Native一致的支持抢占式多线程的线程管理机制，我们构造了多核心并发多任务执行框架

![](http://upload-images.jianshu.io/upload_images/4175145-3f23cc2331373a7f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


一个运算核心对应一个线程

![](http://upload-images.jianshu.io/upload_images/4175145-be46583701f97ea8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们经过精心设计的运算核心结构非常紧凑，对内存的占用量几乎可以忽略不计，就是有上百个线程在APP中运行，也表示毫无压力。

# OCSBridge

桥接器主要由三部分组成

![](http://upload-images.jianshu.io/upload_images/4175145-bdf14a8c6bd1d33a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


OCSBridge各个部分都基于OCSABI来构建，OCSABI使用汇编来实现，确保了解释器可以与Native最底层进行直接高速通信

![](http://upload-images.jianshu.io/upload_images/4175145-11ea15c3ea5071c9?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


值得一提的是，Block Bridge实现最复杂，它必须符合“OCS——Native互调完备性矩阵”

![](http://upload-images.jianshu.io/upload_images/4175145-ef6ae5382dbc4715?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


显然只有这样，才能实现任意自定义Block的双向互调。

# OCSEngine

拥有解释器和桥接器后，我们就可以构造出强大的动态化引擎，并植入到任意APP中去。

OCSEngine框架图如下

![](http://upload-images.jianshu.io/upload_images/4175145-24afe9bc0b1d34ee?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**配置系统**，解析动态化配置文件，使用OC Runtime时特性，为进程动态添加/替换OC 方法。

**运算系统**，职责由OCSVM承担。

**通信系统**，职责由OCSBridge承担。

OCSEngine启动流程如下

![](http://upload-images.jianshu.io/upload_images/4175145-384e38568f973e7f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


OCSEnging完成启动后，就可以和Native环境融为一体，流畅地完成各种逻辑执行。

# **OCS化Native代码流程**

只需要使用OCS转化宏把代码包裹起来，就可以由OCSCompiler自动完成转化流程。无论是几百行代码的迷你小模块，还是上万行代码的超级巨无霸类，均可一网打尽，便捷性不言而喻。

![](http://upload-images.jianshu.io/upload_images/4175145-15499c72a8a33fc7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 性能测试

OCS上线前，我们邀请了手Q性能专项测试团队进行对OCS进行了性能专项测试。

我们使用手Q挂件商城作为测试案例，主要原因如下：

1.挂件商城是XPatch插件实现的，并且保留了完整的Native代码。

2.挂件商城使用了布局非常复杂、内容丰富的的列表，这为测试滑动流畅度提供了绝佳的场景。

![](http://upload-images.jianshu.io/upload_images/4175145-c2f66a6ae7bb693e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


测试方法：分别针对不同实现分别编写不同测试包，使用相同的测试环境进行测试。

测试机器：iPhone 5s。

各项测试数据如下图表：

![](http://upload-images.jianshu.io/upload_images/4175145-c0a79aa1ac307f29?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/4175145-215468e271c33344?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/4175145-26c9d2051e4167ff?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/4175145-b0381547a0c5e71d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从以上各数据都不难看出，OCS性能跟Native相当接近。特别是对用户体验影响最大的滑动流畅度方面，OCS的表现几乎跟Native持平，事实上，我们从肉眼上已经难以分辨出页面到底是用Native实现还是OCS实现的了。

# 海量用户实战

OCS自9月份在手Q上线，目前已经已经有近20个业务、数万行代码使用OCS进行了动态化，安装包减量接近1M。所转业务日PV高达千万级别，并快速向亿级别挺进。同时，Crash率呈逐步下降趋势（OCS的本身的Bug不断得到收敛），最新线上版本全量OCS插件日Crash率（Crash总次数/打开总次数）为十万分之二。另外，尚未收到性能测试部门或外部用户遇到OCS业务出现卡顿、死锁相关反馈。

![](http://upload-images.jianshu.io/upload_images/4175145-e674a865454e46c3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/4175145-82450bdc94828dc8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


实战数据也表明，在四大一致性原则指导下设计实现的OCS动态化框架，其易用性、稳定性、和流畅性，都经得起大规模代码转化与海量用户验证的实战考验。

# **团队介绍**

我们OCS联合开发小组六位成员来自鹅厂各个部门，有白银峰、周刘纪、董超这样才华横溢的移动互联网新生代，又有殷诗壮、张恺、徐国欢这样混迹码界多年的老油条。然而年龄与部门距离并没有形成不可逾越的鸿沟，对技术创新的浓厚兴趣与对提升生产效率的本能饥渴使得我们并肩作战的决心从未动摇，我们潜心钻研攻克了OCS一个又一个的难题。

![](http://upload-images.jianshu.io/upload_images/4175145-2da0cc251d35586b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 常见问题与解答

问:OCSEngine的安装包大小如何？

答：OCSEngine编到APP中的安装包增量不到200k。对，你没看错，不到200k。

问：OCS为啥不用JS作为中间语言呢？

答：这是一个好问题，从图灵等价的角度来看，我们可以使用JS、Lua甚至甲骨文来作为中间语言，只要你喜欢就好。但经过理论验证，在我们看来，OCScript会是更好的选择。为什么呢？事实上我们可以使用计算理论和信息论来进行论证，但限于篇幅，我们就不在这里展开。后续我们会专门写一篇论文性质的文章来讨论这一课题，文章题目初步拟为《第三方脚本语言作中间语言的iOS动态化方案探索，从快速入门到果断放弃》，后续欢迎关注。

问:OCS有开放计划吗？

答：我们非常愿意与业界同行相互交流学习，一同促进iOS动态化方案的进步。对于方案原理，我们已经在实践开放计划，而对于具体工程实现开放，由于受到法律约束，目前还没有明确时间表，但我们也在积极推动之中。

# **结语**

曾经有人说过：如果你问一个程序员此时此刻正在在干什么，他一定会毫不犹豫地告诉你，他不是在改Bug，就是在去改Bug的路上。

Bug是程序员幸福生活的头号杀手。让我们感到欣慰的是，OCS从上线到现在，都没有收到OCS框架本身给使用OCS进行动态化的业务引入重大Bug的反馈（零星引入Bug的反馈基本是由于框架实现过程中手抖引入的）。

OCS不仅是一种崭新的“拿起就用”的动态化方案，更是成一种“用完就走”的可依赖的开发方式。通过OCS，我们捍卫了程序员作为普通劳动者应有的合法权利：每一个人都可以更加高效地完成动态化工作，并在忙碌了一天后，可以放下工作上的纷扰，花多点时间陪陪身边的伴侣和家人，并得到充分的休息，保证第二天可以满血复原，继续着改变世界的工作。

在本文的最后，我们衷心对支持我们的老板们和信任我们的合作伙伴们的表示感谢。也愿OCS可以早日给业界带来力所能及的贡献。

文／OCScript（简书作者）
原文链接：http://www.jianshu.com/p/6c756ce76758
著作权归作者所有，转载请联系作者获得授权，并标注“简书作者”。



