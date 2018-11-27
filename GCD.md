> 本文摘自[iOS与OS X多线程和内存管理](https://github.com/aozhimin/awesome-iOS-resource/blob/master/Books/Objective-C%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B%20iOS%E4%B8%8EOS%20X%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%92%8C%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.pdf)  

# GCD
## 手动生成Dispatch Queue
### Serial Dispatch Queue （等待现在执行中处理）
### Concurrent Dispatch Queue （不等待现在执行中处理）
每个Serial Dispatch Queue中同时只能执行一个追加处理，但是可以生成多个Serial Dispatch Queue达到多个追加处理同时执行的效果。但是大量Serial Dispatch Queue会消耗大量内存，引起大量上下文切换，大幅降低系统响应性能。因此只在为了避免多个线程更新相同资源导致数据竞争时使用Serial Dispatch Queue

``` objc  
dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.objcTest.MySerialDispatchQueue")
```
``` objc
dispatch_queue_create(const char * _Nullable label, dispatch_queue_attr_t  _Nullable attr)
```
的第二个参数置为NULL表示生成Serial Dispatch Queue，置为DISPATCH\_QUEUE\_CONCURRENT 则生成Concurrent Dispatch Queue   

## Main Dispatch Queue/Global Dispatch Queue
Main Dispatch Queue是在主线程中执行的Dispatch Queue，因为主线程只有1个，所以Main Dispatch Queue是Serial Dispatch Queue（在Main Dispatch Queue中执行与界面更新有关的操作）  
Global Dispatch Queue 是Concurrent Dispatch Queue

### 各种Dispatch Queue获取方法如下
> Main Dispatch Queue
	
	dispatch_queue_t mainispatchQueue = dispatch_get_main_queue();
> Global Dispatch Queue(High Priority)

	dispatch_queue_t globalDispatchQueueHigh = disptch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0)
> Global Disptch Queue(Default Priority)
 
	disptch_queue_t globalDispatchQueueDefault = disptch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
> Global Dispatch Queue(Low Priority)
	
	dispatch_queue_t globalDispatchQueueLow = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0)
> Global Dispatch Queue(Background Priority)

	dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0)
****
使用举例：     

``` objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    //可并行执行的处理
        
    //在main dispatch queue中执行block
    dispatch_async(dispatch_get_main_queue(), ^{
        //只能在主线程执行的处理
    });
});
```

## Dispatch Group   

> 当三个block执行完之后通知主线程进行结束处理   

``` objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, queue, ^{
    NSLog(@"blk0");
});
dispatch_group_async(group, queue, ^{
    NSLog(@"blk1");
});
dispatch_group_async(group, queue, ^{
    NSLog(@"blk2");
});
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"done");
});
```
> 在超时时间内等待三个block执行结束

``` objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, queue, ^{
    NSLog(@"blk0");
});
dispatch_group_async(group, queue, ^{
    NSLog(@"blk1");
});
dispatch_group_async(group, queue, ^{
    NSLog(@"blk2");
});

//阻塞当前线程，等待group结束，超时时间为永久
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
//阻塞当前线程，等待group结束，超时时间为1秒
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
long result = dispatch_group_wait(group, time);
//result 为0表示全部执行结束，否则还有这正在执行的处理
if (result) {
    //属于Dispatch Group的某些处理还在执行中
} else {
    //属于Dispa Group的处理全部执行结束
}
```

## dispatch\_barrier\_async()

> 使用Concurrent Dispatch Queue 和dispatch_barrier_async函数可以实现高效率的数据库访问和文件访问   

当需要在Concurrent Dispatch Queue中需要处理数据竞争问题时，可以使用Dispatch Group与dispatch\_set\_target\_queue(改变优先级)来达到此目的，但这样源代码会较为复杂，gcd提供了更为高效的api:**dispatch\_barrier\_async()**
> 例如在数据库读取过程中进行写入操作  

