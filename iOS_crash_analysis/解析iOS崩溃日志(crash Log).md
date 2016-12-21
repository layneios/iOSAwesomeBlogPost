**原文链接：**[解析iOS崩溃日志(crash Log)](http://lieyunye.github.io/blog/2013/09/10/how-to-analyse-ios-crash-log/)

**Keywords:** `iOS`，`崩溃解析`，`crash`，`崩溃日志`



# 解析IOS崩溃日志(crash Log)

  最近在解析umeng错误分析日志上有了重大突破！

  很显然，我们的应用免不了crash，各种各样的crash，不过大部分在提交至appstore前经过严格的“消毒”后，所剩无几了。but（这个词..）漏网之鱼总是有的嘛（貌似很多..囧）。好吧，看下文：

  首先看一些这些线上app crash 信息：

```
* Application received signal SIGSEGV
* Application received signal SIGBUS
* -[__NSArrayM objectAtIndex:]: index 4294967295 beyond bounds for empty array
* -[JKArray objectAtIndex:]: index (0) beyond bounds (0)

```

SIGSEGV和SIGBUS一般是因为访问已被释放的内存或者调用不存在的方法导致的，余下两个就是数组越界的问题了 这些你都知道的，然后来看看具体的log信息：

Application received signal SIGSEGV

```
Application received signal SIGSEGV
(null)
(
0   CoreFoundation                      0x32f1c3ff  + 186
1   libobjc.A.dylib                     0x3ac17963 objc_exception_throw + 30
2   CoreFoundation                      0x32f1c307  + 106
3   appname                            0x14e1e1 appname + 1364449
4   libsystem_c.dylib                   0x3b08bd33 _sigtramp + 34
5   appname                            0x97525 appname + 615717
6   CoreFoundation                      0x32e6d349 _CFXNotificationPost + 1420
7   Foundation                          0x337879cd  + 168
8   Foundation                          0x337876c1  + 136
9   appname                            0x96f2f appname + 614191
10  Foundation                          0x33858915  + 16
11  Foundation                          0x33798769  + 200
12  Foundation                          0x33798685  + 60
13  CFNetwork                           0x32bf964f  + 26
14  CFNetwork                           0x32bf8d33  + 54
15  CFNetwork                           0x32c21013  + 18
16  CoreFoundation                      0x32e62acd CFArrayApplyFunction + 176
17  CFNetwork                           0x32c21473  + 74
18  CFNetwork                           0x32b85461  + 188
19  CoreFoundation                      0x32ef18f7  + 14
20  CoreFoundation                      0x32ef115d  + 212
21  CoreFoundation                      0x32eeff2f  + 646
22  CoreFoundation                      0x32e6323d CFRunLoopRunSpecific + 356
23  CoreFoundation                      0x32e630c9 CFRunLoopRunInMode + 104
24  GraphicsServices                    0x36a4233b GSEventRunModal + 74
25  UIKit                               0x34d7f2b9 UIApplicationMain + 1120
26  appname                            0xf3df appname + 58335
27  appname                            0x3578 appname + 9592
)

dSYM UUID: 365EF56E-D598-3B94-AD36-BFA13772A4E3
CPU Type: armv7s
Slide Address: 0x00001000
Binary Image: appname
Base Address: 0x000f7000

```

–[__NSArrayM objectAtIndex:]: index 4294967295 beyond bounds for empty array

```
*** -[__NSArrayM objectAtIndex:]: index 4294967295 beyond bounds for empty array
(null)
(
0   CoreFoundation                      0x330dc3ff  + 186
1   libobjc.A.dylib                     0x3add7963 objc_exception_throw + 30
2   CoreFoundation                      0x33027ef9  + 164
3   appname                            0xcbcaf appname + 830639
4   appname                            0x40bc1 appname + 261057
5   appname                            0x3d297 appname + 246423
6   UIKit                               0x34f36569  + 408
7   UIKit                               0x34f1b391  + 1316
8   UIKit                               0x34f32827  + 206
9   UIKit                               0x34eee8c7  + 258
10  QuartzCore                          0x34c9a513  + 214
11  QuartzCore                          0x34c9a0b5  + 460
12  QuartzCore                          0x34c9afd9  + 16
13  QuartzCore                          0x34c9a9c3  + 238
14  QuartzCore                          0x34c9a7d5  + 316
15  QuartzCore                          0x34c9a639  + 60
16  CoreFoundation                      0x330b1941  + 20
17  CoreFoundation                      0x330afc39  + 276
18  CoreFoundation                      0x330aff93  + 746
19  CoreFoundation                      0x3302323d CFRunLoopRunSpecific + 356
20  CoreFoundation                      0x330230c9 CFRunLoopRunInMode + 104
21  GraphicsServices                    0x36c0233b GSEventRunModal + 74
22  UIKit                               0x34f3f2b9 UIApplicationMain + 1120
23  appname                            0xf3df appname + 58335
24  appname                            0x3578 appname + 9592
)

dSYM UUID: 365EF56E-D598-3B94-AD36-BFA13772A4E3
CPU Type: armv7s
Slide Address: 0x00001000
Binary Image: appname
Base Address: 0x000c3000

```

好了，相信你也看出来了，这些具体的crash log 什么都看不出来，都是一些内存地址，帧调用栈等，所以需要进一步的解析，看下文：

看一下上面的crash log，找到一句

```
5   appname                            0x97525 appname + 615717

```

它指出了应用名称，崩溃时的调用方法的地址，文件的地址以及方法所在的行的位置（[具体请看这篇文章](http://www.raywenderlich.com/zh-hans/30818/ios%E5%BA%94%E7%94%A8%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%97%E6%8F%AD%E7%A7%98)），接下来就要符号化（Symbolication）这句,用dwarfdump来检测crash log中dSYM UUID和本地的dSYM文件是否匹配

打开终端：

```
cd /Users/username/Library/Developer/Xcode/Archives/2013-08-30/app 8-30-13 6.19 PM.xcarchive/dSYMs
dwarfdump --uuid appname.app.dSYM
UUID: 9F0AEFA6-4349-30AF-8420-BCEE739DA0B4 (armv7) appname.app.dSYM/Contents/Resources/DWARF/appname
UUID: 365EF56E-D598-3B94-AD36-BFA13772A4E3 (armv7s) appname.app.dSYM/Contents/Resources/DWARF/appname

```

OK,crash log中的dSYM UUID与本地的dYSM文件是相匹配的。好接下来就查一下0x97525这个地址是什么，

```
dwarfdump --arch=armv7 --lookup 0x97525  /Users/username/Library/Developer/Xcode/Archives/2013-08-30/appname\ 8-30-13\ 6.19\ PM.xcarchive/dSYMs/appname.app.dSYM/Contents/Resources/DWARF/appname

```

得到的结果：

```
----------------------------------------------------------------------
File: /Users/username/Library/Developer/Xcode/  Archives/2013-08-30/appname 8-30-13 6.19    PM.xcarchive/dSYMs/appname.app.dSYM/Contents/   Resources/DWARF/appname (armv7)
----------------------------------------------------------------------
Looking up address: 0x0000000000097525 in .debug_info... found!

0x00359c67: Compile Unit: length = 0x000066f1  version = 0x0002  abbr_offset = 0x00000000  addr_size = 0x04  (next CU at 0x0036035c)

0x00359c72: TAG_compile_unit [1] *
         AT_producer( "Apple LLVM version 4.2 (clang-425.0.28) (based on LLVM 3.2svn)" )
         AT_language( DW_LANG_ObjC )
         AT_name( "xxx/EGOImageView.m" )
         AT_low_pc( 0x0009710c )
         AT_stmt_list( 0x000655c1 )
         AT_comp_dir( "xxx" )
         AT_APPLE_optimized( 0x01 )
         AT_APPLE_major_runtime_vers( 0x02 )

0x00359e57:     TAG_subprogram [10] *
             AT_name( "-[EGOImageView imageLoaderDidFailToLoad:]" )
             AT_decl_file( "xxx/EGOImageView.m" )
             AT_decl_line( 96 )
             AT_prototyped( 0x01 )
             AT_APPLE_isa( 0x01 )
             AT_low_pc( 0x00097490 )
             AT_high_pc( 0x00097572 )
             AT_frame_base( r7 )
             AT_object_pointer( {0x00359e6e} )
Line table dir : 'xxx'
Line table file: 'EGOImageView.m' line 99, column 2 with start address 0x00000000000974fe

Looking up address: 0x0000000000097525 in .debug_frame... found!

0x0000c620: FDE
    length: 0x0000000c
    CIE_pointer: 0x00000000
    start_addr: 0x00097490 -[EGOImageView imageLoaderDidFailToLoad:]
range_size: 0x000000e2 (end_addr = 0x00097572)
Instructions: 0x00097490: CFA=4294967295+4294967295

```

看一下结果：发现有AT_name、Line table dir :、Line table file:，aha!找到了出错的地方（出错的这个文件是网上别人写的，有bug，现已不再使用）。

### 注意：如果发现warning: unsupported file type:错误，这个问题是因为有文件或者目录的名称中包含空格，比如：2013-08-30/appname 8-30-13 6.19 ，所以，需要转义一下：2013-08-30/appname\ 8-30-13\ 6.19\ PM.xcarchive

（[参看此文章](http://www.whoslab.me/blog/?cat=24)）

OK,希望能有所帮助，到此为止吧

