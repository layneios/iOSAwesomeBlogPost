**原文链接：** [漫谈iOS Crash收集框架](https://nianxi.net/ios/ios-crash-reporter.html)

**Keywords:** `iOS`，`crash`，`崩溃收集`

## 漫谈iOS Crash收集框架

为了能够第一时间发现程序问题，应用程序需要实现自己的崩溃日志收集服务，成熟的开源项目很多，如 [KSCrash](https://github.com/kstenerud/KSCrash)，[plcrashreporter](https://github.com/plausiblelabs/plcrashreporter)，[CrashKit](https://github.com/kaler/CrashKit) 等。追求方便省心，对于保密性要求不高的程序来说，也可以选择各种一条龙Crash统计产品，如 [Crashlytics](http://try.crashlytics.com/)，[Hockeyapp](http://hockeyapp.net/features/crashreports/) ，[友盟](http://www.umeng.com/umeng30_error_type)，[Bugly](http://bugly.qq.com/) 等等。

* 是否集成越多的Crash日志收集服务就越保险？
* 自己收集的Crash日志和系统生成的Crash日志有分歧，应该相信谁？
* 为什么有大量Crash日志显示崩在main函数里,但函数栈中却没有一行自己的代码？
* 野指针类的Crash难定位，有何妙招来应对？

想解释清这些问题，必须从Mach异常说起

### Mach异常与Unix信号

iOS系统自带的 Apple’s Crash Reporter 记录在设备中的Crash日志，Exception Type项通常会包含两个元素： Mach异常 和 Unix信号。

```
Exception Type:         EXC_BAD_ACCESS (SIGSEGV)
Exception Subtype:      KERN_INVALID_ADDRESS at 0x041a6f3

```

Mach异常是什么？它又是如何与Unix信号建立联系的？ Mach是一个XNU的微内核核心，Mach异常是指最底层的内核级异常，被定义在 `<mach/exception_types.h>`下 。每个thread，task，host都有一个异常端口数组，Mach的部分API暴露给了用户态，用户态的开发者可以直接通过Mach API设置thread，task，host的异常端口，来捕获Mach异常，抓取Crash事件。

所有Mach异常都在host层被`ux_exception`转换为相应的Unix信号，并通过`threadsignal`将信号投递到出错的线程。iOS中的 POSIX API 就是通过 Mach 之上的 BSD 层实现的。

![](https://nianxi.net/assets/img/blog/2015-06-27-posix-bsd-mach.png)

因此，`EXC_BAD_ACCESS (SIGSEGV)`表示的意思是：Mach层的EXC_BAD_ACCESS异常，在host层被转换成SIGSEGV信号投递到出错的线程。既然最终以信号的方式投递到出错的线程，那么就可以通过注册signalHandler来捕获信号:

```
signal(SIGSEGV,signalHandler);

```

捕获Mach异常或者Unix信号都可以抓到crash事件，这两种方式哪个更好呢？ 优选Mach异常，因为Mach异常处理会先于Unix信号处理发生，如果Mach异常的handler让程序exit了，那么Unix信号就永远不会到达这个进程了。转换Unix信号是为了兼容更为流行的POSIX标准(SUS规范)，这样不必了解Mach内核也可以通过Unix信号的方式来兼容开发。

> 因为硬件产生的信号(通过CPU陷阱)被Mach层捕获，然后才转换为对应的Unix信号；苹果为了统一机制，于是操作系统和用户产生的信号(通过调用`kill`和`pthread_kill`)也首先沉下来被转换为Mach异常，再转换为Unix信号。

### Crash收集的实现思路

正如上述所说，可以通过捕获Mach异常、或Unix信号两种方式来抓取crash事件，于是总结起来实现方案就一共有3种。

#### 1）Mach异常方式

![](https://nianxi.net/assets/img/blog/2015-06-27-catch-mach-exception.png)

#### 2）Unix信号方式

```
signal(SIGSEGV,signalHandler);

```

#### 3）Mach异常+Unix信号方式

Github上多数开源项目都采用的这种方式，即使在优选捕获Mach异常的情况下，也放弃捕获EXC_CRASH异常，而选择捕获与之对应的SIGABRT信号。著名开源项目[plcrashreporter](https://github.com/plausiblelabs/plcrashreporter)在代码注释中给出了详细的解释：

> We still need to use signal handlers to catch SIGABRT in-process. The kernel sends an `EXC_CRASH` mach exception to denote SIGABRT termination. In that case, catching the Mach exception in-process leads to process deadlock in an uninterruptable wait. Thus, we fall back on BSD signal handlers for SIGABRT, and do not register for `EXC_CRASH`.

另外，需要重点说明的是：对于应用级异常NSException，还需要特殊处理。 你是否见过崩溃在main函数的crash日志，但是函数栈里面没有你的代码：

```
Thread 0 Crashed:
0       libsystem_kernel.dylib          0x3a61757c   __semwait_signal_nocancel + 0x18
1       libsystem_c.dylib               0x3a592a7c   nanosleep$NOCANCEL + 0xa0
2       libsystem_c.dylib               0x3a5adede   usleep$NOCANCEL + 0x2e
3       libsystem_c.dylib               0x3a5c7fe0   abort + 0x50
4       libc++abi.dylib                 0x398f6cd2   abort_message + 0x46
5       libc++abi.dylib                 0x3990f6e0   default_terminate_handler() + 0xf8
6       libobjc.A.dylib                 0x3a054f62   _objc_terminate() + 0xbe
7       libc++abi.dylib                 0x3990d1c4   std::__terminate(void (*)()) + 0x4c
8       libc++abi.dylib                 0x3990cd28   __cxa_rethrow + 0x60
9       libobjc.A.dylib                 0x3a054e12   objc_exception_rethrow + 0x26
10      CoreFoundation                  0x2f7d7f30   CFRunLoopRunSpecific + 0x27c
11      CoreFoundation                  0x2f7d7c9e   CFRunLoopRunInMode + 0x66
12      GraphicsServices                0x346dd65e   GSEventRunModal + 0x86
13      UIKit                           0x32124148   UIApplicationMain + 0x46c
14      XXXXXX                          0x0003b1f2   main + 0x1f2
15      libdyld.dylib                   0x3a561ab4   start + 0x0

```

可以看出是因为某个NSException导致程序Crash的，只有拿到这个NSException，获取它的`reason`，`name`，`callStackSymbols`信息才能确定出问题的程序位置。

```
/* NSException Class Reference */
@property(readonly, copy) NSString *name;
@property(readonly, copy) NSString *reason;
@property(readonly, copy) NSArray *callStackSymbols;
@property(readonly, copy) NSArray *callStackReturnAddresses;

```

方法很简单，可通过注册`NSUncaughtExceptionHandler`捕获异常信息:

```
static void my_uncaught_exception_handler (NSException *exception) {
    //这里可以取到 NSException 信息
}
NSSetUncaughtExceptionHandler(&my_uncaught_exception_handler);

```

将拿到的NSException细节写入Crash日志，精准的定位出错程序位置：

```
Application Specific Information:
*** Terminating app due to uncaught exception 'NSUnknownKeyException', reason: '[<__NSDictionaryI 0x14554d00> setValue:forUndefinedKey:]: this class is not key value coding-compliant for the key key.'

Last Exception Backtrace:
0 CoreFoundation 0x2f8a3f7e     __exceptionPreprocess + 0x7e
1 libobjc.A.dylib 0x3a054cc     objc_exception_throw + 0x22
2 CoreFoundation 0x2f8a3c94     -[NSException raise] + 0x4
3 Foundation 0x301e8f1e         -[NSObject(NSKeyValueCoding) setValue:forKey:] + 0xc6
4 DemoCrash 0x00085306          -[ViewController crashMethod] + 0x6e
5 DemoCrash 0x00084ecc          main + 0x1cc
6 DemoCrash 0x00084cf8          start + 0x24

```

那么，是不是收到了大量crash在main函数却没有NSException信息的日志，就代表自己集成的Crash日志收集服务没有注册NSUncaughtExceptionHandler呢？不一定，还有另外一种可能，就是被同时存在的其他Crash日志收集服务给坑了。

### 多个Crash日志收集服务共存的坑

是的，在自己的程序里集成多个Crash日志收集服务实在不是明智之举。通常情况下，第三方功能性SDK都会集成一个Crash收集服务，以及时发现自己SDK的问题。当各家的服务都以保证自己的Crash统计正确完整为目的时，难免出现时序手脚，强行覆盖等等的恶意竞争，总会有人默默被坑。

#### 1）拒绝传递 UncaughtExceptionHandler

如果同时有多方通过NSSetUncaughtExceptionHandler注册异常处理程序，和平的作法是：后注册者通过NSGetUncaughtExceptionHandler将先前别人注册的handler取出并备份，在自己handler处理完后自觉把别人的handler注册回去，规规矩矩的传递。不传递强行覆盖的后果是，在其之前注册过的日志收集服务写出的Crash日志就会因为取不到NSException而丢失`Last Exception Backtrace`等信息。（P.S. iOS系统自带的Crash Reporter不受影响）

在开发测试阶段，可以利用 [fishhook](https://github.com/facebook/fishhook) 框架去hook`NSSetUncaughtExceptionHandler`方法，这样就可以清晰的看到handler的传递流程断在哪里，快速定位污染环境者。不推荐利用调试器添加符号断点来检查，原因是一些Crash收集框架在调试状态下是不工作的。

检测代码示例：

```
static NSUncaughtExceptionHandler *g_vaildUncaughtExceptionHandler;
static void (*ori_NSSetUncaughtExceptionHandler)( NSUncaughtExceptionHandler * );

void my_NSSetUncaughtExceptionHandler( NSUncaughtExceptionHandler * handler)
{
    g_vaildUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
    if (g_vaildUncaughtExceptionHandler != NULL) {
        NSLog(@"UncaughtExceptionHandler=%p",g_vaildUncaughtExceptionHandler);
    }

    ori_NSSetUncaughtExceptionHandler(handler);
    NSLog(@"%@",[NSThread callStackSymbols]);

    g_vaildUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
    NSLog(@"UncaughtExceptionHandler=%p",g_vaildUncaughtExceptionHandler);
}

```

对于越狱插件注入应用进程内部，恶意覆盖NSSetUncaughtExceptionHandler的情况，应用程序本身处理起来比较弱势，因为越狱环境下操作时序的玩法比较多权利比较大。

#### 2）Mach异常端口换出+信号处理Handler覆盖

和NSSetUncaughtExceptionHandler的情况类似，设置过的Mach异常端口和信号处理程序也有可能被干掉，导致无法捕获Crash事件。

#### 3）影响系统崩溃日志准确性

应用层参与收集Crash日志的服务方越多，越有可能影响iOS系统自带的Crash Reporter。由于进程内线程数组的变动，可能会导致系统日志中线程的`Crashed` 标签标记错位，可以搜索`abort()`等关键字来复查系统日志的准确性。 若程序因NSException而Crash，系统日志中的`Last Exception Backtrace`信息是完整准确的，不会受应用层的胡来而影响，可作为排查问题的参考线索。

### ObjC野指针类的Crash

收集Crash日志这个步骤没有问题的情况下，还是有很多全系统栈的日志的情况，没有自己一行代码，分析起来十分棘手，ObjC野指针类的Crash正是如此，这里推荐几篇好文章：

* [如何定位Obj-C野指针随机Crash(一)：先提高野指针Crash率](http://bugly.qq.com/blog/?p=200)
* [如何定位Obj-C野指针随机Crash(二)：让非必现Crash变成必现](http://bugly.qq.com/blog/?p=308)
* [如何定位Obj-C野指针随机Crash(三)：加点黑科技让Crash自报家门](http://bugly.qq.com/blog/?p=335)
* [分析objc_msgSend()处崩溃的小技巧](http://www.sealiesoftware.com/blog/archive/2008/09/22/objc_explain_So_you_crashed_in_objc_msgSend.html)

除此之外，在Crash日志中补充记录一些额外信息可以辅助定位，如切面标记线程出处、队列出处，记录用户操作轨迹等等……