```objc
dispatch_queue_t barrierQueue = dispatch_queue_create("com.objcTest.gcd.forBarrier", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(barrierQueue, ^{
    NSLog(@"blk0_for_reading");
});
dispatch_async(barrierQueue, ^{
    NSLog(@"blk1_for_reading");
});
dispatch_async(barrierQueue, ^{
    NSLog(@"blk2_for_reading");
});
dispatch_async(barrierQueue, ^{
    NSLog(@"blk3_for_reading");
});
    
dispatch_barrier_async(barrierQueue, ^{
    NSLog(@"blk_for_writing");
});
    
dispatch_async(barrierQueue, ^{
    NSLog(@"blk4_for_reading");
});
dispatch_async(barrierQueue, ^{
    NSLog(@"blk5_for_reading");
});
dispatch_async(barrierQueue, ^{
    NSLog(@"blk6_for_reading");
});
dispatch_async(barrierQueue, ^{
    NSLog(@"blk7_for_reading");
});
```

运行结果:前4个读操作顺序不定，后4个读操作顺序不定，但是写操作必在前4个读操作之后，后4个读操作之前进行。   
原理：dispatch\_barrier\_async函数会等待追加到Concurrent Dispatch Queue上的并行执行的处理全部结束之后，再将指定的处理追加到该Concurrent Dispatch Queue中，然后在由dispatch\_barrier\_async函数追加的处理执行完毕后，Concurrent Dispatch Queue才恢复为一般的动作，追击到该Concurrent Dispatch Queue的处理又开始并行执行

## dispatch_sync

将指定的block同步追加到指定的Dispatch Queue中，在追加的Block结束之前，dispatch\_sync函数会一直等待（简易版的dispatch\_group\_wait），dispatch\_sync容易引起死锁:  

``` objc
//使用dispatch_sync引起死锁的例子
dispatch_queue_t mainQueue = dispatch_get_main_queue();
dispatch_sync(mainQueue, ^{
    NSLog(@"hello");
});
```
原因:dispatch\_sync函数在主线程中执行，block也被追加到主线程中，block在等待dispatch\_sync函数执行完成，dispatch\_sync在等待block执行完成，引起死锁   
同样的，Serial Dispatch Queue也会引起同样的问题

## dispatch\_apply
dispatch_apply是dispatch_sync函数和Dispatch Group的关联api。该函数按指定的次数将指定的block追加到指定的Dispatch Queue中，并等待全部处理执行结束

``` objc
dispatch_queue_t queueForApply = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply(10, queueForApply, ^(size_t index) {
    NSLog(@"%zu", index);
});
NSLog(@"done");
```

各个处理的执行时间不定，但是输出结果中最后的done必定在最后的位置上。因为dispatch_apply会等待全部处理执行结束，第三个参数会自增

> 对NSArray类对象的所有元素执行处理

``` objc
NSArray *array = @[@1, @2, @3, @4, @5];
dispatch_queue_t queueForApply = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply(array.count, queueForApply, ^(size_t index) {
    NSLog(@"the %zu item of array is %@", index, [array objectAtIndex:index]);
});
NSLog(@"done");
```

另外，由于dispatch\_apply函数与dispatch\_sync相同，会等待处理执行结束，因此推荐在disptch\_async函数中非同步地执行dispatch\_apply函数

``` objc
NSArray *array = @[@1, @2, @3, @4, @5];
dispatch_queue_t queueForApply = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
//在Global Dispatch Queue中非同步执行
dispatch_async(queueForApply, ^{
    //Global Dispatch Queue等待dispatch_apply中全部处理执行结束
    dispatch_apply(array.count, queueForApply, ^(size_t index) {
        //并列处理包含在NSArray 对象中的全部对象
        NSLog(@"the %zu item of array is %@", index, [array objectAtIndex:index]);
    });
    //dispatch_apply中处理全部执行结束
    //在Main Dispatch Queue 中非同步执行
    dispatch_async(dispatch_get_main_queue(), ^{
        //在Main Dispatch Queue中执行处理（界面更新等）
        NSLog(@"done");
    });
});
```

