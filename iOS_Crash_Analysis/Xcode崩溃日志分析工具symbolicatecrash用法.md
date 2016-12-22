**原文链接：** [Xcode崩溃日志分析工具symbolicatecrash用法](http://www.jianshu.com/p/e428501ff278)

**Keywords:** `iOS`，`symbolicatecrash`，`崩溃日志分析`

# Xcode崩溃日志分析工具symbolicatecrash用法

![老规矩](http://upload-images.jianshu.io/upload_images/134882-05033cc84eec1a2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/format/jpg)

## 什么是symbolicatecrash

* * *

symbolicatecrash是Xcode自带的一个分析工具，可以通过机器上的崩溃日志和应用的.dSYM文件定位发生崩溃的位置，把crash日志中的一堆地址替换成代码相应位置。

## 为什么要用symbolicatecrash

* * *

开发者调试错误只需要有真机，并
且连接到xcode上，就可以跟踪发现错了。
但是如果你的APP不是安装在你自己的真机上，比如你的APP发布到App Store(客户下载后你如何跟踪你的APP在他们的机器上？)这时候就要用到symbolicatecrash。
当一款APP软件在IOS设备上崩溃的时候，一份“crash report”将会自动创建并且存储在设备上。crash report描述了APP崩溃的日志。在大多数情况下，包括对每个线程执行一个完整的堆栈跟踪，查看该日志对于APP崩溃调试非常有用。

## 如何查看iphone上的崩溃日志

* * *

// ios8之前
设置
通用
关于本机
诊断与用量
诊断与用量数据

// iOS 8
设置
隐私
诊断与用量
诊断与用量数据

## 如何同步设备日志到我们的mac上

* * *

`如果是其他用户并且是APP Store下的APP`
需要用户在'如何查看iphone上的崩溃日志'中，将《自动发送》开启，打开《与应用开发者共享》，这样用户的APP崩溃后，会提示发送崩溃日志到开发者，开发者就可以在iTunes Connect中下载这些崩溃日志。
`如果是手中的真机`
直接将IPHONE连接到iTunes,打开xcode->window->devices，导出你需要的崩溃日志即可

![导入步骤](http://upload-images.jianshu.io/upload_images/134882-3eb075f8e82c2413.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/format/jpg)

## 如何使用symbolicatecrash分析崩溃日志

* * *

Step 1:在你的MAC桌面创建一个新文件夹，并且命名为"CrashReport"

Step 2:打开前往应用程序,找到 `Xcode` 应用程序, 右击它选中 "显示包内容" ,之后根据下面提供的路径

Xcode6.0之前:
"Contents->Developer->Platforms->iPhoneOS.platform->Developer->Library->PrivateFrameworks->DTDeviceKit.framework->Versions->A->Resources"

OR

"Contents->Developer->Platforms->iPhoneOS.platform->Developer->Library->PrivateFrameworks->DTDeviceKitBase.framework->Versions->A->Resources"

Xcode6.0之后
改成 "Contents/SharedFrameworks"

`实在找不到可以打开终端输入 find /Applications/Xcode.app -name symbolicatecrash -type f ，然后终端会返回这个文件的路径`

只要找到"symbolicatecrash" 文件, 复制然后粘贴到刚才创建的 "CrashReport" 文件夹里面.

Step 3: 从Xcode Archive的二进制文件中找到.dSYM文件和.app文件拷贝到刚才创建的 "CrashReport" 文件夹里面.

![1.png](http://upload-images.jianshu.io/upload_images/134882-3bb26303d5a629ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/format/jpg)

![2.png](http://upload-images.jianshu.io/upload_images/134882-49e6ab4169d61c3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/format/jpg)

![3.png](http://upload-images.jianshu.io/upload_images/134882-0b698eb092b96e3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/format/jpg)

Step 4:打开终端进入CrashReport文件夹，依次输入以下命令行:
`cd /Users/username/Desktop/CrashReport`

`export DEVELOPER_DIR=/Applications/XCode.app/Contents/Developer`

`./symbolicatecrash ./*.crash ./*.app.dSYM > symbol.crash`
这时候终端将会进行处理......
处理结果是生成一个新的文件symbol.crash。然后打开这个文件。
*你就会看到日志跟我们调试APP的控制台输出的内容一样了！*

![你应该看到的是这样的](http://upload-images.jianshu.io/upload_images/134882-faceb6923c35d5b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/format/jpg)

*have fun*

* * *

--EOF--
若无特别说明，本站文章均为原创，转载请保留链接，谢谢

