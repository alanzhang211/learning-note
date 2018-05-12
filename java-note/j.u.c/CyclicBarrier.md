# 原理介绍
CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

CyclicBarrier内部并不是直接使用AQS实现，而是使用了独占锁ReentrantLock来实现的同步。

# 源码分析
## 类图
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/CyclicBarrier-class.jpg)

```
    //内部通过独占锁实现同步
    private final ReentrantLock lock = new ReentrantLock();
    //条件队列，实现线程间的通信；当线程未全部就位时,到达栅栏的线程将被添加到该条件队列
    private final Condition trip = lock.newCondition();
    //记录线程个数，标识有多少线程可以调用await
    private final int parties;
    //在所有线程就位之后且未被唤醒 期间 执行
    private final Runnable barrierCommand;
    //内部就一个变量 broken 用来记录当前屏障是否被打破
    private Generation generation = new Generation();

    //每次线程调用await，减1；直到0，打破屏障；当count==parties时,栅栏打开
    private int count;

    //Generation内部类，broken 并没有被声明为volatile，是因为锁内使用变量不需要
    private static class Generation {
        boolean broken = false;
    }
```

## 核心方法
### await()
```
public int await() throws InterruptedException, BrokenBarrierException {
   try {
       return dowait(false, 0L);
   } catch (TimeoutException toe) {
       throw new Error(toe); // cannot happen
   }
}
```

当前线程调用 CyclicBarrier的该方法会被阻塞，直到满足下面条件之一才会返回：
+ parties 个线程都调用了 await() 函数，也就是线程都到了屏障点。
+ 其它线程调用了当前线程的interrupt()方法中断了当前线程，则当前线程会抛出InterruptedException异常返回。
+ 当前屏障点关联的Generation对象的broken标志被设置为true时候，会抛出BrokenBarrierException 异常。

调用了内部的dowait方法。

```
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        //获取独占锁 lock
        lock.lock();
        try {
            final Generation g = generation;
            //如果屏障被打破
            if (g.broken)
                throw new BrokenBarrierException();
            //如果被中断
            if (Thread.interrupted()) {
                //打破屏障点
                breakBarrier();
                throw new InterruptedException();
            }
            //如果index==0说明所有线程都到到了屏障点，则执行初始化时候传递的任务
            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    //执行任务
                    if (command != null)
                        command.run();
                    ranAction = true;
                   //激活其它调用await而被阻塞的线程，并重置CyclicBarrier
                    nextGeneration();
                    //返回
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            //如果index!=0
            for (;;) {
                try {
                     //没有设置超时时间，
                     if (!timed)
                        trip.await();
                    //设置了超时时间
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    ...
                }
                    ...
            }
        } finally {
            lock.unlock();
        }
    }

   private void nextGeneration() {
       //唤醒条件队列里面阻塞线程
       trip.signalAll();
       //重置CyclicBarrier
       count = parties;
       generation = new Generation();
    }
```

# 应用场景
CyclicBarrier可以用于多线程计算数据，最后合并计算结果的应用场景。比如我们用一个Excel保存了用户所有银行流水，每个Sheet保存一个帐户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水。

# 实战分析
假设4名运动员跑步，全部到达终点结束，实现如下：

*源码地址：https://github.com/alanzhang211/learning/tree/master/juc/cyclicBarrier*

```
/**
 * 运动员实现类
 * Created by alanzhang211 on 2018/5/12.
 */
public class Runner implements Runnable {
    private CyclicBarrier cyclicBarrier;
    private String name;

    public Runner(CyclicBarrier cyclicBarrier, String name) {
        this.cyclicBarrier = cyclicBarrier;
        this.name = name;
    }

    @Override
    public void run() {
        System.out.println("运动员：" + name + " 到达终点！");
        try {
            cyclicBarrier.await();
        } catch (Exception e) {
        }
    }
}

public class CyclicBarrierDemo {
    //4位运动员，都到达终点，结束
    static CyclicBarrier cyclicBarrier = new CyclicBarrier(4, new FinishPoint());

    static class FinishPoint implements Runnable {
        @Override
        public void run() {
            System.out.println("全部到达终点！");
        }
    }

    public static void main(String[] args) {
        List<Runner> runners = new ArrayList<Runner>(4);
        for (int i = 0; i < 4; i++) {
            runners.add(new Runner(cyclicBarrier, "runner" + "-" + i));
        }
        Executor executor = Executors.newFixedThreadPool(5);
        for (Runner runner : runners) {
            executor.execute(runner);
        }
    }
}
```

# FAQ
1. CyclicBarrier和CountDownLatch的区别

CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次。

另外当很多线程在await()方法上等待的时候，如果一个线程被中断，这个线程将抛出InterruptedException异常，其他的等待线程将抛出BrokenBarrierException异常，于是CyclicBarrier 对象就处于损坏状态了。可以通过 isBroken() 判断。

2. CyclicBarrier独占性表现在哪里
CyclicBarrier内部使用独占锁Lock来保证同时只有一个线程调用await方法时候才可以返回，使用loc首先保证了更新计数器count的原子性;另外使用lock的条件变量trip支持了线程间使用 notify，wait 操作进行同步。

3. CyclicBarrier中使用count和parties的区别。
parties：记录线程个数。
count： 一开始等于 parties，每当线程调用await方法后就递减1，当为0时候就表示所有线程都到了屏障。

由于CyclicBarrier是可复用的，使用两个变量原因是用parties始终来记录总的线程个数，当count计数器变为0后，会使用parties赋值给count，已达到复用的作用。
```
public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```

---
*参考资料：*
1. http://gitbook.cn/books/5ada8b951cbb7735a16e5734/index.html
2. 《java并发编程的艺术》