## dispatch\_suspend / dispatch\_resume

当追加大量处理到Dispatch Queue时，在追加处理的过程中，有时希望不执行已追加的处理。例如计算结果被Block截获时，一些处理会对这个验算结果造成影响。   
在这种情况下，只要挂起Dispatch Queue即可。当可执行时再恢复
dispatch_suspend函数挂起指定的Dispatch Queue。

	dispatch_suspend(queue)；
dispatch_resume函数恢复指定的Dispatch Queue。

	dispatch_resume(queue);
这些函数对已经执行的处理没有影响。挂起后，追加到Dispatch Queue中但尚未执行的处理再次之后停止执行。而恢复则使得这些处理能够继续执行。

## Dispatch Semaphore

> 不考虑顺序，将所有数据追加到NSMutableArray中

``` objc
dispatch_queue_t queue= dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
NSMutableArray *array = [[NSMutableArray alloc] init];
    
for (int index = 1; index<100000;index++) {
    dispatch_async(queue, ^{
        [array addObject:[NSNumber numberWithInt:index]];
    });
}
```
编译通过，运行出错: `malloc: *** error for object 0x600003392480: pointer being freed was not allocated`
大量的开辟内存导致内存出错。   
Dispatch Semaphore原理与信号量类似。  
一般用法:

``` objc
//计数值初始化为1
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
//永久等待Dispatch Semaphore计数值大于等于1
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
//在一定时间内等待
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
long result = dispatch_semaphore_wait(semaphore, time);
if (result) {
    //Dispatch Semaphore的计数值为0，在到达指定时间为止待机
} else {
    //Dispatch Semaphore计数值大于等于1或者在待机中的指定时间内计数值大于等于1，Dispatch Semaphore计数值减一
    //可执行需要进行排他控制的处理
    
}
```

用于本题:   

``` objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
//生成Dispatch Semaphore,计数初始值设置为1,保证可访问NSMutableArray类对象的线程同时只能有一个
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
NSMutableArray *array = [[NSMutableArray alloc] init];
for (int index = 0;index<100000;index++) {
    dispatch_async(queue, ^{
        //持续等待Dispatch Semaphore计数值大于等于1
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        //由于Dispatch Semaphore的计数值大于等于1，所以将计数值i减去1，dispatch_semaphore_wait函数返回
        //因此执行到此处时Dispatch Semaphore必为0
        //由于可访问NSMutableArray的线程只有一个，因此可以安全更新
        [array addObject:[NSNumber numberWithInt:index]];
        //排他控制处理结束，所以通过dispatch_semaphore_signal将计数值+1
        //如果有通过dispatch_semaphore_wait函数等待Dispatch Semaphore的计数增加的线程，就由最先等待的线程执行
        dispatch_semaphore_signal(semaphore);
    });
}
```
> 在没有Serial Dispatch Queue和dispatch_barrier_async函数那么大粒度且一部分处理需要进行排他控制的情况下，Dispatch Semaphore便可发挥威力

## dispatch\_once

dispatch\_once函数是保证在应用程序执行中只执行一次指定处理的API。下面这种经常出现的用来进行初始化的源代码可通过dispatch_once函数简化.

``` objc
static int initialized = NO;
if (initialized == NO) {
	//初始化
	initialized = YES;
}
```
如果使用dispatch_once函数，则源代码写为:

``` objc
static dispatch_once_t pred;
dispatch_once(&pred, ^{
	//初始化
});
```
源代码看起来没有太大变化。但是通过dispatch_once函数，该源代码即使在多线程环境下执行，也可保证百分之百安全。   
之前的源代码在大多数情况下也是安全的。但是在多核CPU中，在正在更新表示是否初始化的标志变量时读取，就有可能多次执行初始化处理。而用dispatch_once函数初始化就不必担心这样的问题。这就是所说的单例模式，在生成单例对象时使用。










