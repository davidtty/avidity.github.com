---
layout: post
title: Objective-C 知识干货
date: 2015-05-12
category: "ios"
---

###Object-C runtime
Object-C动态运行时特性是用C和汇编实现的,可以使得程序在运行时才去创建，访问，修改类或对象的属性，方法或协议。

###对象、类和元类
![img](http://zrongl.github.io/images/post/2015-05-12-ios-objectiveC-dry-cargo-img01.jpg)

###Method、SEL和IMP
Mehtod是一个函数整体的总称，包括方法名，返回值，参数及实现。<br/>
SEL本质上是`char *`指针，代表方法的名称。<br/>
IMP本质上是函数指针，指向方法的地址。<br/>
每一个Object-C对象结构中都维护有一个方法列表，其中方法是以key-value的形式存在的，key就是SEL，而value则是IMP，所以OC中同一对象中不能存在名字相同的方法，即使它们有不同的传入参数。<br/>

###消息传递
自己类的cacheList/mehtodList-><br />
父类们的cacheList/mehtodList-><br />
动态方法解析resolveInstanceMethod-><br />
接受者重定向forwardingTargetForSelector-><br />
最后的转发methodSignatureForSelector<br />

####[DeepIntoRuntime](https://github.com/zrongl/DeepIntoRuntime)
####[详情请参阅](https://github.com/zrongl/DeepIntoRuntime/blob/master/OCRuntime.pdf)

###Category
可以动态地为已经存在的类添加新的方法，这样可以保证类的原始设计规模较小，功能增加时再逐步扩展。<br />
**加载时机：**<br/>
1.打开objc源代码，找到 objc-os.mm, 函数`_objc_init`为runtime的加载入口，由libSystem调用，进行初始化操作。<br />
2.之后调用objc-runtime-new.mm -> `map_images`加载map到内存<br />
3.之后调用objc-runtime-new.mm-> `_read_images`初始化内存中的map, 这个时候将会load所有的类，协议还有Category。NSObject的`+load`方法就是这个时候调用的<br />

###Associate
通过**类别**和**访问器**，再结合**联合存储**技术，我们可以为类增加属性。即在categray文件中声明属性及其访问器，并用`objc_setAssociatedObject`和`objc_getAssociatedObject`方法去设置和读取属性。<br />
**实现原理：**<br />
1.有一个单例的`AssociationsHashMap`实例<br />
2.`AssociationsHashMap`实例用于保存一个个的`ObjectAssociationMap`对象<br />
3.每个类都拥有一个`ObjectAssociationMap`实例，每个类通过联合存储模式保存的键值对也都保存在`ObjectAssociationMap`实例中。<br />
4.Key对应的值无所谓，我们需要的是key的地址。<br />

####[DeepIntoCategory](https://github.com/zrongl/DeepIntoCatgory)

###KVO
键值观察建立对对象成员变量的观察，当变量值发生改变时会触发相应的观察事件<br />
**实现原理:**<br />
当某个类的对象第一次被观察时，**系统就会在运行期动态地创建该类的一个派生类**，在这个派生类中重写基类中任何被观察属性的 setter 方法。
**派生类在被重写的 setter 方法实现真正的通知机制**，就如前面手动实现键值观察那样。这么做是基于设置属性会调用 setter 方法，而通过重写就获得了 KVO 需要的通知机制。当然前提是要通过遵循 KVO 的属性设置方式来变更属性值，如果仅是直接修改属性对应的成员变量，是无法实现 KVO 的。
同时派生类还重写了 class 方法以“欺骗”外部调用者它就是起初的那个类。然后系统将这个对象的 isa 指针指向这个新诞生的派生类，因此这个对象就成为该派生类的对象了，因而在该对象上对 setter 的调用就会调用重写的 setter，从而激活键值通知机制。此外，派生类还重写了 dealloc 方法来释放资源。<br />
**`NSNotification的通知回调会在发出post的线程中同步执行`**

####[DeepIntoKVO](https://github.com/zrongl/DeepIntoKVO)

###内存管理
object-C是用引用计数来维护和管理内存，遵循谁创建谁释放的原则。
arc并非其他语言的垃圾回收器，它只是将MRC中需要手动添加的`retian/release/autorelease`方法的过程交给编译去做了相应的处理。<br />
对于`@property`属性的理解：<br />
`retain/assign/strong/weak/unsafe_unretained/copy`<br />
retain/strong可以持有对象，使得对象的引用计数+1；assign/weak/unsafe_unretained只是指向对象，不会引起引用计数的变化;
<pre><code>// ARC环境下
// obj默认被__strong修饰，此时它持有对象 对象的引用计数为1
id obj = [[NSObject alloc] init];
// weak_obj只是指向对象
id __weak weak_obj = obj;
// obj此时失去对对象的持有 对象的引用计数为0 此时系统会回收该对象
obj = nil;
// 由于weak_obj指向的对象已经被销毁 
// 此时系统将被__weak修饰的变量weak_obj赋值为nil
// 如果weak_obj被assign或unsafe_unretained修饰 系统不会将它的值赋为nil 此时weak_obj会变成野指针 是不安全的
NSLog(@"%@", weak_obj);
</code></pre>
`autorelease` 自动释放池<br />
`bridge` void * 与 id之间的转换

####[DeepIntoMemory](https://github.com/zrongl/DeepIntoMemory)

###Block
Block又称匿名函数，他的核心是一段可执行的代码，你可以给它传递参数或获取返回值。<br />
在内存中Block分三种类型：<br />
•`_NSConcreteGlobalBlock` 全局的静态block，不会访问任何外部变量<br />
•`_NSConcreteStackBlock` 保存在栈中的block，出栈时会被销毁<br />
*a.*在局部作用域中声明的block需要在作用域之外调用时，需要对该block进行copy操作，将该block拷贝找堆上，并将其类型转换为`_NSConcreteMallocBlock`类型<br />
*b.*需要对block外的变量进行修改时需要添加__block修饰，当系统将block复制到堆上时，该变量也会一同被拷贝到堆上<br />
*c.*__block修饰符可以解除Block循环引用的问题
<pre><code>__block id tmp = self;
void(^block)(void) = ^{
    tmp = nil;
};
block();</code></pre>
在不允许使用__weak修饰符的情况下，避免使用__unsafe_unretained修饰符，但是前提是Block必须被执行。<br />
•`_NSConcreteMallocBlock` 保存在堆中的block，当引用计数为0时会被销毁<br />

####[DeepIntoBlock](https://github.com/zrongl/DeepIntoBlock)

###Runloop
runloop是iOS中实现的一种事件驱动模型：<br />
**主线程**的runloop默认是启动的，它主要执行更新用户界面的操作。<br />
**子线程**的runloop默认是关闭的，所以在子线程的runloop中添加了一个时间源后，需要手动启动子线程的runloop才能对时间源进行监听和触发。如果想保持子线程长时间运行不退出，可以启动线程的runloop向其中添加一些长时间或周期性的事件源，如：`performSelecter``NSTimer``NSURLConection`等<br />

iOS系统中的Runloop可以运行在以下几种模式中：<br />
`NSRunLoopDefaultMode` 默认模式中几乎包含了所有输入源(NSConnection除外),一般情况下应使用此模式。<br />
`NSRunLoopCommonModes` 这是一个伪模式，其为一组run loop mode的集合，将输入源加入此模式意味着在Common Modes中包含的所有输入源都可以处理。<br />
`UITrackingRunLoopMode` 用户界面拖动操作时处于此种模式下，在此模式下会限制输入事件的处理。例如，当手指按住UITableView拖动时就会处于此模式。

####[DeepIntoRunloop](https://github.com/zrongl/DeepIntoRunloop)

###Thread

#####GCD
*despatch queue按照任务执行顺序分为：<br />
串行队列(serial queue)：串行队列中的任务是顺序执行的(所有任务在一个线程中按顺序执行)<br />
并行队列(concurrent queue)：并行队列中的任务是并发执行的(同时有多个线程并发执行任务)*

dispatch queue分为三种：<br />
**main queue** 属于串行队列，运行主循环runloop，一般执行与界面显示有关的操作需要放到主线程队列中去执行。<br />
**global queue** 属于并行队列，可以通过`dispatch_get_global_queue`函数获取不同优先级(`DISPATCH_QUEUE_PRIORITY_HIGH`，
`DISPATCH_QUEUE_PRIORITY_DEFAULT`，
`DISPATCH_QUEUE_PRIORITY_LOW`，
`DISPATCH_QUEUE_PRIORITY_BACKGROUND`)的全局队列<br />
**custom queue** 通过`dispatch_queue_create`方法创建的队列 ，通过参数可获取指定类型(`DISPATCHQUEUESERIAL`，
`DISPATCHQUEUECONCURRENT`)的队列，用户队列中的任务最终都会被安排到全局队列中去执行。<br />
**相关方法：**<br />
`dispatch_async` 并行执行任务。<br />
`dispatch_sync` 串行执行任务。<br />
`dispatch_after` 延迟执行任务。<br />
`dispatch_apply` 并行执行多次任务。<br />
`dispatch_group` 用于任务调度，可实现等待放入group中的每个queue中所有任务都执行完之后再继续向下执行。<br />
`dispatch_barrier_async` 用于任务调度，可实现等待前几个任务并行执行完成后，单独执行下面一条任务，等这条执行完成后，再并行执行后面的任务。<br />
`dispatch_source` 用于监听系统的源，然后触发相应的处理。<br />

#####NSOperation
`main`与`start`<br />默认情况下`main`函数不会做任何事情，如果只是执行一些简单的任务只需要重载`main`函数即可；但是如果要执行一些复杂的并发操作需要重载`start`并手动更新operation的状态(excting，finish等)然后在`start`函数中进行一些operation状态的检查，满足条件后再调用`main`执行操作。<br /><br />

*GCD与NSOperation的比较：<br />*
*1.* `GCD`是底层的C语言构成的API，而`NSOperationQueue`及相关对象是Objc的对象。在GCD中，在队列中执行的是由block构成的任务，这是一个轻量级的数据结构；而`NSOperation`作为一个对象，为我们提供了更多的选择；<br />
*2.* 在`NSOperationQueue`中，我们可以随时取消已经设定要准备执行的任务(当然，已经开始的任务就无法阻止了)，而GCD没法停止已经加入queue的block(其实是有的，但需要许多复杂的代码)；<br />
*3.* `NSOperation`能够方便地设置依赖关系，我们可以让一个Operation依赖于另一个Operation，这样的话尽管两个Operation处于同一个并行队列中，但前者会直到后者执行完毕后再执行；<br />
*4.* 我们能将KVO应用在`NSOperation`中，可以监听一个Operation是否完成或取消，这样子能比`GCD`更加有效地掌控我们执行的后台任务；<br />
*5.* 在`NSOperation`中，我们能够设置`NSOperation`的priority优先级，能够使同一个并行队列中的任务区分先后地执行，而在GCD中，我们只能区分不同任务队列的优先级，如果要区分block任务的优先级，也需要大量的复杂代码；<br />
*6.* 我们能够对`NSOperation`进行继承，在这之上添加成员变量与成员方法，提高整个代码的复用度，这比简单地将block任务排入执行队列更有自由度，能够在其之上添加更多自定制的功能。<br />
*7.* `NSOperationQueue`将任务装载进主线程的队列中，runloop每执行一个循环都会取出队列中的一个任务去执行，这样使得在每两个operation被执行之间可以允许进行更新视图的操作。对于`dispatch_async`和`performSelectorOnMainThread`，则会将装载进队列中的block或selector一个接一个的执行，而不会插入更新视图或其他类似的操作。

#####NSThread

####[DeepIntoThread](https://github.com/zrongl/DeepIntoThread)

###iOS沙盒目录及其用途
下图为出自[官方文档](https://developer.apple.com/library/mac/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40010672-CH1-SW1)

![img](http://zrongl.github.io/images/post/2015-05-12-ios-objectiveC-dry-cargo-img02.png)

####MyApp.app
存储应用程序本身的数据，包括<font color="#ff0000">资源文件和可执行文件</font>等，并且该目录为只读的。程序启动以后，会根据需要从该目录中动态加载代码或资源到内存，这里用到了lazy loading的思想。<br />

####Documents
存储应用程序的数据文件，并且这些数据应该是<font color="#ff0000">不可再生的</font>。<br />
*Documents/Inbox*<br />
<font color="#ff0000">存储由外部应用请求当前应用程序打开的文件</font>。

####Library
苹果建议用来存放<font color="#ff0000">默认设置或其它状态信息</font>。除Caches子目录外，其他目录会被iTunes同步。<br />
*Library/Caches*<br />
存储<font color="#ff0000">可再生的应用程序的数据文件</font>，如网络请求的数据，并且应用程序负责删除该目录下的文件。<br />
*Library/Preferences*<br />
存储应用程序的<font color="#ff0000">偏好设置文件</font>，程序中使用NSUserDefaults写的数据会以plist的文件形式存储在该目录下。<br />

####Temp
保存各种<font color="#ff0000">临时文件</font>，即应用再次启动时不需要的文件，该目录下的东西随时(比如系统磁盘空间不足时)有可能<font color="#ff0000">被系统清理掉</font>。
