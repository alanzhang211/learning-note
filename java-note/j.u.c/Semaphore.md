# 原理介绍
Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。内部通过Sync（继承于AQS）实现，支持公平和非公平的两种策略。

# 源码分析
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/Semaphore-class.jpg)

## 两种策略
```
private final Sync sync;
```
内部通过Sync的两个子类FairSync、NonfairSync实现同步。默认实现是非公平的。

```
//默认是非公平的信号量
public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

//选择性策略构造信号量
public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

## 一个状态
state值表示当前持有的信号量个数。
```
Sync(int permits) {
       setState(permits);
   }
```

## 核心方法
### acquire()
```
public void acquire() throws InterruptedException {
        //委托给sync
        sync.acquireSharedInterruptibly(1);
    }

//AQS
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        //如果线程被中断，则抛出中断异常
        if (Thread.interrupted())
            throw new InterruptedException();
        //否者调用sync子类方法尝试获取,这里根据构造函数确定使用公平策略
        if (tryAcquireShared(arg) < 0)
            //如果获取失败则放入阻塞队列,然后再次尝试如果失败则调用park方法挂起当前线程
            doAcquireSharedInterruptibly(arg);
    }

```
> 当前线程调用该方法时候目的是希望获取一个信号量资源，如果当前信号量计数个数大于0，并且当前线程获取到了一个信号量则该方法直接返回，当前信号量的计数会减少1。否者会被放入AQS的阻塞队列，当前线程被挂起，直到其它线程调用了release方法释放了信号量，并且当前线程通过竞争获取到了该信号量。同样，此方法支持中断。

tryAcquireShared根据具体的子类，实现公平和非公平获取信号量。

```
//非公平NonfairSync实现
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
 }

final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
     //获取当前信号量值
     int available = getState();
     //计算当前剩余值
     int remaining = available - acquires;
     //如果当前剩余小于0或者CAS设置成功则返回
     if (remaining < 0 ||
         compareAndSetState(available, remaining))
         return remaining;
    }
}

//公平FairSync
protected int tryAcquireShared(int acquires) {
            for (;;) {
                //如果阻塞队列有前驱节点：有，放弃获取，进入阻塞队列；如果没有，获取并更新信号量
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

### release()
```
public void release() {
        //委托给sync，即AQS实现
        sync.releaseShared(1);
    }

//AQS
public final boolean releaseShared(int arg) {
        //循环,直到释放成功。tryReleaseShared由子类实现
        if (tryReleaseShared(arg)) {
            //资源释放成功则调用park唤醒AQS队列里面最先挂起的线
            doReleaseShared();
            return true;
        }
        return false;
    }

//Semaphore内部Sync实现
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```
每次调用释放1次锁。信号量是线程共享的，信号量没有和固定线程绑定，多个线程可以同时使用CAS去更新信号量的值而不会被阻塞。


# 应用场景
Semaphore=N,依据N的不同取值，有不同的使用场景：
1. N=0：异步任务，同步返回。
```
//伪代码
private static ExecutorService threadPool = Executors.newFixedThreadPool(5);
Semaphore semaphore = new Semaphore(0);
public void asynSemaphore() {
    threadPool.execute(new Runnable() {
        @Override
        public void run() {
            //内部通过single唤醒线程
            semaphore.notify();
        }
    }
    //外部线程等待
    semaphore.Wait(OUT_TIME);
    return;
}
```
2. N=1：加锁，则可以作为互斥锁使用。
```
//伪代码
Semaphore semaphore = new Semaphore(1);
public void lockSemaphore() {
    semaphore.acquire();
    ....
    semaphore.release();
}
```
3. N>1：控制并发线程数，可以用于做流量控制。
```
//伪代码
//并发5个线程
private static ExecutorService executor = Executors.newFixedThreadPool(5);
//限制2个
private static Semaphore semaphore = new Semaphore(2);
public static void main(String args[]){
    for(int i=0;i<5;i++){
        executor.execute(new Runnable() {
            public void run() {
                try {
                    semaphore.acquire();
                    ...
                    TimeUnit.SECONDS.sleep(10);
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
    executor.shutdown();
}
```

# 实战分析
有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。

```
public class SemaphoreTest {
    private static final int THREAD_COUNT = 30;
    private static ExecutorServicethreadPool=Executors.newFixedThreadPool(THREAD_COUNT);
    private static Semaphore s = new Semaphore(10);
    public static void main(String[] args) {
        for (inti = 0; i< THREAD_COUNT; i++) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        s.acquire();
                        System.out.println("save data");
                        s.release();
                    } catch (InterruptedException e) {

                    }
                }
            });
        }
        threadPool.shutdown();
    }
}
```
# FAQ

---
*参考资料：*
1. http://gitbook.cn/books/5ada8b951cbb7735a16e5734/index.html
2. 《java并发编程的艺术》
