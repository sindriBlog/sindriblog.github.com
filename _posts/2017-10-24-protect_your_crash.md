---
title: 质量监控-保护你的crash
date: 2017-10-24 08:00:00
tags:
- 质量监控
---

如何去衡量一款应用的质量好坏？为了回答这一问题，`APM`这一目的性极强的工具向开发顺应而生。最早的`APM`开发只关注于`crash`、`cpu`这类的硬性指标。而随着移动开发市场的成熟，越来越多的数据指标也被加入到了`APM`的采集范畴中，包括感官体验相关的数据和使用习惯等。

然而，无论`APM`最终如何发展，其最核心的采集指标一定是`crash`数据。一套完善的`crash`监控方案可以快速的发现并协助完成问题定位，从而能够及时止损，避免更多的损失。而反过来说，如果`crash`不能及时被发现，又或者因为采集链中出现异常导致了数据丢失，对于开发者和公司来说，这都会是一个噩梦。

## crash采集
细分之下，`crash`分别存在`mach exception`、`signal`以及`NSException`三种类型，每一种类型表示不同分层上的`crash`，也拥有各自的捕获方式。

- `mach exception`
 
    `mach异常`由处理器陷阱引发，在异常发生后会被异常处理程序转换成`Mach消息`，接着依次投递到`thread`、`task`和`host`端口。如果没有一个端口处理这个异常并返回`KERN_SUCCESS`，那么应用将被终止。每个端口拥有一个`异常端口数组`，系统暴露了后缀为`_set_exception_ports`的多个`API`让我们注册对应的异常处理到端口中。
    
    ![](http://upload-images.jianshu.io/upload_images/2833754-3846169f772536d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    `mach异常`即便注册了对应的处理，也不会导致影响原有的投递流程。此外，即便不去注册`mach异常`的处理，最终经过一系列的处理，`mach异常`会被转换成对应的`UNIX信号`，一种`mach异常`对应了一个或者多个信号类型。因此在捕获`crash`要提防二次采集的可能。
    
    ![](http://upload-images.jianshu.io/upload_images/2833754-009bdca3149d428a.png?imageMogr2/auto-orient/strip%7CimageView2/2)

- `NSException`
 
    `NSException`发生在`CoreFoundation`以及更高抽象层，在`CoreFoundation`层操作发生异常时，会通过`__cxa_throw`函数抛出异常。在通过`NSSetUncaughtExceptionHandler`注册`NSException`的捕获函数之后，崩溃发生时会调用这个捕获函数。~~但如果没有任何函数去捕获这个异常~~ 如果在捕获函数中没有进行操作终止应用，最终异常会通过`abort()`来抛出一个`SIGABRT`信号。
    
    ![](http://upload-images.jianshu.io/upload_images/783864-65123adcd0e5fb08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    由于`NSException`的抽象层次足够高，相比较其他的`crash`类型，`NSException`是可以被人为的阻止`crash`的。比如`@try-catch`机制能够捕获块中发生的异常，避免应用被杀死。但由于`try-catch`的开销和回报不成正比，往往不会使用这种机制。其二是`crash防护`，这一手段通过`hook`掉上层接口来规避`crash`风险，但是只建议用于线上防护，而且`hook`未必不会导致其他的问题。

- `signal`
 
    `signa`会导致`crash`，这是多数`iOS`开发者对于信号的印象。传递`crash`信息其实只是信号的一部分功能，信号是一套基于`POSIX标准`开发的通信机制，具体可以阅读[Signal-wikipedia](https://en.wikipedia.org/wiki/Signal_(IPC))。在`signal.h`中声明了`32`种异常信号，下面列出一部分的信号异常对：
    
    | 信号 | 异常 |
    | --- | --- |
    | SIGILL | 执行了非法指令，一般是可执行文件出现了错误 |
    | SIGTRAP | 断点指令或者其他trap指令产生 |
    | SIGABRT | 调用abort产生 |
    | SIGBUS | 非法地址。比如错误的内存类型访问、内存地址对齐等 |
    | SIGSEGV | 非法地址。访问未分配内存、写入没有写权限的内存等 |
    | SIGFPE | 致命的算术运算。比如数值溢出、NaN数值等 |
    
    虽然存在三种`crash`，但由于`mach exception`会在`BSD`层被转换成`UNIX信号`，`NSException`在未被捕获的情况下会调用`abort`抛出信号，因此即便是我们只注册了`signal`的处理，只要注册的`signal`足够多，理论上也是能捕获到全部的`crash`。
    
    ![](http://upload-images.jianshu.io/upload_images/783864-e62ce7845472cb94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    
## 采集冲突
由于`crash`的捕获机制只会保存最后一个注册的`handle`，因此如果项目中残留或者存在另外的第三方框架采集`crash`信息时，经常性的会存在冲突。解决冲突的做法是在注册自己的`handle`之前保存已注册的处理函数，便于发生崩溃后能将`crash`信息连续的传递下去。
    
    
    struct sigaction my_action;
    static struct sigaction registered_action;
    static NSUncaughtExceptionHandler *previousHandle;
        
    void signal_handler(int signal) {
        ......
    }
    
    void exception_handler(NSException *exception) {
        ......
    }
        
    void registerCrashHandle() {
        previousHandle = NSGetUncaughtExceptionHandler();
        NSSetUncaughtExceptionHandler(&exception_handler);
        
        myAction.sa_handler = &signal_handler;
        sigemptyset(&my_action.sa_mask);
        sigaction(SIGABRT, &my_action, &registered_action);
    }
    
一般来说，一个经验丰富的开发者在注册`crash`回调时都会主动的去保存其他函数，避免因为冲突导致别人的数据丢失。但是即便按照这样的方式来注册你的回调，也不代表我们的处理函数是安全的。最重要的原因在于完成回调的注册之后，我们无法保证后续会不会有其他人继续注册，如果有就会存在被替换掉的风险

## 解决方案
按照正常方式的做法，能保证先于我们注册的`crash`回调不会被我们拦截导致失败，但如果在我们后方存在另外的注册，我们需要一个有效的机制来保护我们的采集数据。解决问题的收益是不变的，所以解决方案理当尽可能的低开销和低风险。

如何去判断我们的`handle`是否安全？这要求我们对已注册的`handle`进行检测。首先检测时机要选择在哪？由于`crash`是可能发生在应用启动阶段的，因此`crash`采集一般也是发生在`didLaunch`这个时间，下图是我绘制的应用启动到完全启动的几个重要阶段：

![](http://upload-images.jianshu.io/upload_images/783864-ddd56666f11ae7de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`applicationActive`这个阶段基本上是能保证`crash`相关的注册都完成的，因此冲突检测可以放到这个阶段进行。

### 周期性检测
利用已有的周期性机制或者使用定时器来进行`handle`冲突检测。可以分别使用`通知`和`定时器`两个机制来完成周期性检测方案

- `监听应用状态`

    监听`UIApplicationDidBecomeActiveNotification`在应用进入活跃状态时做检测：
    
        - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
            ......
            [[NSNotificationCenter defaultCenter] addObserver: [SignalHandler sharedHandler] selector: @selector(checkRegisterCrashHandler) name: UIApplicationDidBecomeActiveNotification object: nil];
            ......   
        }
        
        static struct sigaction existActions[32];
        static int fatal_signals[] = {
            SIGILL,
            SIGBUS,
            SIGABRT,
            SIGPIPE,
        };
        
        - (void)checkRegisterCrashHandler {
            struct sigaction oldAction;
            for (int idx = 0; idx < sizeof(fatal_signals) / sizeof(int); idx++) {
                sigaction(fatal_signals[idx], NULL, &oldAction);
                if (oldAction.sa_handler != &signal_handler) {
                    existActions[fatal_signals[idx]] = oldAction;
                    
                    struct sigaction myAction;
                    myAction.sa_handler = &signal_handler;
                    sigemptyset(&myAction.sa_mask);
                    sigaction(SIGABRT, &myAction, NULL);
                }
            }
        }
    
- 定时器检测

    创建定时器来进行周期性的检测，相比通知的机制，可以控制检测间隔：

        - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
            ......
            NSTimer *timer = [[NSTimer alloc] initWithFireDate: [NSDate date] interval: 30 target: [SignalHandler sharedHandler] selector: @selector(checkRegisterCrashHandler) userInfo: nil repeats: YES];
            [[NSRunLoop currentRunLoop] addTimer: timer forMode: NSRunLoopCommonModes];
            [timer fire];
            ......   
        }

### hook注册函数
通过`hook`调用注册`handle`的对应函数，建立一个回调数组来保存非`exception_handle`的所有回调，后续处理完我们的采集，再逐个调起。由于捕获函数都是基于`C`接口的，因此我们需要[fishhook](https://github.com/facebook/fishhook)来提供相应的`hook`功能。

    struct SignalHandler {
        void (*signal_handler)(int);
        struct SignalHandler *next;
    }
    struct SignalHandler *previousHandlers[32];

    void append(struct SignalHandler *handlers, struct SignalHandler *node) { 
        ......
    }
    
    static int (*origin_sigaction)(int, const struct sigaction *__restrict, struct sigaction * __restrict) = NULL;
    
    int custom_sigaction(int signal, const struct sigaction *__restrict new_action, struct sigaction * __restrict old_action) {
        if (new_action.sa_handler != signal_handler) {
            append(previousHandlers[signal], new_action);
            return origin_sigaction(signal, NULL, old_action);
        } else {
            return origin_sigaction(signal, new_action, old_action);
        }
    }

### 风险
在周期性检测的方案下，假设存在`handle`注册链（依次从左到右）：

`previous` <- `exception_handle` <- `other`

在检测时发现当前回调是`other`，于是重新注册我们的回调，保存`other`。但是假如`other`也保存了我们的回调，这样可能会导致崩溃发生的时候，调用顺序变成一个死循环。

![](http://upload-images.jianshu.io/upload_images/783864-3dff6046e47956e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`hook`方案则是因为在调用`origin_sigaction`时会传入`old_action`，可能导致另外的注册者保存了我们的`exception_handle`，并在最后处理的时候出现同样的循环调用问题。对于`hook`方案来说，解决方法要简单很多，只需要在非我们的注册调用`origin_sigaction`时不传入`old_action`就能保证其他注册者无法获取到我们的回调：

    int custom_sigaction(int signal, const struct sigaction *__restrict new_action, struct sigaction * __restrict old_action) {
        if (new_action.sa_handler != signal_handler) {
            append(previousHandlers[signal], new_action);
            return origin_sigaction(signal, NULL, NULL);
        } else {
            return origin_sigaction(signal, new_action, old_action);
        }
    }

而使用周期性监测，就需要考虑是否放弃`other`的回调，最终只保证`exception_handle`和`previous`和更早之前的注册能够被顺利调起。

另外，`hook`还存在一个风险是假如第三方同样做了`hook`掉注册函数的处理，并且做了筛选处理，最终导致的结果是没办法完成任何一个注册。两害相较取其轻，个人的建议是使用周期性检测方案。


## 最简单的方式
上述的两套方案都存在风险点，而且这些风险点对于应用来说都算是致命的。那么有没有几乎没有风险又能解决问题的办法呢？答案是肯定的，那就是不要用有潜在风险的第三方，或者和第三方开发者商量提供一个无需`crash`采集的版本。

在应用发生崩溃的时候，此时的`崩溃所在线程`是极不稳定的，不稳定性包括几点：

- `内存不稳定`

    如果是内存相关错误引发的`crash`，比如内存过载、野指针等，此时线程的内存是危险状态。如果这时候在`handle`中再次分配内存，极有可能导致二次`crash`

- `死锁`

    大多数底层的的核心`API`会涉及到加锁处理，这一情况在`signal`错误中出现的较多。而作为上层调用方的我们是不自知的，此时错误的操作可能导致线程陷入死锁状态
    
理论上当我们拦截了一个`signal`的时候，此时的应用会陷入内核并停止工作，应用页面卡死，这时候我们可执行时长是无限的。如果处理链过长，耗时过多或者陷入某种循环，会造成一种应用卡死而非崩溃的错觉，而经过我厂大量的统计，`应用卡死`要比`应用崩溃`更让人难以接受。此外，过多的处理链会增加回调流程上的风险点。如果链条上的某个点发生了二次崩溃，会导致后续的处理都无法执行。因此，不用第三方或者让第三方去除`crash`采集，是一种可行且高效的手段。

![](http://upload-images.jianshu.io/upload_images/783864-c1d0dc3ace633edd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/783864-c9fff26dd02e604a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 其他
文中提到过一次现在比较流行的`crash防护`手段，这里还是想说两句。在开发中，`crash防护`会造成依赖心理，降低对风险的敏感。而在线上，这种方案可能屏蔽了大量的低级错误，也是让我不能容忍的，当然循环引用的防护属于例外。最后安利一波寒神的[XXShield](https://github.com/ValiantCat/XXShield)，除了容器类的防`crash`都值得学习，尤其是正确的`method swizzling`姿势。

## 参考
[Foundation](https://github.com/apportable/Foundation)

[iOS异常捕获](http://www.iosxxx.com/blog/2015-08-29-iosyi-chang-bu-huo.html)

[libc++ api spec](https://libcxxabi.llvm.org/spec.html)

[Linux信号处理机制](http://hutaow.com/blog/2013/10/19/linux-signal/)

[浅谈Mach Exceptions](http://www.jianshu.com/p/725e7d69272c)

[漫谈iOS Crash收集框架](https://nianxi.net/ios/ios-crash-reporter.html)

[源码剖析signal和sigaction的区别](http://blog.csdn.net/wangzuxi/article/details/44814825)

[iOS Crash捕获及堆栈符号化思路剖析](http://www.jianshu.com/p/29051908c74b)

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)


