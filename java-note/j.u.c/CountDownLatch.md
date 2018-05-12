# 原理介绍
CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行。

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

![图](https://github.com/alanzhang211/learning-note/raw/master/img/CountDownLatch.png)

# 源码分析
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/countdownlatch-class.jpg)


内部通过同步器Sync实现锁机制，Sync继承于AQS。

```
private final Sync sync;
```

## 一个状态
记录了计数器的值。

```
Sync(int count) {
    setState(count);
}
```


## 构造函数
```
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```
构造函数中的计数值（count）实际上就是闭锁需要等待的线程数量。这个值只能被设置一次，而且CountDownLatch没有提供任何机制去重新设置这个计数值(*CyclicBarrier*可以重置)。

调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个CountDownLatch的引用传递到线程里即可。

## 核心方法
### await()
当线程调用了CountDownLatch对象的await方法后，当前线程会被阻塞。满足下列条件之一会返回：
1. 所有线程都调用了CountDownLatch对象的countDown方法。计数器为0.
2. 其他线程调用当前线程的interrupt方法中断当前线程。（CountDownLatch支持中断）

```
//支持中断获取共享资源
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

//AQS中获取共享资源支持中断
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
            //如果线程被中断则抛异常
        if (Thread.interrupted())
            throw new InterruptedException();
            //如果计数器为0，直接返回；否则进入AQS等待队列
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

//CountDownLatch中实现tryAcquireShared方法。
protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
```
### countDown()
当线程调用了该方法后，会递减计数器的值，递减后如果计数器为0则会唤醒所有调用await方法而被阻塞的线程,否者什么都不做。

```
//计数器减去1，委托给AQS的releaseShared
public void countDown() {
        sync.releaseShared(1);
    }

//AQS
public final boolean releaseShared(int arg) {
        //释放资源
        if (tryReleaseShared(arg)) {
            //释放共享资源
            doReleaseShared();
            return true;
        }
        return false;
    }
//CountDownLatch实现tryReleaseShared
protected boolean tryReleaseShared(int releases) {
            // 循环进行cas，直到当前线程成功完成cas使计数值（状态值state）减一并更新到state
            for (;;) {
                int c = getState();
                //防止计数器被减为负数
                if (c == 0)
                    return false;
                //计数器减1
                int nextc = c-1;
                //设置计数器state
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }

```

# 应用场景
1. 实现线程等待：开始执行前等待n个线程完成各自任务。
2. 死锁检测：一个非常方便的使用场景是，你可以使用n个线程访问共享资源，在每次测试阶段的线程数目是不同的，并尝试产生死锁。
3. 实现最大的并行性：有时我们想同时启动多个线程，实现最大程度的并行性。

# 实战分析
应用于多维分析报表中：报表统计时，最终会依据多个大区的数据进行汇总，得出最终结果。那么，汇总作为主线程等待；不同大区统计可以作为一个统计线程。知道每个大区线程统计完成，汇总线程在进行sum汇总。


# FAQ
1. CountDownLatch 中的await为什么是调用AQS的共享资源的acquireInterruptibly方法？

是因为这里状态值需要的并不是非0即1的独占效果；而是共享的状态计数器。

---
*参考资料：*
1. 《java并发编程的艺术》
2. http://gitbook.cn/books/5ada8b951cbb7735a16e5734/index.html
3. https://letcheng.github.io/2016/05/26/java-countdownlatch.html
4. http://www.importnew.com/15731.html
