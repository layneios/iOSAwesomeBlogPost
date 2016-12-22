**原文链接：** [Xcode 8 Instruments 学习（五）](http://www.jianshu.com/p/0783cb5e1a46)

**Keywords:** `iOS`，`Xcode`，`Instruments`

# Xcode 8 Instruments 学习（五）

##### 使用Instruments的 Zombies 检测僵尸对象

> Instruments为我们提供了一个检测僵尸对象的工具：Zombies。
> 使用这个工具时，将会自动开启Enable Zombie Objects模式，而不需要我们自己手动去设置。
> 如图一 所示，我们可以看到”Zombies” （zɒmbi）这个工具。基本操作和其它工具一样，启动后点击工具栏上的红色按钮来启动程序。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-6f0d1a1921e5897b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图一.png
> 
> 
> 
> 在程序运行期间，如果定位到僵尸对象，则会弹出一个提示对话框，如图二所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-500273cd4810c95e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图二.png
> 
> 
> 
> 我们可以点击对话框右侧的箭头来定位到具体的代码及调用栈，图三所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-271ddeb841f8212d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图三.png
> 
> 
> 
> 双击调用栈对应的方法后，还可以查看具体的代码，如图四所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-921cc6b3940eb6ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图四.png
> 
> 

//==================================

> Xcode的 Debug navigator中提供了几个计量器来帮助我们跟踪程序的性能，包括CPU、内存、电量等。如图5和6所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-a17ff062ead51f90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图5.png
> 
> 
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-fe743c37ca2d9b7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图6.png
> 
> 
> 
> 在每个计量器的详情面板中的右上角，都提供了一个Profile in Instruments按钮，如图6所示(Energy Impact除外，其在面板详情中有几个按钮直接打开Instruments指定的模板，如图7所示 )
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-b5d9f9257ddf24f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图7.png
> 
> 
> 
> 这些按钮可以让我们直接跳转到Instruments中。
> 在点击这些按钮时，会弹出一个提示框，提示“Transfer current debug session?”，下面三个按钮，如图8所示。
> 
> ![](http://upload-images.jianshu.io/upload_images/1738027-1d503ebcef32cf2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 
> 图8.png
> 
> 
> 
> Transfer会在程序当前的运行状态中直接切换到Instruments，然后继续跟踪程序的运行状态；而Restart则是关闭当前运行的程序，重新开始一次新的Profile。 不过，这两种情况都会关闭当前的性能分析(profiling)，启动Instruments，初始一个新的性能分析。

