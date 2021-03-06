---
title: 分析实现-伪单例设计
tags:
- 分析实现
---

首先，本文大概率并非实用性文章，即是说你读了本文基本不会带来任何技术上的提升。纯粹出于笔者学习过程中发现的有趣的知识点，然后结合这些技术点的尝试的总结。

> 何谓`伪单例设计`

即实现了与单例相同的功能，但是却和正常开发中的单例有着不一样的代码设计。虽然严格的按照单例思想来看的话，两种设计都能被称作单例，但是为了区分两者，我称这种方式为`伪单例设计`。本文中涉及到两种`伪单例设计`方案，包括`extern`和`弱符号`两种方式。

## extern
> extern可以置于变量或者函数前，以标示变量或者函数的定义在别的文件中，提示编译器遇到此变量和函数时在其他模块中寻找其定义。

`extern`是一个非常有趣的修饰符，允许我们在不依赖某个文件或者外部模块的情况下使用这些外部的函数或者变量。也就是说在不同的文件中我们可以访问同一个变量、函数，并且还不用引入头文件产生依赖关系。因此来说这些`extern`修饰的变量几乎都是`static`类型的，拥有和应用一样的生命周期。

比如此前我做的平滑度监控方案中需要监听应用进入前后台，但是又不想导入`UIKit`，通过`extern`声明相关的通知名称就可以使用：

    extern NSString *UIApplicationDidEnterBackgroundNotification;
    extern NSString *UIApplicationWillEnterForegroundNotification;
    
    - (instancetype)init {
        if (self = [super init]) {
            [[NSNotificationCenter defaultCenter] addObserver: self selector: @selector(didReceiveApplicationDidEnterBackgroundNotification:) name: UIApplicationDidEnterBackgroundNotification object: nil];
            [[NSNotificationCenter defaultCenter] addObserver: self selector: @selector(didReceiveApplicationWillEnterForegroundNotification:) name: UIApplicationWillEnterForegroundNotification object: nil];
        }
        return self;
    }

那么`extern`究竟做了什么让我们可以跨文件访问属性或者函数呢？在应用编译的过程中，总共分为`预处理`、`编译`、`汇编`以及`链接`四个过程，在前三个过程完成时，项目内部的文件分别被转换成机器语言的二进制文件，而第四步`链接`负责把这些文件。在忽略掉大量的编译细节后，整体的流程如下图：

