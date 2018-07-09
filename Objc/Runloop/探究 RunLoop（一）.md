# 探究 Runloop （一）
>对于 iOS 开发者来说， Runloop 应该是大家都熟悉的，这篇文章只是本人在阅读源码时的一些理解。

### Runloop 相关结构体

Objective-C 是对 C 的封装，所以一切相关的底层其实都是通过 C 来实现的，我们先看下 Runloop 相关的概念。
这里说的 Runloop 并不是 Apple 在 iOS 中的实现，因为 iOS 不是开源的。我们现在所说的其实只是 Apple 开源的 CF 框架的源码，但是管中窥豹，通过它也能了解 Apple 的思路。

#### __CFRunLoop
首先是 Runloop，底层是 C 的结构体

``` C
struct __CFRunLoop {
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    volatile _per_run_data *_perRunData;   // 预设状态
    pthread_t _pthread; // 线程
    CFMutableSetRef _commonModes; // 存储 ModeName
    CFMutableSetRef _commonModeItems; 
    CFRunLoopModeRef _currentMode; // 当前 Mode
    CFMutableSetRef _modes;  // 所有的 Mode
    ...
};
```

上面列出的是相对重要的部分， 我们一个一个看。没有细说的注意看代码块中的注释
首先 **_wakeUpPort** 这是用来进行唤醒 Runloop 的特定端口；
然后是 **_perRunData** 这个需要结合下面代码来看：

``` C
// 是否是 停止状态
CF_INLINE Boolean __CFRunLoopIsStopped(CFRunLoopRef rl) {
    return (rl->_perRunData->stopped) ? true : false;
}

// 标记为停止状态
CF_INLINE void __CFRunLoopSetStopped(CFRunLoopRef rl) {
    rl->_perRunData->stopped = 0x53544F50;	// 'STOP'
}

...

```

可以看到，runloop 的很多状态都是存储在 **_perRunData** 之中的，我们可以把它看做是 runloop 的状态指示器。

**_pthread** 的类型是 **pthread_t** ，这是 pthread 的底层，这样也说明了，为什么线程和 **runloop** 是一一对应的；

**_commonModes** 这是一个集合， 主要存储着 modename；

**_commonModeItems** 也是一个集合， 主要存储着 modename 对应的 **CFRunLoopModeRef** 中的 Source/Observer/Timer；

#### CFRunLoopModeRef

通过 Runloop 的结构体，我们可以看到，**CFRunLoopModeRef** 在其中占据很大的比例，那么我们继续看 **CFRunLoopModeRef** 是什么东西：

``` C
struct __CFRunLoopMode {
    CFStringRef _name;
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    ...
};
```

很长一段，我只贴出来一部分，其他可以看源码。可以看到也是一个结构体（其实所有OC类的底层实现都是C中的结构体）。

在这其中，其实就是三类成员，**__CFRunLoopSource**， **__CFRunLoopObserver** 和 **__CFRunLoopTimer**。   

它们的关系可以看这幅图，比较直观：
![](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_0.png)

那么继续，我们在聊聊这三个家伙是个啥。

#### __CFRunLoopSource

Mode 结构体中 source 有两个版本，一个source0，一个source1。
先说下，source 是什么，runloop 中，需要一个标识来告诉它，是否有事件需要处理，然而单纯的使用一个标识符无法实现比较复杂的需求，这时候 source 就出现了，通过调用方法，来得到一个 BOOL 值，通过这个值来确定是否需要处理事件。

为什么这么说？我们需要看下这个方法：

```
	static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode){
			...
			Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
	    if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
	    }
			...
			
			sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
	    sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls) || sourceHandledThisLoop;
			...
	
	}
	
```

这是 Runloop 启动方法，后面会仔细分析，现在主要看贴出来的方法。

可以看到，runloop 处理事件主要是通过三个方法：
**__CFRunLoopDoSources0** ， **__CFRunLoopDoSource1** 和 **__CFRunLoopDoBlocks**。 

前两个是用来标志是否存在需要处理的事件，如果存在，则调用第三个方法，处理事件。

Source 也分为2个版本 Source0 和 Source1。

• **Source0** 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。

• **Source1** 包含了一个 mach_port 和一个回调（函数指针），这种 Source 能主动唤醒 RunLoop 的线程。


#### CFRunLoopTimerRef

**CFRunLoopTimerRef** 就很好理解了，它实际上是和 NSTimer 是同一个东西，也就是说，它就是一个定时器。这里简单介绍一下，后面计划有一篇关于 **NSTimer** 的文章。

```C

struct __CFRunLoopTimer {
    CFRunLoopRef _runLoop;
    CFMutableSetRef _rlModes;
    CFTimeInterval _interval;		/* immutable */
    CFTimeInterval _tolerance;          /* mutable */
    CFAbsoluteTime _nextFireDate;
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;	/* immutable */
    ...
};

```
看上面的结构， **__CFRunLoopTimer** 主要包含下列相关：

**_runLoop**：NSTimer 必须和 runloop 配合使用，那么 timer 持有 runloop 是可以理解咯； 
**_interval**： 间隔时间；  
**_tolerance**： 容忍时间，也就是说允许延迟多长时间；  
**_nextFireDate**：下次触发时间，理论上应该每次累加 _interval ，但是由于容忍度的存在，并不能两次相隔时间并不一定和 _interval 相等；  
**_order**： 优先级；  
**_callout**：timer 关联的回调事件；   

#### __CFRunLoopObserver

```C
    struct __CFRunLoopObserver {
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */
};
```

**__CFRunLoopObserver** 中除了常规的几个成员外， 比较重要的应该就是 **_activities** ， 它标识了 runloop 的状态，主要包括  

```C
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),               // 即将进入 Loop
    kCFRunLoopBeforeTimers = (1UL << 1),        // 即将处理 Timers
    kCFRunLoopBeforeSources = (1UL << 2),       // 即将处理 Sources
    kCFRunLoopBeforeWaiting = (1UL << 5),       // 即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),        // 刚从休眠中唤醒
    kCFRunLoopExit = (1UL << 7),                // 即将退出Loop
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};

```

通过这些状态，我们可以判断出 Loop 正处于什么状态，并且可以根据状态不同而做相应的事件，比如监听 ```kCFRunLoopBeforeWaiting```， 在这将在异步线程渲染好的UI在主线程进行更新，从而避免出现主线程阻塞，增加页面流畅度等等。







