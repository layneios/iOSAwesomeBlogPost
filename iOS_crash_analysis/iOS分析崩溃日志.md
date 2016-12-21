**原文链接：**[iOS 分析崩溃日志](http://www.jianshu.com/p/727eff8b5404)

**Keywords:** `iOS`，`崩溃解析`，`crash`

# iOS分析崩溃日志

## **前言**

  iOS分析定位崩溃问题有很多种方式，但是发布到AppStore的应用如果崩溃了，我们该怎么办呢？通常我们都会在系统中接入统计系统，在系统崩溃的时候记录下崩溃日志，下次启动时将日志发送到服务端，比较好的第三方有umeng之类的。今天我们来讲一下通过崩溃日志来分析定位我们的bug。

## **dYSM文件**

  分析崩溃日志的前提是我们需要有**dYSM文件**，这个文件是我们用archive打包时生成的**.xcarchive**后缀的文件包。一个良好的习惯是，在你打包提交审核的时候，将生成的.xcarchive与ipa文件一同拷贝一份，按照版本号保存下来，这样如果线上出现问题可以快速的找到你想要的文件来定位你的问题。

![](http://upload-images.jianshu.io/upload_images/2030896-908e107a5818a2e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## **崩溃日志**

一般崩溃日志都会像下面这样:

```
NSConcreteMutableAttributedString addAttribute:value:range:: nil value
(null)
((
    CoreFoundation                      0x0000000185c642f4 <redacted> + 160
    libobjc.A.dylib                     0x00000001974880e4 objc_exception_throw + 60
    CoreFoundation                      0x0000000185c64218 <redacted> + 0
    Foundation                          0x0000000186a9dfb4 <redacted> + 152
    Xmen                                0x10073fb30 Xmen + 7600944
    Xmen                                0x1006bbbf4 Xmen + 7060468
    UIKit                               0x000000018a9a47fc <redacted> + 60
    UIKit                               0x000000018a9a512c <redacted> + 104
    UIKit                               0x000000018a6b2b6c <redacted> + 88
    UIKit                               0x000000018a9a4fd4 <redacted> + 444
    UIKit                               0x000000018a78e274 <redacted> + 1012
    UIKit                               0x000000018a999aac <redacted> + 2904
    UIKit                               0x000000018a785268 <redacted> + 172
    UIKit                               0x000000018a6a1760 <redacted> + 580
    QuartzCore                          0x0000000189fe9e1c <redacted> + 152
    QuartzCore                          0x0000000189fe4884 <redacted> + 320
    QuartzCore                          0x0000000189fe4728 <redacted> + 32
    QuartzCore                          0x0000000189fe3ebc <redacted> + 276
    QuartzCore                          0x0000000189fe3c3c <redacted> + 528
    QuartzCore                          0x0000000189fdd364 <redacted> + 80
    CoreFoundation                      0x0000000185c1c2a4 <redacted> + 32
    CoreFoundation                      0x0000000185c19230 <redacted> + 360
    CoreFoundation                      0x0000000185c19610 <redacted> + 836
    CoreFoundation                      0x0000000185b452d4 CFRunLoopRunSpecific + 396
    GraphicsServices                    0x000000018f35b6fc GSEventRunModal + 168
    UIKit                               0x000000018a70afac UIApplicationMain + 1488
    Xmen                                0x1008cf9c0 Xmen + 9238976
    libdyld.dylib                       0x0000000197b06a08 <redacted> + 4
)
dSYM UUID: 30833A40-0F40-3980-B76B-D6E86E4DBA85
CPU Type: arm64
Slide Address: 0x0000000100000000
Binary Image: Xmen
Base Address: 0x000000010007c000
```

是不是看的一脸懵逼，下面我来教大家如何结合crash log 与 dYSM文件 来分析定位出代码崩溃在哪一个文件的哪一行代码

## **提取崩溃日志中有用的信息**

* NSConcreteMutableAttributedString addAttribute:value:range:: nil value 崩溃的原因是value为nil
* " 4 Xmen 0x10073fb30 Xmen + 7600944" 它指出了应用名称，崩溃时的调用方法的地址，文件的地址以及方法所在的行的位置，我们需要的是这一个:"0x10073fb30"。
* "dSYM UUID: 30833A40-0F40-3980-B76B-D6E86E4DBA85"。
* "CPU Type: arm64"。

## **开始分析**

* 打开终端进入到你的dYSM文件的目录下面:
    `cd /Dandy/XMEN/上线版本/2.0.17_105/aaaa.xcarchive/dSYMs`
* 验证下崩溃日志中的UUID与本地的dYSM文件是否是相匹配的：
    "Xmen"为你的app名称
    `dwarfdump --uuid Xmen.app.dSYM`
    结果是:

    ```
    UUID: BFF6AE00-8B5F-39BD-AFD0-27707C489B25 (armv7) Xmen.app.dSYM/Contents/Resources/DWARF/Xmen
    UUID: 30833A40-0F40-3980-B76B-D6E86E4DBA85 (arm64) Xmen.app.dSYM/Contents/Resources/DWARF/Xmen
    ```

    发现与我们日志中的:UUID和CPU Type是相匹配的
* 查找错误信息
    `dwarfdump --arch=arm64 --lookup 0x10073fb30 /Dandy/XMEN/上线版本/2.0.17_105/aaaa.xcarchive/dSYMs/Xmen.app.dSYM/Contents/Resources/DWARF/Xmen`
    "arm64"与"0x1008cf9c0"分别对应于上面我们从日志中提取出来的有用信息
    "/Dandy/XMEN/上线版本/2.0.17_105/aaaa.xcarchive/dSYMs/Xmen.app.dSYM/Contents/Resources/DWARF/Xmen"对应于你本地dYSM文件目录
    "Xmen"对应于你的app名称
    结果像下面这样:

    ```
    File: /Dandy/XMEN/上线版本/2.0.17_105/aaaa.xcarchive/dSYMs/Xmen.app.dSYM/Contents/Resources/DWARF/Xmen (arm64)
    Looking up address: 0x000000010073fb30 in .debug_info... found!
    0x00219b05: Compile Unit: length = 0x00003dd0  version = 0x0002  abbr_offset = 0x00000000  addr_size = 0x08  (next CU at 0x0021d8d9)
    0x00219b10: TAG_compile_unit [107] *
    AT_producer( "Apple LLVM version 8.0.0 (clang-800.0.42.1)" )
    AT_language( DW_LANG_ObjC )
    AT_name( "/Dandy/checkSvn/iOS/xmen/Xmen/Modules/StoreManage/SellerOrder/View/DSSellerOrderSectionHeaderView.m" )
    AT_stmt_list( 0x001272a9 )
    AT_comp_dir( "/Dandy/checkSvn/iOS/xmen" )
    AT_APPLE_major_runtime_vers( 0x02 )
    AT_low_pc( 0x000000010072b8ac )
    AT_high_pc( 0x000000010074e350 )
    0x0021aec5:    TAG_subprogram [119] *
    AT_low_pc( 0x0000000100739810 )
    AT_high_pc( 0x000000010074006c )
    AT_frame_base( reg29 )
    AT_object_pointer( {0x0021aee3} )
    AT_name( "-[DSSellerOrderSectionHeaderView updateContentWithOrderData:isEdit:]" )
    AT_decl_file( "/Dandy/checkSvn/iOS/xmen/Xmen/Modules/StoreManage/SellerOrder/View/DSSellerOrderSectionHeaderView.m" )
    AT_decl_line( 248 )
    AT_prototyped( 0x01 )
    0x0021af36:        TAG_lexical_block [138] *
    AT_ranges( 0x00008640
    [0x000000010073cf90 - 0x000000010073fb88)
    [0x000000010073fbc0 - 0x000000010073fbc4)
    End )
    Line table dir : '/Dandy/checkSvn/iOS/xmen/Xmen/Modules/StoreManage/SellerOrder/View'
    Line table file: 'DSSellerOrderSectionHeaderView.m' line 680, column 9 with start address 0x000000010073faf8
    Looking up address: 0x000000010073fb30 in .debug_frame... not found.
    ```

    我们来从终端结果来分析出我们想要的结果:
    这一行告诉我们崩溃的代码所在的文件的目录

    ```
    Line table dir : '/Dandy/checkSvn/iOS/xmen/Xmen/Modules/StoreManage/SellerOrder/View'
    ```

    这一行告诉我们崩溃代码所在的具体文件

    ```
    AT_decl_file( "/Dandy/checkSvn/iOS/xmen/Xmen/Modules/StoreManage/SellerOrder/View/DSSellerOrderSectionHeaderView.m" )
    ```

    这一行告诉我们崩溃代码是在哪一个方法里面

    ```
    AT_name( "-[DSSellerOrderSectionHeaderView updateContentWithOrderData:isEdit:]" )
    ```

    这一行告诉我们崩溃代码具体在哪一行了

    ```
    Line table file: 'DSSellerOrderSectionHeaderView.m' line 680, column 9 with start address 0x000000010073faf8
    ```

    好的，现在我们分析到了崩溃代码在哪一行了，下面我们来找一找bug

## **查找bug**

  我们的代码都应该有托管平台，每个版本上线都需要打一个**tag**，这是一个好习惯。下面我拉下我崩溃的对应版本的tag，找到崩溃代码那一行：

![](http://upload-images.jianshu.io/upload_images/2030896-46b84b37cf488cf1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  结合崩溃日志中告诉我:崩溃的原因是value为nil,发现是因为receiverTelephone字段中有中文导致转url时为nil导致的，下面的解bug就看各自本领啦。

## **结语**

  希望对您有帮助，谢谢支持~