![](http://upload-images.jianshu.io/upload_images/783864-c95b644349ffdcb2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果存在`import`的依赖，那么在预处理阶段就会被替换成相应的代码，因此上图不包含`import`的编译过程。可以看到，文件之间的合并实际上发生在`链接`也就是最后一步，在此之前机器指令的二进制就已经完成了，此时单个文件的二进制是不能确定外部文件的地址信息的。为什么在通过`extern`声明之后这些生成的二进制指令在编译的时候没有出错呢？

在完成`链接`之后，所有的二进制指令文件会被合并成一个大的二进制文件包，在`iOS`开发中最终生成的二进制包被我们称作`Mach-O`：

![](http://cc.cocimg.com/api/uploads/20150122/1421892661838860.gif)

实际上不管这个二进制包是不是`Mach-O`的格式，也都是被分成很多个段`section`。每个段存储了其应该存储的信息，基本上各种博客都会告诉你里面有`.TEXT`、`.DATA`这两个，还有一些其余的比较少人提起。其中有个重要的段存储了名为`重定位表`的信息，就是这个表协同作用避免了文件间访问出现错误。

在编译后产生的二进制数据中，`符号`是用来表示变量、函数、方法等称呼。对于使用`extern`修饰的符号，在编译过程中会生成一个对应的`重定位`信息，这些信息记录了`extern`修饰的属性、函数在文件中的偏移、符号类型、符号属性等重要信息。在`链接`发生时，会根据表上的信息对各个二进制包的内容进行修正，这个修正的过程称为`重定位`。当然整个过程更加的复杂，有兴趣的可以私下去查看相关书籍，笔者就不多阐述了。

## 弱符号
上面已经说了，`符号`用来标识变量、函数等信息，就像身份证对于我们。对于编译器而言，每一个符号存在着`强弱`的关系。不要一看到`强弱`跟自然而言的跟`强弱引用`联想起来，两者毫不相干。对于编译器来说，所有函数和已经初始化的全局变量默认为`强符号`，未初始化的全局变量为`弱符号`。对于这两种符号，有三条至关重要的规则：

- 不允许强符号被多次定义；如果存在多个强符号定义，则链接器报符号重复定义错误。

- 如果一个符号在某个目标文件中是强符号，在其他文件中都是弱符号，那么选择强符号。

- 如果一个符号在所有目标文件中都是弱符号，那么选择其中占用空间最大的一个。

通俗来讲就是同一个声明的全局变量在多个文件中定义了同一个变量，那么编译器会报错。假如我们想要同时可以存在这么两个同名变量，又不希望编译出错，编译器提供了一个修饰符来将某个符号修饰为若符号：

    __attribute__((weak)) int _share_integer = 0;
    __attribute__((weakref)) void _share_func() { }

不过经笔者测试，函数的`weakref`修饰在`Xcode`的编译环境下会报错，如有知道为什么的，万请指教。在测试中，也意外的发现了`weak`对函数的修饰照样有用。通过弱符号的修饰，我们可以在三个文件中声明同名的函数，实际上不管在哪个文件中调用最终都会只调用其中一个函数：

    /// Object1.m
    __attribute__((weak)) dispatch_semaphore_t _shared_lock() {
        NSLog(@"function in Object1");
        static dispatch_semaphore_t _shared_lock;
        static dispatch_once_t _once;
        dispatch_once(&_once, ^{
            _shared_lock = dispatch_semaphore_create(1);
        });
        return _shared_lock;
    }
    
    @implementation NSObject1
    
    + (void)load {
        _shared_lock();
    }
    
    @end

    /// Object2.m 
    __attribute__((weak)) dispatch_semaphore_t _shared_lock() {
        NSLog(@"function in Object2");
        static dispatch_semaphore_t _shared_lock;
        static dispatch_once_t _once;
        dispatch_once(&_once, ^{
            _shared_lock = dispatch_semaphore_create(1);
        });
        return _shared_lock;
    }
    
    @implementation NSObject2
    
    + (void)load {
        _shared_lock();
    }
    
    @end
    
    /// Object3.m
    __attribute__((weak)) dispatch_semaphore_t _shared_lock() {
        NSLog(@"function in Object3");
        static dispatch_semaphore_t _shared_lock;
        static dispatch_once_t _once;
        dispatch_once(&_once, ^{
            _shared_lock = dispatch_semaphore_create(1);
        });
        return _shared_lock;
    }
    
    @implementation NSObject3
    
    + (void)load {
        _shared_lock();
    }
    
    @end

运行之后访问的总是同一个方法：

![](http://upload-images.jianshu.io/upload_images/783864-b54dd4099f354807.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然，可以通过取消掉`__attribute__`修饰的方式控制最终确定的符号，比如将`Object3`中的修饰去掉之后，再次运行：

![](http://upload-images.jianshu.io/upload_images/783864-71910e244c8d767e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同样的，`弱符号`修饰的方式依赖于编译阶段的特性。比如`extern`对应的是`重定向表`，那么`弱符号`对应的是`.COMMON`段，所有弱符号会在`链接`前被放入这个段中，最终才通过大小比对、以及强弱符号的对比决定出真正的符号。

## 比较
相比较传统使用的`单例`模式，这两种`伪单例`模式并不需要引入单例文件的依赖，却又能保证在多处使用这些符号的地方保证符号的单一性。当然这种无依赖并不是没有代价的，传统的`单例`总是更可靠的，因为在使用时已经明确了符号在二进制中的位置、依赖关系，代码因此会更加健壮。而`extern`和`弱符号`又有着各自的缺陷：

- `extern`

    `extern`是通过重定位机制实现的，这意味着在链接阶段，`extern`修饰的符号的地址必须是存在的。也就是说如果`extern`修饰的符号没有定义或者初始化，那么程序将会报错。鉴于多数的开发者或多或少使用过，就不再多说。

- `弱符号`

    与`extern`不同的是，弱符号本身修饰的符号是可以自定义的。即是说可以同时存在多份定义和实现，但是最终只会使用一份，这种特性意味着让代码灵活性更强。什么意思？
    
    当某个业务被拆分成多个组件时，大部分时间总是存在着我们想要共用的属性，这也意味着依赖。添加一个抽象层是一种很好的解决方案，但是往往这些共用属性又不至于单独用一个抽象层来完成。比如我在`FPS`、`CPU`等多种性能监控中想要通过一个自定义的串行队列来控制执行顺序，这时候就能通过`弱符号`的修饰来避免创建过多的队列，或者`dispatch_semaphore_t`。
 
    当然，`弱符号`也意味着你在使用不开源的第三方库时，可能发生了符号重复。无论最终确认的符号是哪个，对于我们和第三方来说都可能导致致命的错误。
 
 
## 尾话
来北京也有三周了，前端时间和北京的朋友出来玩，期间讨论到技术相关的话题。有一句话我深刻的认同：

> 那些我们拖欠的，总有一天要还债

技术债更是如此，所谓经济基础决定上层建筑。随着工龄和技术的提高，阻隔我们技术前进的终究是我们曾经因偷懒欠下的债，想要越过这个坎，只能回头还债。只是在我们年轻的时候，还债总是更容易的。望各位读者且`code`且珍惜！

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)


