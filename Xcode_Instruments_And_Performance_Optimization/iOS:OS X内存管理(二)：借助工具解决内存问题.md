**原文链接：** [iOS/OS X内存管理(二)：借助工具解决内存问题](http://www.jianshu.com/p/09c5141d4531)

**Keywords:** `iOS`，`内存管理`，`性能优化`，`Instruments`

# iOS/OS X内存管理(二)：借助工具解决内存问题

上一篇博客[iOS/OS X内存管理(一)：基本概念与原理](http://www.jianshu.com/p/1928b54e1253)主要讲了iOS/OSX 内存管理中引用计数和内存管理规则，以及引入ARC新的内存管理机制之后如何选择ownership qualifiers(`__strong`、`__weak`、`__unsafe_unretained`和`__autoreleasing`)来管理内存。这篇我们主要关注在实际开发中会遇到哪些内存管理问题，以及如何使用工具来调试和解决。

![](http://upload-images.jianshu.io/upload_images/166109-00c90f0f030c3665.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在往下看之前请下载实例[MemoryProblems](https://github.com/samlaudev/MemoryProblems)，我们将以这个工程展开如何检查和解决内存问题。

# 悬挂指针问题

悬挂指针([Dangling Pointer](https://en.wikipedia.org/wiki/Dangling_pointer))就是当指针指向的对象已经释放或回收后，但没有对指针做任何修改(一般来说，将它指向空指针)，而是仍然指向原来已经回收的地址。如果指针指向的对象已经释放，但仍然使用，那么就会导致程序crash。

当你运行MemoryProblems后，点击**悬挂指针**那个选项，就会出现`EXC_BAD_ACCESS`崩溃信息

![](http://upload-images.jianshu.io/upload_images/166109-14751cda6424d749.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看看这个`NameListViewController`是做什么的？它继承`UITableViewController`，主要显示多个名字的信息。它的实现文件如下：

```
static NSString *const kNameCellIdentifier = @"NameCell";

@interface NameListViewController ()

#pragma mark - Model
@property (strong, nonatomic) NSArray *nameList;

#pragma mark - Data source
@property (assign, nonatomic) ArrayDataSource *dataSource;

@end

@implementation NameListViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    self.tableView.dataSource = self.dataSource;
}

#pragma mark - Lazy initialization
- (NSArray *)nameList
{
    if (!_nameList) {
        _nameList = @[@"Sam", @"Mike", @"John", @"Paul", @"Jason"];
    }
    return _nameList;
}

- (ArrayDataSource *)dataSource
{
    if (!_dataSource) {
        _dataSource = [[ArrayDataSource alloc] initWithItems:self.nameList
                                              cellIdentifier:kNameCellIdentifier
                                              tableViewStyle:UITableViewCellStyleDefault
                                          configureCellBlock:^(UITableViewCell *cell, NSString *item, NSIndexPath *indexPath) {
            cell.textLabel.text = item;
        }];
    }
    return _dataSource;
}

@end
```

要想通过tableView显示数据，首先要实现`UITableViewDataSource`这个协议，为了瘦身controller和复用data source，我将它分离到一个类`ArrayDataSource`来实现`UITableViewDataSource`这个协议。然后在`viewDidLoad`方法里面将`dataSource`赋值给`tableView.dataSource`。

解释完`NameListViewController`的职责后，接下来我们需要思考出现`EXC_BAD_ACCESS`错误的**原因和位置**信息。

一般来说，出现`EXC_BAD_ACCESS`错误的原因都是**悬挂指针**导致的，但具体是哪个指针是悬挂指针还不确定，因为控制台并没有给出具体crash信息。

### 启用NSZombieEnabled

要想得到更多的crash信息，你需要启动`NSZombieEnabled`。具体步骤如下：

1. 选中`Edit Scheme`，并点击

    ![](http://upload-images.jianshu.io/upload_images/166109-f4e0337f766e1e89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
2. Run -> Diagnostics -> Enable Zombie Objects

    ![](http://upload-images.jianshu.io/upload_images/166109-ae4f6b55212b75a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
设置完之后，再次运行和点击悬挂指针，虽然会再次crash，但这次**控制台**打印了以下有用信息：

![](http://upload-images.jianshu.io/upload_images/166109-9fe90d621bf6ce06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

信息`message sent to deallocated instance 0x7fe19b081760`大意是向一个已释放对象发送信息，也就是已释放对象还调用某个方法。现在我们大概知道什么原因导致程序会crash，但是具体哪个对象被释放还仍然使用呢？

点击上面红色框的`Continue program execution`按钮继续运行，截图如下：

![](http://upload-images.jianshu.io/upload_images/166109-654444b25d8c5155.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

留意上面的两个红色框，它们两个地址是一样，而且`ArrayDataSource`前面有个`_NSZombie_`修饰符，说明`dataSource`对象被释放还仍然使用。

再进一步看`dataSource`声明属性的修饰符是`assign`

```
#pragma mark - Data source
@property (assign, nonatomic) ArrayDataSource *dataSource;
```

而`assign`对应就是`__unsafe_unretained`，它跟__weak相似，被它修饰的变量都不持有对象的所有权，但当变量指向的对象的RC为0时，变量并不设置为nil，而是继续保存对象的地址。

因此，在`viewDidLoad`方法中

```
- (void)viewDidLoad {
    [super viewDidLoad];

    self.tableView.dataSource = self.dataSource;    
    /*  由于dataSource是被assign修饰，self.dataSource赋值后，它对象的对象就马上释放，
     *  而self.tableView.dataSource也不是strong，而是weak，此时仍然使用，所有会导致程序crash
     */
}
```

分析完原因和定位错误代码后，至于如何修改，我想大家都心知肚明了，如果还不知道的话，留言给我。

# 内存泄露问题

还记得上一篇[iOS/OS X内存管理(一)：基本概念与原理](http://www.jianshu.com/p/1928b54e1253)的引用循环例子吗？它会导致内存泄露，上次只是文字描述，不怎么直观，这次我们尝试使用**Instruments**里面的子工具**Leaks**来检查内存泄露。

### 静态分析

一般来说，在程序未运行之前我们可以先通过[Clang Static Analyzer](http://clang-analyzer.llvm.org/)(静态分析)来检查代码是否存在bug。比如，内存泄露、文件资源泄露或访问空指针的数据等。下面有个静态分析的例子来讲述如何启用静态分析以及静态分析能够查找哪些bugs。

启动程序后，点击静态分析，马上就出现crash

![](http://upload-images.jianshu.io/upload_images/166109-036d86de2b9e9424.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时，即使启用*NSZombieEnabled*，控制台也不能打印出更多有关bug的信息，具体原因是什么，等下会解释。

打开`StaticAnalysisViewController`，里面引用[Facebook Infer](http://fbinfer.com/)工具的代码例子，包含个人日常开发中会出现的bugs：

```
@implementation StaticAnalysisViewController

#pragma mark - Lifecycle
- (void)viewDidLoad
{
    [super viewDidLoad];

    [self memoryLeakBug];
    [self resoureLeakBug];
    [self parameterNotNullCheckedBlockBug:nil];
    [self npeInArrayLiteralBug];
    [self prematureNilTerminationArgumentBug];
}

#pragma mark - Test methods from facebook infer iOS Hello examples
- (void)memoryLeakBug
{
     CGPathRef shadowPath = CGPathCreateWithRect(self.inputView.bounds, NULL);
}

- (void)resoureLeakBug
{
    FILE *fp;
    fp=fopen("info.plist", "r");
}

-(void) parameterNotNullCheckedBlockBug:(void (^)())callback {
    callback();
}

-(NSArray*) npeInArrayLiteralBug {
    NSString *str = nil;
    return @[@"horse", str, @"dolphin"];
}

-(NSArray*) prematureNilTerminationArgumentBug {
    NSString *str = nil;
    return [NSArray arrayWithObjects: @"horse", str, @"dolphin", nil];
}

@end
```

下面我们通过静态分析来检查代码是否存在bugs。有两个方式：

* 手动静态分析：每次都是通过点击菜单栏的*Product* -> *Analyze*或快捷键*shift + command + b*

![](http://upload-images.jianshu.io/upload_images/166109-a890797a4457159d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 自动静态分析：在Build Settings启用**Analyze During 'Build'**，每次编译时都会自动静态分析

    ![](http://upload-images.jianshu.io/upload_images/166109-5c1dcdd871fcb891.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
**静态分析结果如下：**

![](http://upload-images.jianshu.io/upload_images/166109-6c032a57f0fef09b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过静态分析结果，我们来分析一下为什么*NSZombieEnabled*不能定位`EXC_BAD_ACCESS`的错误代码位置。由于callback传入进来的是null指针，而*NSZombieEnabled*只能针对某个已经释放对象的地址，所以启动*NSZombieEnabled*是不能定位的，不过可以通过静态分析可得知。

### 启动Instruments

有时使用**静态分析**能够检查出一些内存泄露问题，但是有时只有**运行时**使用Instruments才能检查到，启动Instruments**步骤如下**：

1. 点击Xcode的菜单栏的 *Product* -> *Profile* 启动Instruments

    ![](http://upload-images.jianshu.io/upload_images/166109-95b4ea305007d321.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
2. 此时，出现Instruments的工具集，选中**Leaks**子工具点击

    ![](http://upload-images.jianshu.io/upload_images/166109-379b199e81584b16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
3. 打开**Leaks**工具之后，点击红色圆点按钮启动`Leaks`工具，在Leaks工具启动同时，模拟器或真机也跟着启动

    ![](http://upload-images.jianshu.io/upload_images/166109-03e04393903c0c6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
4. 启动**Leaks**工具后，它会在**程序运行时**记录内存分配信息和检查是否发生内存泄露。当你点击**引用循环**进去那个页面后，再返回到主页，就会发生内存泄露

![内存泄露.gif](http://upload-images.jianshu.io/upload_images/166109-1148d40299015b5f.gif?imageMogr2/auto-orient/strip)

![](http://upload-images.jianshu.io/upload_images/166109-7dca560ad71e2273.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果发生内存泄露，我们怎么**定位**哪里发生和**为什么**会发生内存泄露？

### 定位内存泄露

借助Leaks能很快定位内存泄露问题，在这个例子中，步骤如下：

* 首先点击**Leak Checks**时间条那个红色叉

    ![](http://upload-images.jianshu.io/upload_images/166109-02b6904da2ff94f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
* 然后双击某行内存泄露调用栈，会直接跳到内存泄露代码位置

    ![](http://upload-images.jianshu.io/upload_images/166109-e157b6d2c837daa2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 分析内存泄露原因

上面已经定位好内存泄露代码的位置，至于原因是什么？可以查看上一篇的[iOS/OS X内存管理(一)：基本概念与原理](http://www.jianshu.com/p/1928b54e1253)的循环引用例子，那里已经有详细的解释。

# 难以检测Block引用循环

大多数的内存问题都可以通过静态分析和Instrument Leak工具检测出来，但是有种block引用循环是难以检测的，看我们这个**Block内存泄露**例子，跟上面的**悬挂指针**例子差不多，只是在`configureCellBlock`里面调用一个方法`configureCell`。

```
- (ArrayDataSource *)dataSource
{
    if (!_dataSource) {
        _dataSource = [[ArrayDataSource alloc] initWithItems:self.nameList
                                              cellIdentifier:kNameCellIdentifier
                                              tableViewStyle:UITableViewCellStyleDefault
                                          configureCellBlock:^(UITableViewCell *cell, NSString *item, NSIndexPath *indexPath) {
                                              cell.textLabel.text = item;

                                              [self configureCell];
                                          }];
    }
    return _dataSource;
}

- (void)configureCell
{
    NSLog(@"Just for test");
}

- (void)dealloc
{
    NSLog(@"release BlockLeakViewController");
}
```

我们首先用**静态分析**来看看能不能检查出内存泄露：

![](http://upload-images.jianshu.io/upload_images/166109-c9f8a4c970462eb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

结果是**没有**任何内存泄露的提示，我们再用Instrument Leak工具在运行时看看能不能检查出：

![](http://upload-images.jianshu.io/upload_images/166109-68e795cea155fd8e.gif?imageMogr2/auto-orient/strip)

结果跟使用静态分析一样，还是没有任何内存泄露信息的提示。

那么我们怎么知道这个`BlockLeakViewController`发生了内存泄露呢？还是根据iOS/OS X内存管理机制的一个**基本原理**：当某个对象的引用计数为0时，它就会自动调用`- (void)dealloc`方法。

在这个例子中，如果`BlockLeakViewController`被navigationController pop出去后，没有调用`dealloc`方法，说明它的某个属性对象仍然被持有，未被释放。而我在`dealloc`方法打印**release BlockLeakViewController**信息：

```
- (void)dealloc
{
    NSLog(@"release BlockLeakViewController");
}
```

在我点击返回按钮后，其并没有打印出来，因此这个`BlockLeakViewController`存在内存泄露问题的。至于如何解决block内存泄露这个问题，很多基本功扎实的同学都知道如何解决，不懂的话，自己查资料解决吧！

# 总结

一般来说，在创建工程的时候，我都会在Build Settings启用Analyze During 'Build'，每次编译时都会自动静态分析。这样的话，写完一小段代码之后，就马上知道是否存在内存泄露或其他bug问题，并且可以修bugs。而在运行过程中，如果出现`EXC_BAD_ACCESS`，启用NSZombieEnabled，看出现异常后，控制台能否打印出更多的提示信息。如果想在运行时查看是否存在内存泄露，使用Instrument Leak工具。但是有些内存泄露是很难检查出来，有时只有通过手动覆盖`dealloc`方法，看它最终有没有调用。

