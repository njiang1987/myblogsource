title: '[转]使用dispatch_group来进行线程同步'
date: 2016-11-18 15:41:26
tags: [iOS]
---



我的上篇文章[iOS中多个网络请求的同步问题总结](http://www.jianshu.com/p/07eb268c93f2)中用到了dispatch_group来进行线程同步，对用法不是特别熟悉所以整理这篇文章来加深记忆*(闲着也是闲着)*。

<!-- more -->

### 一、简单介绍下将会用到的一些东西

**英语不好就不翻译官方文档了..**

#### 1、dispatch_group_async

```
 Submits a block to a dispatch queue and associates the block with the given
  dispatch group
//将一个block(代码块)加入到dispatch_queue_t queue中并和dispatch_group_t group相关联
void
dispatch_group_async(dispatch_group_t group,
    dispatch_queue_t queue,
    dispatch_block_t block);
```

**个人理解：**将代码块dispatch_block_t block放入队列dispatch_queue_t queue中执行；并和调度组dispatch_group_t group相互关联；如果dispatch_queue_t queue中所有的block执行完毕会调用dispatch_group_notify并且dispatch_group_wait会停止等待；

#### 2、dispatch_group_enter(group)、dispatch_group_leave(group)

```
   Calling this function indicates another block has joined the group through
  a means other than dispatch_group_async(). Calls to this function must be
 * balanced with dispatch_group_leave().
调用这个方法标志着一个代码块被加入了group，和dispatch_group_async功能类似；
需要和dispatch_group_enter()、dispatch_group_leave()成对出现；
void
dispatch_group_enter(dispatch_group_t group);
```

**个人理解：**和内存管理的引用计数类似，我们可以认为group也持有一个整形变量(**只是假设**)，当调用enter时计数加1，调用leave时计数减1，当计数为0时会调用dispatch_group_notify并且dispatch_group_wait会停止等待；

#### 3、dispatch_group_notify

```
void
dispatch_group_notify(dispatch_group_t group,dispatch_queue_t queue,
dispatch_block_t block);
```

个人理解：

4、dispatch_group_wait

```
long
dispatch_group_wait(dispatch_group_t group, dispatch_time_t timeout);
```

个人理解：

和dispatch_group_notify功能类似(*多了一个dispatch_time_t参数可以设置超时时间*)，在group上任务完成前，dispatch_group_wait会阻塞当前线程(*所以不能放在主线程调用*)一直等待；当group上任务完成，或者等待时间超过设置的超时时间会结束等待；

二、dispatch_group_async代码示例

```
- (void)groupSync
{
    dispatch_queue_t disqueue =  dispatch_queue_create("com.shidaiyinuo.NetWorkStudy", DISPATCH_QUEUE_CONCURRENT);
    dispatch_group_t disgroup = dispatch_group_create();
    dispatch_group_async(disgroup, disqueue, ^{
        NSLog(@"任务一完成");
    });
    dispatch_group_async(disgroup, disqueue, ^{
        sleep(8);
        NSLog(@"任务二完成");
    });
    dispatch_group_notify(disgroup, disqueue, ^{
        NSLog(@"dispatch_group_notify 执行");
    });
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        dispatch_group_wait(disgroup, dispatch_time(DISPATCH_TIME_NOW, 5 * NSEC_PER_SEC));
        NSLog(@"dispatch_group_wait 结束");
    });
}
```

向group中放入两个任务(**准确讲是将任务加入到了并行队列disqueue中执行，然后队列和group关联当队列上任务执行完毕时group会进行同步**)，第二个任务会等待8秒所以第一个任务会先完成；会先打印**任务一完成**再打印**任务二完成**，当两个任务都完成时dispatch_group_notify中的block会执行；会接着打印**dispatch_group_notify 执行**；dispatch_group_wait设置了超时时间为5秒所以它会在5秒后停止等待打印**dispatch_group_wait 结束**(*任务二会等待8秒所以它会在任务二完成前打印*)；

![](/uploads/2016/11/18/543531-769b6ce8223ab1b2.jpeg)

测试输出结果

**需要注意的：**dispatch_group_wait是同步的所以不能放在主线程执行。
**补充：** dispatch_group会等和它关联的所有的dispatch_queue_t上的任务都执行完毕才会发出同步信号(**dispathc_group_notify的代码块block会被执行,group_wati会结束等待**)。也就是说一个group可以关联多个任务队列；下面给出示例：

```
- (void)groupSync2
{
    dispatch_queue_t dispatchQueue = dispatch_queue_create("ted.queue.next1", DISPATCH_QUEUE_CONCURRENT);
    dispatch_queue_t globalQueue = dispatch_get_global_queue(0, 0);
    dispatch_group_t dispatchGroup = dispatch_group_create();
    dispatch_group_async(dispatchGroup, dispatchQueue, ^(){
        sleep(5);
        NSLog(@"任务一完成");
    });
    dispatch_group_async(dispatchGroup, dispatchQueue, ^(){
        sleep(6);
        NSLog(@"任务二完成");
    });
    dispatch_group_async(dispatchGroup, globalQueue, ^{
        sleep(10);
        NSLog(@"任务三完成");
    });
    dispatch_group_notify(dispatchGroup, dispatch_get_main_queue(), ^(){
        NSLog(@"notify：任务都完成了");
    });
}
```

上面的代码里有两个队列一个是我自己创建的并行队列dispatchQueue，另一个是系统提供的并行队列globalQueue；dispatch_group_notify会等这两个队列上的任务都执行完毕才会执行自己的代码块。

![](/uploads/2016/11/18/543531-a838003743ac6fb6.jpeg)

三、dispatch_group_enter、dispatch_group_level示例

和dispatch_async相比，当我们调用n次dispatch_group_enter后再调用n次dispatch_group_level时，dispatch_group_notify和dispatch_group_wait会收到同步信号；这个特点使得它非常适合处理**异步任务的同步**当异步任务开始前调用**dispatch_group_enter**异步任务结束后调用**dispatch_group_leve**；

下面是代码示例：

```
- (void)groupSync
{
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_enter(group);
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        sleep(5);
        NSLog(@"任务一完成");
        dispatch_group_leave(group);
    });
    dispatch_group_enter(group);
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        sleep(8);
        NSLog(@"任务二完成");
        dispatch_group_leave(group);
    });
    dispatch_group_notify(group, dispatch_get_global_queue(0, 0), ^{
        NSLog(@"任务完成");
    });
}
```

示例代码中在global_queue上执行sleep任务模拟网络请求。

![img](/uploads/2016/11/18/543531-3e7fe042d6527cd3.jpeg)

文／liang1991（简书作者）
原文链接：http://www.jianshu.com/p/228403206664
著作权归作者所有，转载请联系作者获得授权，并标注“简书作者”。