**原文链接：** [检测iOS的APP性能的一些方法](http://www.starming.com/index.php?v=index&view=91)

**Keywords:** `iOS`，`性能优化`，`性能检测`

## 检测iOS的APP性能的一些方法

首先如果遇到应用卡顿或者因为内存占用过多时一般使用Instruments里的来进行检测。但对于复杂情况可能就需要用到子线程监控主线程的方式来了，下面我对这些方法做些介绍：

# Time Profiler

可以查看多个线程里那些方法费时过多的方法。先将右侧Hide System Libraries打上勾，这样能够过滤信息。然后在Call Tree上会默认按照费时的线程进行排序，单个线程中会也会按照对应的费时方法排序，选择方法后能够通过右侧Heaviest Stack Trace里双击查看到具体的费时操作代码，从而能够有针对性的优化，而不需要在一些本来就不会怎么影响性能的地方过度优化。

# Allocations

这里可以对每个动作的前后进行Generations，对比内存的增加，查看使内存增加的具体的方法和代码所在位置。具体操作是在右侧Generation Analysis里点击Mark Generation，这样会产生一个Generation，切换到其他页面或一段时间产生了另外一个事件时再点Mark Generation来产生一个新的Generation，这样反复，生成多个Generation，查看这几个Generation会看到Growth的大小，如果太大可以点进去查看相应占用较大的线程里右侧Heaviest Stack Trace里查看对应的代码块，然后进行相应的处理。

# Leak

可以在上面区域的Leaks部分看到对应的时间点产生的溢出，选择后在下面区域的Statistics>Allocation Summary能够看到泄漏的对象，同样可以通过Stack Trace查看到具体对应的代码区域。

# 开发时需要注意如何避免一些性能问题

## NSDateFormatter

通过Instruments的检测会发现创建NSDateFormatter或者设置NSDateFormatter的属性的耗时总是排在前面，如何处理这个问题呢，比较推荐的是添加属性或者创建静态变量，这样能够使得创建初始化这个次数降到最低。还有就是可以直接用C，或者这个NSData的Category来解决[https://github.com/samsoffes/sstoolkit/blob/master/SSToolkit/NSData%2BSSToolkitAdditions.m](https://github.com/samsoffes/sstoolkit/blob/master/SSToolkit/NSData%2BSSToolkitAdditions.m)

## UIImage

这里要主要是会影响内存的开销，需要权衡下imagedNamed和imageWithContentsOfFile，了解两者特性后，在只需要显示一次的图片用后者，这样会减少内存的消耗，但是页面显示会增加Image IO的消耗，这个需要注意下。由于imageWithContentsOfFile不缓存，所以需要在每次页面显示前加载一次，这个IO的操作也是需要考虑权衡的一个点。

## 页面加载

如果一个页面内容过多，view过多，这样将长页面中的需要滚动才能看到的那个部分视图内容通过开启新的线程同步的加载。

## 优化首次加载时间

通过Time Profier可以查看到启动所占用的时间，如果太长可以通过Heaviest Stack Trace找到费时的方法进行改造。

# 监控卡顿的方法

还有种方法是在程序里去监控性能问题。可以先看看这个Demo，地址[https://github.com/ming1016/DecoupleDemo](https://github.com/ming1016/DecoupleDemo)。 这样在上线后可以通过这个程序将用户的卡顿操作记录下来，定时发到自己的服务器上，这样能够更大范围的收集性能问题。众所周知，用户层面感知的卡顿都是来自处理所有UI的主线程上，包括在主线程上进行的大计算，大量的IO操作，或者比较重的绘制工作。如何监控主线程呢，首先需要知道的是主线程和其它线程一样都是靠NSRunLoop来驱动的。可以先看看CFRunLoopRun的大概的逻辑

```source-c
int32_t __CFRunLoopRun()
{
    __CFRunLoopDoObservers(KCFRunLoopEntry);
    do
    {
        __CFRunLoopDoObservers(kCFRunLoopBeforeTimers);
        __CFRunLoopDoObservers(kCFRunLoopBeforeSources); //这里开始到kCFRunLoopBeforeWaiting之间处理时间是感知卡顿的关键地方

        __CFRunLoopDoBlocks();
        __CFRunLoopDoSource0(); //处理UI事件

        //GCD dispatch main queue
        CheckIfExistMessagesInMainDispatchQueue();

        //休眠前
        __CFRunLoopDoObservers(kCFRunLoopBeforeWaiting);

        //等待msg
        mach_port_t wakeUpPort = SleepAndWaitForWakingUpPorts();

        //等待中

        //休眠后，唤醒
        __CFRunLoopDoObservers(kCFRunLoopAfterWaiting);

        //定时器唤醒
        if (wakeUpPort == timerPort)
            __CFRunLoopDoTimers();

        //异步处理
        else if (wakeUpPort == mainDispatchQueuePort)
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()

        //UI，动画
        else
            __CFRunLoopDoSource1();

        //确保同步
        __CFRunLoopDoBlocks();

    } while (!stop && !timeout);

    //退出RunLoop
    __CFRunLoopDoObservers(CFRunLoopExit);
}
```

根据这个RunLoop我们能够通过CFRunLoopObserverRef来度量。用GCD里的dispatch_semaphore_t开启一个新线程，设置一个极限值和出现次数的值，然后获取主线程上在kCFRunLoopBeforeSources到kCFRunLoopBeforeWaiting再到kCFRunLoopAfterWaiting两个状态之间的超过了极限值和出现次数的场景，将堆栈dump下来，最后发到服务器做收集，通过堆栈能够找到对应出问题的那个方法。

```source-c
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    MyClass *object = (__bridge MyClass*)info;
    object->activity = activity;
}

static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info){
    SMLagMonitor *lagMonitor = (__bridge SMLagMonitor*)info;
    lagMonitor->runLoopActivity = activity;

    dispatch_semaphore_t semaphore = lagMonitor->dispatchSemaphore;
    dispatch_semaphore_signal(semaphore);
}

- (void)endMonitor {
    if (!runLoopObserver) {
        return;
    }
    CFRunLoopRemoveObserver(CFRunLoopGetMain(), runLoopObserver, kCFRunLoopCommonModes);
    CFRelease(runLoopObserver);
    runLoopObserver = NULL;
}

- (void)beginMonitor {
    if (runLoopObserver) {
        return;
    }
    dispatchSemaphore = dispatch_semaphore_create(0); //Dispatch Semaphore保证同步
    //创建一个观察者
    CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
    runLoopObserver = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                              kCFRunLoopAllActivities,
                                              YES,
                                              0,
                                              &runLoopObserverCallBack,
                                              &context);
    //将观察者添加到主线程runloop的common模式下的观察中
    CFRunLoopAddObserver(CFRunLoopGetMain(), runLoopObserver, kCFRunLoopCommonModes);

    //创建子线程监控
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        //子线程开启一个持续的loop用来进行监控
        while (YES) {
            long semaphoreWait = dispatch_semaphore_wait(dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, 30*NSEC_PER_MSEC));
            if (semaphoreWait != 0) {
                if (!runLoopObserver) {
                    timeoutCount = 0;
                    dispatchSemaphore = 0;
                    runLoopActivity = 0;
                    return;
                }
                //两个runloop的状态，BeforeSources和AfterWaiting这两个状态区间时间能够检测到是否卡顿
                if (runLoopActivity == kCFRunLoopBeforeSources || runLoopActivity == kCFRunLoopAfterWaiting) {
                    //出现三次出结果
                    if (++timeoutCount < 3) {
                        continue;
                    }

                    //将堆栈信息上报服务器的代码放到这里

                } //end activity
            }// end semaphore wait
            timeoutCount = 0;
        }// end while
    });

}
```

有时候造成卡顿是因为数据异常，过多，或者过大造成的，亦或者是操作的异常出现的，这样的情况可能在平时日常开发测试中难以遇到，但是在真实的特别是用户受众广的情况下会有人出现，这样这种收集卡顿的方式还是有价值的。

## [](https://github.com/ming1016/study/wiki/%E6%A3%80%E6%B5%8BiOS%E7%9A%84APP%E6%80%A7%E8%83%BD%E7%9A%84%E4%B8%80%E4%BA%9B%E6%96%B9%E6%B3%95#%E5%A0%86%E6%A0%88dump%E7%9A%84%E6%96%B9%E6%B3%95)堆栈dump的方法

第一种是直接调用系统函数获取栈信息，这种方法只能够获得简单的信息，没法配合dSYM获得具体哪行代码出了问题，类型也有限。这种方法的主要思路是signal进行错误信号的获取。代码如下

```source-objc
static int s_fatal_signals[] = {
    SIGABRT,
    SIGBUS,
    SIGFPE,
    SIGILL,
    SIGSEGV,
    SIGTRAP,
    SIGTERM,
    SIGKILL,
};

static int s_fatal_signal_num = sizeof(s_fatal_signals) / sizeof(s_fatal_signals[0]);

void UncaughtExceptionHandler(NSException *exception) {
    NSArray *exceptionArray = [exception callStackSymbols]; //得到当前调用栈信息
    NSString *exceptionReason = [exception reason];       //非常重要，就是崩溃的原因
    NSString *exceptionName = [exception name];           //异常类型
}

void SignalHandler(int code)
{
    NSLog(@"signal handler = %d",code);
}

void InitCrashReport()
{
    //系统错误信号捕获
    for (int i = 0; i < s_fatal_signal_num; ++i) {
        signal(s_fatal_signals[i], SignalHandler);
    }

    //oc未捕获异常的捕获
    NSSetUncaughtExceptionHandler(&UncaughtExceptionHandler);
}
int main(int argc, char * argv[]) {
    @autoreleasepool {
        InitCrashReport();
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

使用PLCrashReporter的话出的报告看起来能够定位到问题代码的具体位置了。

```source-objc
NSData *lagData = [[[PLCrashReporter alloc]
                                          initWithConfiguration:[[PLCrashReporterConfig alloc] initWithSignalHandlerType:PLCrashReporterSignalHandlerTypeBSD symbolicationStrategy:PLCrashReporterSymbolicationStrategyAll]] generateLiveReport];
PLCrashReport *lagReport = [[PLCrashReport alloc] initWithData:lagData error:NULL];
NSString *lagReportString = [PLCrashReportTextFormatter stringValueForCrashReport:lagReport withTextFormat:PLCrashReportTextFormatiOS];
//将字符串上传服务器
NSLog(@"lag happen, detail below: \n %@",lagReportString);
```

下面是测试Demo里堆栈里的内容

```
2016-03-28 14:59:26.922 HomePageTest[4803:201212]  INFO: Reveal Server started (Protocol Version 25).
2016-03-28 14:59:27.134 HomePageTest[4803:201212] 费时测试
2016-03-28 14:59:29.262 HomePageTest[4803:201212] 费时测试
2016-03-28 14:59:30.865 HomePageTest[4803:201212] 费时测试
2016-03-28 14:59:32.115 HomePageTest[4803:201212] 费时测试
2016-03-28 14:59:33.369 HomePageTest[4803:201212] 费时测试
2016-03-28 14:59:34.832 HomePageTest[4803:201212] 费时测试
2016-03-28 14:59:34.836 HomePageTest[4803:201615] lag happen, detail below: 
 Incident Identifier: 73BEF9D2-EBE3-49DF-B95B-7392635631A3
CrashReporter Key:   TODO
Hardware Model:      x86_64
Process:         HomePageTest [4803]
Path:            /Users/daiming/Library/Developer/CoreSimulator/Devices/444AAB95-C393-45CC-B5DC-0FB8611068F9/data/Containers/Bundle/Application/9CEE3A3A-9469-44F5-8112-FF0550ED8009/HomePageTest.app/HomePageTest
Identifier:      com.xiaojukeji.HomePageTest
Version:         1.0 (1)
Code Type:       X86-64
Parent Process:  debugserver [4805]

Date/Time:       2016-03-28 06:59:34 +0000
OS Version:      Mac OS X 9.2 (15D21)
Report Version:  104

Exception Type:  SIGTRAP
Exception Codes: TRAP_TRACE at 0x10aee6f79
Crashed Thread:  2

Thread 0:
0   libsystem_kernel.dylib              0x000000010ec6b206 __semwait_signal + 10
1   libsystem_c.dylib                   0x000000010e9f2b9e usleep + 54
2   HomePageTest                        0x000000010aedf934 -[TestTableView configCell] + 820
3   HomePageTest                        0x000000010aee5c89 -[testTableViewController observeValueForKeyPath:ofObject:change:context:] + 601
4   Foundation                          0x000000010b9c2564 NSKeyValueNotifyObserver + 347
5   Foundation                          0x000000010b9c178f NSKeyValueDidChange + 466
6   Foundation                          0x000000010b9bf003 -[NSObject(NSKeyValueObservingPrivate) _changeValueForKey:key:key:usingBlock:] + 1176
7   Foundation                          0x000000010ba1d35f _NSSetObjectValueAndNotify + 261
8   HomePageTest                        0x000000010aec9c26 -[DCTableView tableView:cellForRowAtIndexPath:] + 262
9   UIKit                               0x000000010c872e43 -[UITableView _createPreparedCellForGlobalRow:withIndexPath:willDisplay:] + 766
10  UIKit                               0x000000010c872f7b -[UITableView _createPreparedCellForGlobalRow:willDisplay:] + 74
11  UIKit                               0x000000010c847a39 -[UITableView _updateVisibleCellsNow:isRecursive:] + 2996
12  UIKit                               0x000000010c87c01c -[UITableView _performWithCachedTraitCollection:] + 92
13  UIKit                               0x000000010c862edc -[UITableView layoutSubviews] + 224
14  UIKit                               0x000000010c7d04a3 -[UIView(CALayerDelegate) layoutSublayersOfLayer:] + 703
15  QuartzCore                          0x000000010c49a59a -[CALayer layoutSublayers] + 146
16  QuartzCore                          0x000000010c48ee70 _ZN2CA5Layer16layout_if_neededEPNS_11TransactionE + 366
17  QuartzCore                          0x000000010c48ecee _ZN2CA5Layer28layout_and_display_if_neededEPNS_11TransactionE + 24
18  QuartzCore                          0x000000010c483475 _ZN2CA7Context18commit_transactionEPNS_11TransactionE + 277
19  QuartzCore                          0x000000010c4b0c0a _ZN2CA11Transaction6commitEv + 486
20  QuartzCore                          0x000000010c4bf9f4 _ZN2CA7Display11DisplayLink14dispatch_itemsEyyy + 576
21  CoreFoundation                      0x000000010e123c84 __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__ + 20
22  CoreFoundation                      0x000000010e123831 __CFRunLoopDoTimer + 1089
23  CoreFoundation                      0x000000010e0e5241 __CFRunLoopRun + 1937
24  CoreFoundation                      0x000000010e0e4828 CFRunLoopRunSpecific + 488
25  GraphicsServices                    0x0000000110479ad2 GSEventRunModal + 161
26  UIKit                               0x000000010c719610 UIApplicationMain + 171
27  HomePageTest                        0x000000010aee0fdf main + 111
28  libdyld.dylib                       0x000000010e92b92d start + 1

Thread 1:
0   libsystem_kernel.dylib              0x000000010ec6bfde kevent64 + 10
1   libdispatch.dylib                   0x000000010e8e6262 _dispatch_mgr_init + 0

Thread 2 Crashed:
0   HomePageTest                        0x000000010b04a445 -[PLCrashReporter generateLiveReportWithThread:error:] + 632
1   HomePageTest                        0x000000010aee6f79 __28-[SMLagMonitor beginMonitor]_block_invoke + 425
2   libdispatch.dylib                   0x000000010e8d6e5d _dispatch_call_block_and_release + 12
3   libdispatch.dylib                   0x000000010e8f749b _dispatch_client_callout + 8
4   libdispatch.dylib                   0x000000010e8dfbef _dispatch_root_queue_drain + 1829
5   libdispatch.dylib                   0x000000010e8df4c5 _dispatch_worker_thread3 + 111
6   libsystem_pthread.dylib             0x000000010ec2f68f _pthread_wqthread + 1129
7   libsystem_pthread.dylib             0x000000010ec2d365 start_wqthread + 13

Thread 3:
0   libsystem_kernel.dylib              0x000000010ec6b6de __workq_kernreturn + 10
1   libsystem_pthread.dylib             0x000000010ec2d365 start_wqthread + 13

Thread 4:
0   libsystem_kernel.dylib              0x000000010ec65386 mach_msg_trap + 10
1   CoreFoundation                      0x000000010e0e5b64 __CFRunLoopServiceMachPort + 212
2   CoreFoundation                      0x000000010e0e4fbf __CFRunLoopRun + 1295
3   CoreFoundation                      0x000000010e0e4828 CFRunLoopRunSpecific + 488
4   WebCore                             0x0000000113408f65 _ZL12RunWebThreadPv + 469
5   libsystem_pthread.dylib             0x000000010ec2fc13 _pthread_body + 131
6   libsystem_pthread.dylib             0x000000010ec2fb90 _pthread_body + 0
7   libsystem_pthread.dylib             0x000000010ec2d375 thread_start + 13

Thread 5:
0   libsystem_kernel.dylib              0x000000010ec6b6de __workq_kernreturn + 10
1   libsystem_pthread.dylib             0x000000010ec2d365 start_wqthread + 13

Thread 6:
0   libsystem_kernel.dylib              0x000000010ec6b6de __workq_kernreturn + 10
1   libsystem_pthread.dylib             0x000000010ec2d365 start_wqthread + 13

Thread 7:
0   libsystem_kernel.dylib              0x000000010ec6b6de __workq_kernreturn + 10
1   libsystem_pthread.dylib             0x000000010ec2d365 start_wqthread + 13

Thread 2 crashed with X86-64 Thread State:
   rip: 0x000000010b04a445    rbp: 0x0000700000093da0    rsp: 0x0000700000093b10    rax: 0x0000700000093b70 
   rbx: 0x0000700000093cb0    rcx: 0x0000000000001003    rdx: 0x0000000000000000    rdi: 0x000000010b04a5ca 
   rsi: 0x0000700000093b40     r8: 0x0000000000000014     r9: 0x0000000000000000    r10: 0x000000010ec65362 
   r11: 0x0000000000000246    r12: 0x00007fdaf5800940    r13: 0x0000000000000000    r14: 0x0000000000000009 
   r15: 0x0000700000093b90 rflags: 0x0000000000000202     cs: 0x000000000000002b     fs: 0x0000000000000000 
    gs: 0x0000000000000000 

Binary Images:
       0x10aebb000 -        0x10b0a5fff +HomePageTest x86_64  <13a9b9abded0364fb42e65847b3acbbb> /Users/daiming/Library/Developer/CoreSimulator/Devices/444AAB95-C393-45CC-B5DC-0FB8611068F9/data/Containers/Bundle/Application/9CEE3A3A-9469-44F5-8112-FF0550ED8009/HomePageTest.app/HomePageTest
       0x10b2a1000 -        0x10b2c8fff  dyld_sim x86_64  <0bf161d7efa93cbeae2b84f9a70fc853> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/dyld_sim
       0x10b30d000 -        0x10b314fff  libBacktraceRecording.dylib x86_64  <604a2e49cf2f39f7883b37336adb402e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libBacktraceRecording.dylib
       0x10b31b000 -        0x10b31ffff  libViewDebuggerSupport.dylib x86_64  <3d54d318b67735e5874c72a20317b523> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator9.2.sdk/Developer/Library/PrivateFrameworks/DTDDISupport.framework/libViewDebuggerSupport.dylib
       0x10b326000 -        0x10b337fff  libz.1.dylib x86_64  <a603371a88ff3f8e86de61d7443888a9> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libz.1.dylib
       0x10b33d000 -        0x10b5b9fff  CFNetwork x86_64  <cda8332f7c2836df98f216a6802a990a> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CFNetwork.framework/CFNetwork
       0x10b791000 -        0x10b93dfff  CoreGraphics x86_64  <291e992b6a4e3362823863313d3850b4> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreGraphics.framework/CoreGraphics
       0x10b9b9000 -        0x10bc35fff  Foundation x86_64  <0bbf69fb34a63fa3a540ec4ce375212c> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Foundation.framework/Foundation
       0x10be29000 -        0x10c173fff  ImageIO x86_64  <28325a1c347230ada3e8c4dc833ef05a> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/ImageIO.framework/ImageIO
       0x10c23c000 -        0x10c300fff  MobileCoreServices x86_64  <40a78b7dfb1e31fc85d0d98657f979c6> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/MobileCoreServices.framework/MobileCoreServices
       0x10c382000 -        0x10c4fdfff  QuartzCore x86_64  <b9ff29d64f6e306ea01703ae4cd10f29> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/QuartzCore.framework/QuartzCore
       0x10c5b9000 -        0x10c625fff  Security x86_64  <5c3df729a91c365ba314108c0b668df6> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Security.framework/Security
       0x10c67b000 -        0x10c6c4fff  SystemConfiguration x86_64  <cfa2c5a5c2123d80a230892aedce7ece> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/SystemConfiguration.framework/SystemConfiguration
       0x10c6f4000 -        0x10d3ccfff  UIKit x86_64  <b5c879e5e265364da6b695d7bb4ef462> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/UIKit.framework/UIKit
       0x10dc38000 -        0x10df9afff  libobjc.A.dylib x86_64  <1643ada50eaa3d938654df70ddca4045> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libobjc.A.dylib
       0x10e075000 -        0x10e076fff  libSystem.dylib x86_64  <3deb27f2e0b5314abf0a8880e3d52560> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libSystem.dylib
       0x10e07c000 -        0x10e3e2fff  CoreFoundation x86_64  <f47aa9b1fe6e3472928e6622ec331f1f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
       0x10e54b000 -        0x10e6e3fff  MapKit x86_64  <2acdf735086f3e5897a4520417957aab> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/MapKit.framework/MapKit
       0x10e807000 -        0x10e80bfff  libcache.dylib x86_64  <8fe45f989f2e3ccaaf0b2a5fdb2d0177> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libcache.dylib
       0x10e810000 -        0x10e81bfff  libcommonCrypto.dylib x86_64  <45b23c8e43993a6b93f9ba749105bed3> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libcommonCrypto.dylib
       0x10e828000 -        0x10e82ffff  libcompiler_rt.dylib x86_64  <c4852a1cdd6736168b91a75b50f2b65e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libcompiler_rt.dylib
       0x10e837000 -        0x10e83efff  libcopyfile.dylib x86_64  <4be0e6ca0b3a313c97901d33d3ea75a8> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libcopyfile.dylib
       0x10e845000 -        0x10e8bcfff  libcorecrypto.dylib x86_64  <34495822e7913cdbbb141c083740b04e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libcorecrypto.dylib
       0x10e8d5000 -        0x10e902fff  libdispatch.dylib x86_64  <04d9590852903114af8e9de07daac6d4> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/introspection/libdispatch.dylib
       0x10e929000 -        0x10e92bfff  libdyld.dylib x86_64  <236c574a79e73e1abf19e1cbc04ac582> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libdyld.dylib
       0x10e932000 -        0x10e932fff  liblaunch.dylib x86_64  <6fa06587ed1a3b5d99234a093f8d5b7e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/liblaunch.dylib
       0x10e938000 -        0x10e93dfff  libmacho.dylib x86_64  <1e351755db163b5c82fdb47b354fcb51> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libmacho.dylib
       0x10e943000 -        0x10e944fff  libremovefile.dylib x86_64  <a69be47eef3f35aebbd6eb19688a9dfa> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libremovefile.dylib
       0x10e949000 -        0x10e960fff  libsystem_asl.dylib x86_64  <34369616ac8339a8b4ee9d1a7df0ba0f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_asl.dylib
       0x10e96d000 -        0x10e96efff  libsystem_blocks.dylib x86_64  <61456881d96432178fff58643863c59e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_blocks.dylib
       0x10e974000 -        0x10e9fcfff  libsystem_c.dylib x86_64  <fc0dde24a78d39ca848ac937d5903386> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_c.dylib
       0x10ea24000 -        0x10ea26fff  libsystem_configuration.dylib x86_64  <c09c551d48d0371daaf79132e6c7ab73> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_configuration.dylib
       0x10ea2d000 -        0x10ea2ffff  libsystem_containermanager.dylib x86_64  <e0dc470e6ba2349097410f280fe09ed4> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_containermanager.dylib
       0x10ea34000 -        0x10ea35fff  libsystem_coreservices.dylib x86_64  <40b277d390db3e7ea6b50de8a9279786> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_coreservices.dylib
       0x10ea3b000 -        0x10ea51fff  libsystem_coretls.dylib x86_64  <55dc52c450e83eb5b4d26dbe24018539> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_coretls.dylib
       0x10ea5d000 -        0x10ea65fff  libsystem_dnssd.dylib x86_64  <d5c0910695913a799ffffb6cf4653c00> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_dnssd.dylib
       0x10ea6c000 -        0x10ea8ffff  libsystem_info.dylib x86_64  <f3ab2fc36ae332ab86398880e4325938> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_info.dylib
       0x10eaa2000 -        0x10eaa3fff  libsystem_sim_kernel.dylib x86_64  <3402a54a835033c997e9fc27db24d85b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_sim_kernel.dylib
       0x10eaa9000 -        0x10ead6fff  libsystem_m.dylib x86_64  <67570d2171683e96b77aeeda269b8bcb> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_m.dylib
       0x10eade000 -        0x10eafafff  libsystem_malloc.dylib x86_64  <bd202b02a72d329999d6307a1aa14482> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_malloc.dylib
       0x10eb04000 -        0x10eb5efff  libsystem_network.dylib x86_64  <a850352b5da23be89939b2652af84160> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_network.dylib
       0x10eb92000 -        0x10eb9cfff  libsystem_notify.dylib x86_64  <d55e395a67f9350f9058866e3c7665f0> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_notify.dylib
       0x10eba5000 -        0x10eba7fff  libsystem_sim_platform.dylib x86_64  <3fe0d2367d073f7088330d74f4e48829> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_sim_platform.dylib
       0x10ebad000 -        0x10ebadfff  libsystem_sim_pthread.dylib x86_64  <1e18f800d5ed31dcb0746cabc31ab297> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_sim_pthread.dylib
       0x10ebb2000 -        0x10ebb5fff  libsystem_sandbox.dylib x86_64  <6ccd8c991b093603b0c721dbc3536f5b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_sandbox.dylib
       0x10ebbc000 -        0x10ebccfff  libsystem_trace.dylib x86_64  <75d0d4effb95381db5dd2d32a54da7fe> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libsystem_trace.dylib
       0x10ebdb000 -        0x10ebe1fff  libunwind.dylib x86_64  <072cea349492300fb68c39c751845ede> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libunwind.dylib
       0x10ebe8000 -        0x10ec0efff  libxpc.dylib x86_64  <3e9ae1639a9535d3ba2c2e7144498580> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/system/libxpc.dylib
       0x10ec2c000 -        0x10ec35fff  libsystem_pthread.dylib x86_64  <327cecd0b88131538fcc4fd4818b7f16> /usr/lib/system/libsystem_pthread.dylib
       0x10ec43000 -        0x10ec4bfff  libsystem_platform.dylib x86_64  <d3a27e107f083603acc87a92b2c04bab> /usr/lib/system/libsystem_platform.dylib
       0x10ec54000 -        0x10ec72fff  libsystem_kernel.dylib x86_64  <9ceb6c3b1caf3c32a9fd93bc72cbcea1> /usr/lib/system/libsystem_kernel.dylib
       0x10ec88000 -        0x10ecb1fff  libc++abi.dylib x86_64  <81525def7b933f8296838959fa4ce98d> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libc++abi.dylib
       0x10ecc0000 -        0x10ed13fff  libc++.1.dylib x86_64  <889b6cbf910536c89fae86374f07e767> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libc++.1.dylib
       0x10ed67000 -        0x10ed7cfff  libMobileGestalt.dylib x86_64  <05d34bd9e19d306c885605aecafe9893> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libMobileGestalt.dylib
       0x10edb3000 -        0x10ee0efff  IOKit x86_64  <853021f275723ee4b9fc0684bbee030b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/IOKit.framework/Versions/A/IOKit
       0x10ee40000 -        0x10ef3afff  libsqlite3.dylib x86_64  <22bfc8d7f5f632919b01fdc883e24d26> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libsqlite3.dylib
       0x10ef52000 -        0x10f041fff  libxml2.2.dylib x86_64  <cbe1fcf1d9b63516b8fa96cb45841d40> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libxml2.2.dylib
       0x10f077000 -        0x10f27efff  libicucore.A.dylib x86_64  <5c7ce20d3b0f3cd2b19e2c8d4cae9147> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libicucore.A.dylib
       0x10f341000 -        0x10f351fff  libbsm.0.dylib x86_64  <7f09d0545ae037119f611768227b64b3> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libbsm.0.dylib
       0x10f359000 -        0x10f359fff  Accelerate x86_64  <1b1483eeaea537a09ee40b3b8faf8acb> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Accelerate
       0x10f35c000 -        0x10f81afff  vImage x86_64  <00f092dc41ec3f8e852efd7ce1efb1e2> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Frameworks/vImage.framework/vImage
       0x10f875000 -        0x10f875fff  vecLib x86_64  <cdfef7cd1b45390e81e561db4c07b35f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Frameworks/vecLib.framework/vecLib
       0x10f878000 -        0x10f981fff  libvDSP.dylib x86_64  <ddafe8e5e10e33389a7a08c67895c980> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Frameworks/vecLib.framework/libvDSP.dylib
       0x10f98e000 -        0x10fd90fff  libLAPACK.dylib x86_64  <39f724015fda3ef298579cd38168b4e0> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Frameworks/vecLib.framework/libLAPACK.dylib
       0x10fdc0000 -        0x10ff80fff  libBLAS.dylib x86_64  <3d2d6842f59035d795247d6bf77503ca> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Frameworks/vecLib.framework/libBLAS.dylib
       0x10ff9f000 -        0x11003afff  libvMisc.dylib x86_64  <6716692bfaaf33a29d95873d7c93e570> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Frameworks/vecLib.framework/libvMisc.dylib
       0x110044000 -        0x11005afff  libLinearAlgebra.dylib x86_64  <6880200dcd1d3e53a92e388887919970> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Frameworks/vecLib.framework/libLinearAlgebra.dylib
       0x110063000 -        0x110073fff  libSparseBLAS.dylib x86_64  <e8069ab282f0344780f5286fc2b9e2c4> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accelerate.framework/Frameworks/vecLib.framework/libSparseBLAS.dylib
       0x11007b000 -        0x110099fff  libextension.dylib x86_64  <9f870091398d38fa8138b45d528cc7f2> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libextension.dylib
       0x1100ba000 -        0x1100e5fff  libarchive.2.dylib x86_64  <6349fcc5aa9d3103b49c8813f0c14f4b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libarchive.2.dylib
       0x1100f0000 -        0x11010bfff  libCRFSuite.dylib x86_64  <512e912e804033d980f0141de864a82a> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libCRFSuite.dylib
       0x110116000 -        0x110117fff  liblangid.dylib x86_64  <31a81762a91b36d2a3210ca0bf929fa4> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/liblangid.dylib
       0x11011c000 -        0x11012afff  libbz2.1.0.dylib x86_64  <dc78bdfa5cb0386e97b09dedeeb048f8> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libbz2.1.0.dylib
       0x110130000 -        0x11014afff  liblzma.5.dylib x86_64  <7a19c9a4e7b930b7a32c9ab778b9bcd4> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/liblzma.5.dylib
       0x110152000 -        0x1101dffff  AppleJPEG x86_64  <e0884d4cdfd63a3b91f54a9c5b5e0b8d> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/AppleJPEG.framework/AppleJPEG
       0x11023f000 -        0x110379fff  CoreText x86_64  <c28657f25ac837b4ab950f78e41e5d90> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreText.framework/CoreText
       0x11041f000 -        0x110438fff  CoreVideo x86_64  <1869a884854b346bbd06dfafaf765470> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreVideo.framework/CoreVideo
       0x11044f000 -        0x11045bfff  OpenGLES x86_64  <2e82c1d6cd293127b14be85f3f78c983> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/OpenGLES.framework/OpenGLES
       0x110468000 -        0x110468fff  FontServices x86_64  <d57462a3c6c83a47850e8b5c208b54cc> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/FontServices.framework/FontServices
       0x11046d000 -        0x110481fff  GraphicsServices x86_64  <c351fe76be413e44a3b13d3160497f31> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/GraphicsServices.framework/GraphicsServices
       0x110499000 -        0x11057bfff  libFontParser.dylib x86_64  <b506fe2b071c3a32bb95aa79ad96eb01> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/FontServices.framework/libFontParser.dylib
       0x110634000 -        0x11063efff  libGFXShared.dylib x86_64  <dad95c8768613142bf18ce4551145a6e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/OpenGLES.framework/libGFXShared.dylib
       0x110646000 -        0x11068bfff  libGLImage.dylib x86_64  <cfe342e9fad03b839d345841c94f3b7b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/OpenGLES.framework/libGLImage.dylib
       0x110695000 -        0x110697fff  libCVMSPluginSupport.dylib x86_64  <3927944c0d263f9aaa026dcc0d5e38e0> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/OpenGLES.framework/libCVMSPluginSupport.dylib
       0x11069d000 -        0x1106a5fff  libCoreVMClient.dylib x86_64  <ef6b47d7dda13f92945569e78548a759> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/OpenGLES.framework/libCoreVMClient.dylib
       0x1106ae000 -        0x111371fff  libLLVMContainer.dylib x86_64  <15f2b50ec12c3b5eaf42159df72376df> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/OpenGLES.framework/libLLVMContainer.dylib
       0x1116b8000 -        0x11179cfff  UIFoundation x86_64  <b2d0e3aedc8c3c66a901b7dafeaa1e2f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/UIFoundation.framework/UIFoundation
       0x111810000 -        0x11181afff  UserNotificationServices x86_64  <bdff2d68e1bc3f3ab83e6516f3b34c6c> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/UserNotificationServices.framework/UserNotificationServices
       0x11182c000 -        0x111862fff  FrontBoardServices x86_64  <236f3bdbd0eb3e48b084acea42cb321f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/FrontBoardServices.framework/FrontBoardServices
       0x1118af000 -        0x1118f7fff  BaseBoard x86_64  <1a7979a63f5f30678004145f1c1fed0a> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/BaseBoard.framework/BaseBoard
       0x111948000 -        0x111a05fff  CoreUI x86_64  <ae1a6a3eb7ef3da0803688d9d2bdafa8> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/CoreUI.framework/CoreUI
       0x111afd000 -        0x111decfff  VideoToolbox x86_64  <d818deb4d5b036f795eeeaf3bdf58617> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/VideoToolbox.framework/VideoToolbox
       0x111e6a000 -        0x111e78fff  MobileAsset x86_64  <1c6c8a87693839d3918dd9e78d72e26e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/MobileAsset.framework/MobileAsset
       0x111e88000 -        0x111ea7fff  BackBoardServices x86_64  <6db39b59eb313174836d9e7906fdb212> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/BackBoardServices.framework/BackBoardServices
       0x111ed0000 -        0x1120a4fff  CoreImage x86_64  <7271d38b9e593543a956c13c5de5da6a> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreImage.framework/CoreImage
       0x1121f6000 -        0x11221bfff  DictionaryServices x86_64  <e317ebfeb1a734b3979053d75e497fad> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/DictionaryServices.framework/DictionaryServices
       0x11223f000 -        0x112263fff  SpringBoardServices x86_64  <cc1a9a762b8b35faa95392b3e4622d4d> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/SpringBoardServices.framework/SpringBoardServices
       0x112292000 -        0x1122d9fff  AppSupport x86_64  <b9622cb94ded38e08a59224510d859cb> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/AppSupport.framework/AppSupport
       0x112315000 -        0x112345fff  TextInput x86_64  <07de88f839b330af910590041812bb64> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/TextInput.framework/TextInput
       0x112384000 -        0x112469fff  WebKitLegacy x86_64  <0c79980d94183ccb93144bc9eabcb6ae> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/WebKitLegacy.framework/WebKitLegacy
       0x112529000 -        0x1135f4fff  WebCore x86_64  <67d6527a7ceb3c8fa85ce85c2bf38119> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/WebCore.framework/WebCore
       0x113eec000 -        0x113fc6fff  ProofReader x86_64  <53554e5080083c5ea16438704ba83695> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/ProofReader.framework/ProofReader
       0x113ffc000 -        0x114006fff  libAccessibility.dylib x86_64  <17a037a4c93b3bb8ae12e830128e6d99> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libAccessibility.dylib
       0x114019000 -        0x114070fff  PhysicsKit x86_64  <f93aa95b8d5133219d236f0a02b9b038> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/PhysicsKit.framework/PhysicsKit
       0x11409c000 -        0x1140a6fff  AssertionServices x86_64  <3a58c83f614530a59403e96edba2b0e6> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/AssertionServices.framework/AssertionServices
       0x1140b9000 -        0x1144f0fff  FaceCore x86_64  <7971b628d43231a08f14ea7c4e2138b0> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/FaceCore.framework/FaceCore
       0x114708000 -        0x11478dfff  CoreMedia x86_64  <dd9a55e018b8319d92fe4b2b0c9bd52f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreMedia.framework/CoreMedia
       0x1147f7000 -        0x114850fff  ColorSync x86_64  <0681057f0a74363fa8d20aa7862c1e7b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/ColorSync.framework/ColorSync
       0x114872000 -        0x114874fff  SimulatorClient x86_64  <c2f2a63c1b7e384982099a98992c7721> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/SimulatorClient.framework/SimulatorClient
       0x11487b000 -        0x1148c8fff  CoreAudio x86_64  <794b0ba43f1a3790b617f3fb253debeb> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreAudio.framework/CoreAudio
       0x1148ec000 -        0x1148f0fff  AggregateDictionary x86_64  <b1bee24fe8e838d389075db856f7702b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/AggregateDictionary.framework/AggregateDictionary
       0x1148f8000 -        0x114921fff  libxslt.1.dylib x86_64  <6c42a533a8703a36ad402b99fbf0f73c> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libxslt.1.dylib
       0x11492e000 -        0x114945fff  libmarisa.dylib x86_64  <e3ddc2ae594e3eb894dd994528a10deb> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libmarisa.dylib
       0x114952000 -        0x114f13fff  JavaScriptCore x86_64  <c5c9fbe108973d42ad1ea2015d805490> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/JavaScriptCore.framework/JavaScriptCore
       0x115091000 -        0x11537cfff  AudioToolbox x86_64  <28b1b9057fdf37dcb669386be1820c5c> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/AudioToolbox.framework/AudioToolbox
       0x1154b2000 -        0x1154b7fff  TCC x86_64  <7a73d618505938a6a1fadda3aaf0f88c> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/TCC.framework/TCC
       0x1154c0000 -        0x11554efff  LanguageModeling x86_64  <6fd4f9ff9e393b1a87ab68a9df224e8d> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/LanguageModeling.framework/LanguageModeling
       0x115564000 -        0x115575fff  libcmph.dylib x86_64  <a056b047f15a3df085065ed5f0368f16> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libcmph.dylib
       0x11557e000 -        0x115670fff  libiconv.2.dylib x86_64  <192bd7fdc2943f0ea17b88dbdbc3329b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libiconv.2.dylib
       0x115680000 -        0x115685fff  MediaAccessibility x86_64  <6e41f0c55b703cb2825e200350469a58> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/MediaAccessibility.framework/MediaAccessibility
       0x115691000 -        0x115769fff  AddressBookUI x86_64  <58a5a1b2d8b63e62bc4671df5cec5596> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/AddressBookUI.framework/AddressBookUI
       0x11583e000 -        0x115f69fff  VectorKit x86_64  <36ef06e69448347b973f84f4486ecac2> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/VectorKit.framework/VectorKit
       0x1161f1000 -        0x116268fff  AddressBook x86_64  <d4d77496b2443bcc9b8476c7c6a9cc79> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/AddressBook.framework/AddressBook
       0x1162bf000 -        0x11631ffff  CoreLocation x86_64  <a099137b2feb30ad82864ccda12993f2> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreLocation.framework/CoreLocation
       0x116342000 -        0x1167a0fff  GeoServices x86_64  <ef3a5a767f7c3c81b7653763382ae25b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/GeoServices.framework/GeoServices
       0x116aeb000 -        0x116afbfff  ProtocolBuffer x86_64  <e840436419f43a6b8cc22daff981f66e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/ProtocolBuffer.framework/ProtocolBuffer
       0x116b0d000 -        0x116b93fff  Contacts x86_64  <7e411c617cb53ad1b43f91e18de6fcb0> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Contacts.framework/Contacts
       0x116c4c000 -        0x116d4ffff  ContactsUI x86_64  <425fe9f6c9463295991786b39850b851> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/ContactsUI.framework/ContactsUI
       0x116e35000 -        0x116e41fff  IntlPreferences x86_64  <b71be9b2f9e33eafab9a301e47a8e6fa> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/IntlPreferences.framework/IntlPreferences
       0x116e4f000 -        0x116e6afff  PlugInKit x86_64  <c3066f229d413a048da42f603c2e9b43> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/PlugInKit.framework/PlugInKit
       0x116e85000 -        0x116ebbfff  Accounts x86_64  <1f649ba6d7a33bc6b8b2eb93799ab4ee> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/Accounts.framework/Accounts
       0x116eed000 -        0x117069fff  AVFoundation x86_64  <904190b56923390191dea51a838bd14f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/AVFoundation.framework/AVFoundation
       0x1171ef000 -        0x1171f3fff  CommunicationsFilter x86_64  <ff85c618e7e63fada07316f1b9c8d1fc> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/CommunicationsFilter.framework/CommunicationsFilter
       0x1171f9000 -        0x11721efff  DataAccessExpress x86_64  <e3eb7c5fb6b13a95810d4cfcdead074a> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/DataAccessExpress.framework/DataAccessExpress
       0x117242000 -        0x117300fff  ManagedConfiguration x86_64  <b66ed9c2ab553fd981e95750598ba938> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/ManagedConfiguration.framework/ManagedConfiguration
       0x117399000 -        0x1173d3fff  CoreSpotlight x86_64  <98bf6cfb785735eebf190db4b1c3ae64> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreSpotlight.framework/CoreSpotlight
       0x117416000 -        0x117434fff  vCard x86_64  <9055c75785e4395d8996060255c1631b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/vCard.framework/vCard
       0x117463000 -        0x11748cfff  ContactsFoundation x86_64  <63f515588ea93a06a063e73787d7d15c> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/ContactsFoundation.framework/ContactsFoundation
       0x1174cb000 -        0x117735fff  CoreData x86_64  <e358a02dacfd3e31a9992b6b7ea9e7c5> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreData.framework/CoreData
       0x117854000 -        0x117856fff  OAuth x86_64  <732d326fd0983422aa6a09ea1938e7d7> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/OAuth.framework/OAuth
       0x11785c000 -        0x117875fff  libcompression.dylib x86_64  <1823f7e097263ccd8b713df4f4788903> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libcompression.dylib
       0x11787d000 -        0x1179c1fff  MobileSpotlightIndex x86_64  <6c81b940503b3f7e9a0bfd40658b5fdf> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/MobileSpotlightIndex.framework/MobileSpotlightIndex
       0x117a0b000 -        0x117a0ffff  ParsecSubscriptionServiceSupport x86_64  <dcfd384db24a3ab5991d224a4824301f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/ParsecSubscriptionServiceSupport.framework/ParsecSubscriptionServiceSupport
       0x117a18000 -        0x117cabfff  libmecabra.dylib x86_64  <9f9feceeace7387aaa81ec59d0c3f9b1> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libmecabra.dylib
       0x117d33000 -        0x117d38fff  ConstantClasses x86_64  <2f5266d656843f3c822dc0d2fd6aaa8e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/ConstantClasses.framework/ConstantClasses
       0x117d41000 -        0x117d4cfff  libChineseTokenizer.dylib x86_64  <8d393ad002733693a63608ded180c4c4> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libChineseTokenizer.dylib
       0x117d5b000 -        0x117dcdfff  libAVFAudio.dylib x86_64  <97ad418fb598389fb2c4115f634a61c7> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/AVFoundation.framework/libAVFAudio.dylib
       0x117e29000 -        0x1180e3fff  MediaToolbox x86_64  <5587701cabc23ff0b1fd6d81d0309789> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/MediaToolbox.framework/MediaToolbox
       0x1181ef000 -        0x118223fff  Celestial x86_64  <4cd1941e81a43e3e9ef4bee37a0ee762> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/Celestial.framework/Celestial
       0x118274000 -        0x118275fff  BTLEAudioController x86_64  <a629490c2a8d3f48ba051bc19ee1d3da> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/BTLEAudioController.framework/BTLEAudioController
       0x11827b000 -        0x1182dffff  IMFoundation x86_64  <5243547afc7834479cb6434788543d2f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/IMFoundation.framework/IMFoundation
       0x11832a000 -        0x118336fff  CommonUtilities x86_64  <13c0b53f0c9d37a2915f0793fe809a9f> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/CommonUtilities.framework/CommonUtilities
       0x118343000 -        0x118374fff  libtidy.A.dylib x86_64  <0e2291591e4b3f25881e6c4a98bb38e0> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libtidy.A.dylib
       0x118385000 -        0x118394fff  CacheDelete x86_64  <e35e427c786c35429fac8b60b27650a7> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/CacheDelete.framework/CacheDelete
       0x1183a2000 -        0x1183c4fff  PersistentConnection x86_64  <e9fd98bc5ac739bfb4f4979f159fc2cd> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/PersistentConnection.framework/PersistentConnection
       0x1183ea000 -        0x1183f1fff  DataMigration x86_64  <e0a5b78e95763ff9874c17198d2ab6f7> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/DataMigration.framework/DataMigration
       0x1183fd000 -        0x118471fff  CoreTelephony x86_64  <4cb6c821e0da3386848cdf10650ba48a> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreTelephony.framework/CoreTelephony
       0x1184d9000 -        0x1184dffff  libcupolicy.dylib x86_64  <3b9f4709f1b139adb3630698596f4755> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libcupolicy.dylib
       0x1184e9000 -        0x118520fff  libTelephonyUtilDynamic.dylib x86_64  <bdfdb334207d3deba45c9f2ceafd28fe> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libTelephonyUtilDynamic.dylib
       0x11855e000 -        0x11857efff  CoreBluetooth x86_64  <5cb697ee9e803be689662c722d23792b> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreBluetooth.framework/CoreBluetooth
       0x11ac19000 -        0x11ac21fff  libMobileGestaltExtensions.dylib x86_64  <f9379481b6703400a6affe719472b1e1> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/libMobileGestaltExtensions.dylib
       0x11ac7e000 -        0x11ac80fff  libCGXType.A.dylib x86_64  <a9d3ce5c71e1342b8837a140f26d1dcc> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreGraphics.framework/Resources/libCGXType.A.dylib
       0x11ac8c000 -        0x11acb3fff  libRIP.A.dylib x86_64  <4fa5309d6dea38f1af4b8d1b3f33654e> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreGraphics.framework/Resources/libRIP.A.dylib
       0x11ae05000 -        0x11ae10fff  libGSFontCache.dylib x86_64  <f24b9a1cdbe43bcfae632fc165171473> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/FontServices.framework/libGSFontCache.dylib
       0x11bf53000 -        0x11bf7afff  CoreServicesInternal x86_64  <3f769739c15c3968a37853f346b0a862> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/CoreServicesInternal.framework/CoreServicesInternal
2016-03-28 14:59:35.586 HomePageTest[4803:201212] 费时测试
```

