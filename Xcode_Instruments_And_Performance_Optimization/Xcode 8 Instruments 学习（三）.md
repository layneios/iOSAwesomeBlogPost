**原文链接：** [Xcode 8 Instruments 学习（三）](http://www.jianshu.com/p/b3443352169c)

**Keywords:** `iOS`，`Xcode`，`Instruments`

# Xcode 8 Instruments 学习（三）

###### 使用Instruments的 Allocations （aləˈkeɪʃ(ə)n,分配）工具

> Allocations 分配工具。它能给出你所有创建和存储它们的内存的详细信息，它也显示你保留了每个对象的计数。
> 
> * 我们经常需要从服务器下载大量图片，特别是开发照片类的应用。但往往稍不注意，内存使用就会暴增，所以得保证把这些图片缓存下来以便重复使用。重新载入图片时，内存都会出现使用峰值。应用先分配大量内存来替换原有图片，然后再释放掉这部分内存，可想而知这样的操作效率高不了，而且如果要下载更大的文件，呃，局面大概会失控吧。用异步方式来加载图片，这样用不着等到所有图片下载完才能在界面中显示。大多数图像缓存库都会把加载工作放到后台，以避免延长主线程的响应周期。

###### 言归正传 步骤：

> 1.Xcode和选择Product->Profile。
> 2.然后，选择Allocations启动
> 如下图十二，图十三，图十四
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-956cd560b3a00b6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十二.png
> 
> 
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-e2ad82b433590b8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十三.png
> 
> 
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-ba071adb0cb112f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十四.png
> 
> 
> 
> ###### 这个时候你会发现两个曲目。一个叫(分配)Allocations，以及一个被称为VM Tracker（ˈtrakə,追踪者）(虚拟机跟踪);虚拟机跟踪也是非常有用的，但更复杂一点。
> 
> * * *
> 
> 实际上，我们用Allocations工具也可以检测僵尸对象，如 图十五 所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-ee5d9ccbcc29aa39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 十五.png
> 
> 
> 
> 我们在属性面板中勾选”Enable NSZombie detection”，其效果和单独使用Zombies工具是一样的。

* * *

> Instruments访问多次运行的跟踪数据
> 
> ###### # Instruments在一次运行期间可以记录App的多次运行记录。以Allocations为例，开启Instruments后，每结束一次Allocations分析，这条分析就会被记录下来，下次再开启分析时，我们仍然可以看到前一次分析的信息，如图十六所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-2f17929c1158e613.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图十六.png
> 
> 
> 
> 通过这些记录，我们可以对比每次分析的差别。这样我们就可以边修改程序，边用Instruments来对其进行分析，并通过这种对比来观察修改的效果 >当然，关闭Instruments时，如果不保存信息，这些记录会被清理掉。

* * *

> 说到内存问题，我们更多的会想到内存泄露和野指针，而实际上还有一类看似不是问题的内存问题：Abandoned Memory(被遗弃的内存)。这类内存可能由于某些原因被分配，但并非一直需要，只是可能在程序运行期的某个时间需要，如内存缓存的图片，还有一个比较普遍的东西–单例。
> 我们可能会为某个模块创建一个单例对象来维护这个模块所需要的数据，但在退出模块后，这个单例对象依然存在。与内存泄露不同，这些对象从技术上讲依然是有效的。但实际上可能在程序后续的运行中不会再被使用。
> 使用Instruments定位内存问题，内存泄露和野指针的定位相对来说容易些，内存泄露使用Leaks，野指针则可以使用僵尸对象。而Abandoned Memory则相对不那么明显。Abandoned Memory可以采用所谓的Generational Analysis方法来分析，即反复进入退出某一场景，查看内存的分配与释放情况，以定位哪些对象是属于Abandoned Memory的范畴。
> 在Allocations工具中，有专门的Generational Analysis设置，如下图十七所示
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-84a7b3bee5f6a0f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 十七.png
> 
> 
> 
> 我们可以在程序运行时，在进入某个模块前标记一个Generation，这样会生成一个快照。然后进入、退出，再标记一个Generation，如下图十八所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-3b7abcea24b58be7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 十八.png
> 
> 
> 
> 在详情面板中我们可以看到两个Generation间内存的增长情况，其中就可能存在潜在的被遗弃的对象，如下图十九所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-46517d4fa9b81043.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 十九.png
> 
> 
> 
> 定位到问题，即可做相应的优化。
> 原文链接：[http://www.jianshu.com/p/c558806983cd](http://www.jianshu.com/p/c558806983cd)

###### 使用instrument测试内存泄露 工具 Allocations 测试是否内存泄露 使用标记，可以更省事省力的测试页面是否有内存泄露

> 1、设置Generations
> 11.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-f5f24bfce13d612a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 11.jpg
> 
> 
> 
> 2、选择mark generation
> 12.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-e95a20c27d9e9df1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 12.jpg
> 
> 
> 
> 3、使用方法 在进入测试页面之前，mark一下----->进入页面----->退出----->mark------>进入------->退出------->mark------>进入
> 如此往复5、6次，就可以看到如下结果 13.png
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-9aa79d1ad7985a0d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 13.jpg
> 
> 
> 
> 这种情况下是内存有泄露，看到每次的增量都是好几百K或者上M的，都是属于内存有泄露的，这时候就需要检测下代码
> 一般情况下，100K以下都属于正常范围，growth表示距离你上次mark的增量

