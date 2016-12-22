**原文链接：** [Xcode 8 Instruments 学习（四）](http://www.jianshu.com/p/ca6e25bf4604)

**Keywords:** `iOS`，`Xcode`，`Instruments`

# Xcode 8 Instruments 学习（四）

##### 这边文章不是讲Instruments 的，另外两种检测内存泄漏的方法

> 内存泄漏。其实有两种泄漏。
> 第一个是真正的内存泄漏，一个对象尚未被释放，但是不再被引用的了。因此，存储器不能被重新使用。
> 第二类泄漏是比较麻烦一些。这就是所谓的“无界内存增长”。这发生在内存继续分配，并永远不会有机会被释放。
> 如果永远这样下去你的程序占用的内存会无限大,当超过一定内存的话 会被系统的看门狗给kill掉.
> 
> ##### 另外两种检测内存泄漏的方法
> 
> ###### 第一种
> 
> NSZombieEnabled设置的使用 （zɒmbi 僵尸 ）
> 虽然iOS 5.0版本之后加入了ARC机制，由于相互引用关系比较复杂时，内存泄露还是可能存在。所以了解原理很重要。如果知道crash的地方了，但是不知道具体crash的原因应该怎么办？
> 又或者当遇到EXC_BAD_ACCESS错误的时候，该怎么处理？
> （设置 Enable Zombie Objects参数，在可执行选项，这有时候有助于缩小问题原因。
> 
> ###### 具体设置方法是：点击Product--> Scheme -->Edit Scheme 在Run选项的Diagnositics中设置Enable Zombie Objects。然后Close。再次运行，可能会出现一些问题提示。）
> 
> 1、运行Demo。
> 先实现准备好的内存泄露的Demo：leak app
> 打开运行，崩溃截图：如图五
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-9612b4916a02a548.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图五.png
> 
> 
> 
> 2.解决方案: 设置NSZombieEnabled
> 这是一个 “EXC_BAD_ACCESS”错误。我们打开XCode的选项：“NSZombieEnabled” 。在crash时可能会给你更多的一些提示信息。如图四 选择Edit Scheme 会出现一个弹窗如图六操作 勾选 Zomble Objects
> 3.再次运行，再次crash，这次在output窗口会看到多了一项错误信息： 2016-11-05 16:51:27.916 XXX [27377:981617] *** -[People setStr:]: message sent to deallocated instance 0x60800000c380
> 大概意思是：向已释放的内存发送消息。也就是说使用了已释放的内存，在C语言相当于使用了“野指针”

##### 第二种

> 静态内存分析--> Analyze
> 分析到哪里有内存泄露 ( an(ə)lʌɪz 爱ne来z 对...分析）
> 1.不运行程序， app没有了Crash,直接对代码进行内存分析，查看一下代码是否有内存泄露
> 优点：分析速度快，并且可以对所有的代码进行内存分析
> 缺点：分析结果不一定准确（没有运行程序，根据代码的上下文语法结构）
> 2.注意：如果有提示有内存泄露，一定结合代码查看代码是否有问题
> 操作步骤:
> 1、Analyze是静态分析工具 可以通过菜单 Product→Analyze启动
> 或者
> 2、(shift+command+b) 图八
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-fbe2677494dbe9be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图八.png
> 
> 
> 
> 效果如：图七
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-c51a88ef95acab7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图七.png
> 
> 
> 
> 点击 图九 里的蓝色按钮
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-2160ecf2c3cefec8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图九.png
> 
> 
> 
> 显示如图十 所示
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-24267e532b24bd9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十.png
> 
> 
> 
> Analyze-xcode编辑和解析工具
> iOS的分析工具可以发现编译中的warning，内存泄漏隐患，甚至还可以检查出logic上的问题；所以在自测阶段一定要解决Analyze发现的问题，可以避免出现严重的bug；
> 内存泄漏隐患提示：Potential Leak of an object allocated on line ……
> 数据赋值隐患提示：The left operand of …… is a garbage value;
> 对象引用隐患提示：Reference-Counted object is used after it is released;

###### 动态内存分析(Profile == Instruments )

> 真正运行程序，对程序进行内存分析（查看内存分配情况、内存泄露）
> 优点：分析非常准确，如果发现有提示内存泄露，基本可以断定代码问题
> 缺点：分析效率低（真正运行了一段代码，才能对该代码进行内存分析）
> 注意点：如果发现有内存泄露，基本需要修改代码（基本有内泄露）
> 操作步骤: Product -->Profile-->Allocations
> 二.内存使用注意
> 1.加载小图片\使用频率比较高的图片
> 1> 利用imageNamed:方法加载过的图片, 永远有缓存, 这个缓存是由系统管理的, 无法通过代码销毁缓存
> 2.加载大图片\使用频率比较低的图片(一次性的图片, 比如版本新特性的图片)
> 1> 利用
> initWithContentsOfFile:\imageWithContentsOfFile:\imageWithData:等方法加载过的图片, 没有缓存, 只要用完了, 就会自动销毁
> 2> 基本上, 除imageNamed:方法以外, 其他加载图片的方式, 都没有缓存
> 三.2个专业术语
> 1.内存泄漏
> 1> 该释放的对象, 没有被释放(已经不再使用的对象, 没有被释放)
> 2.内存溢出(Out Of Memory)
> 1> 内存不够用了
> 2> 数据长度比较小的数据类型 存储了 数据长度比较大的数据
> 四.图片在沙盒中的存在形式(dɪˈplɔɪm(ə)nt,部署)
> 1.如果项目的Deployment Target  1> 所有图片直接暴露在沙盒的资源包(main Bundle), 不会压缩到Assets.car文件
> 2.如果项目的Deployment Target >= 7.x (支持图片压缩)
> 1> 放在Images.xcassets里面的所有图片会压缩到Assets.car文件, 不会直接暴露在沙盒的资源包(main Bundle)
> 2> 没有放在Images.xcassets里面的所有图片会直接暴露在沙盒的资源包(main Bundle), 不会压缩到Assets.car文件
> 3.总结
> 1> 会压缩到Assets.car文件, 没有直接暴露在沙盒的资源包(main Bundle)
> 
> * 条件 : "Deployment Target >= 7.x" 并且是 "放在Images.xcassets里面的所有图片"
> * 影响 : 无法得到图片的全路径, 只能通过图片名(imageNamed:方法)来加载图片, 永远会有缓存
> 
> 2> 不会压缩到Assets.car文件, 直接暴露在沙盒的资源包(main Bundle)
> 
> * 条件 : 除1> 以外的所有情况
> * 影响 : 可以得到图片的全路径, 可以通过全路径(imageWithContentsOfFile:方法)来加载图片, 不会有缓存
> 
> 4.结论
> 1> 小图片\使用频率比较高的图片
> 
> ```
>  * 放在Images.xcassets里面
>  2> 大图片\使用频率比较低的图片(一次性的图片, 比如版本新特性的图片)
>  * 不要放在Images.xcassets里面
> ```
> 
> 五.设备信息相关的开发(非私有API, 底层API)
> 1.设备的型号
> 2.设备的CPU型号\使用情况
> 3.设备的内存容量\使用情况
> 4.设备的硬盘容量\使用情况
> 5.......
> 6.推荐的第三方库
> uidevice-extension
> 
> ```
>  * 地址 : https://github.com/erica/uidevice-extension
>  * 实现思路 : 利用分类给UIDevice进行了扩展
>  * 使用难易度 : 非常简单
> ```

```
 六.如何让程序尽量减少内存泄漏
 1.非ARC
 * Foundation对象(OC对象) : 只要方法中包含了alloc\new\copy\mutableCopy\retain等关键字, 那么这些方法产生的对象, 就必须在不再使用的时候调用1次release或者1次autorelease
 * CoreFoundation对象(C对象) : 只要函数中包含了create\new\copy\retain等关键字, 那么这些方法产生的对象, 就必须在不再使用的时候调用1次CFRelease或者其他release函数

 2.ARC(只自动管理OC对象, 不会自动管理C语言对象)
 * CoreFoundation对象(C对象) : 只要函数中包含了create\new\copy\retain等关键字, 那么这些方法产生的对象, 就必须在不再使用的时候调用1次CFRelease或者其他release函数

 3.block的注意
 // block的内存默认在栈里面(系统自动管理)
 void (^test)() = ^{

 };
 // 如果对block进行了Copy操作, block的内存会迁移到堆里面(需要通过代码管理内存)
 Block_copy(test);
 // 在不需要使用block的时候, 应该做1次release操作
 Block_release(test);
 [test release];
```

