# 实现原理
Fork/Join是java 7提供的一种并行执行框架。把大任务切分成若干小任务，然后再将各个小任务的结果进行汇总。其内部思想也就是一种“分治”。

![fork-join](https://github.com/alanzhang211/learning-note/raw/master/img/fork-join.png)

**算法描述**
```
Result solve(Problem problem) {
	if (problem is small)
		directly solve problem
		else {
			split problem into independent parts
			fork new subtasks to solve each part
			join all subtasks
			compose result from subresults
		}
}
```
## 设计
1. 分割任务：将一个大任务进行切分，切分成很多足够小的任务。
2. 执行结果合并：将分割的子任务放在双端队列里，启动线程分别处理队列中的任务；子任务执行完，将结果统一放在一个队列中，启用一个线程从队列里拿数据，然后合并这些数据。

以上两个工作，Fork/Join框架交给了ForkJoinTask和ForkJoinPool来处理。

**ForkJoinTask**：使用ForkJoin框架创建任务，它提供了两种操作：fork()和join()操作机制。在使用中，通常使用两个子类：

RecursiveAction：用于没有返回结果的任务。

RecursiveTask：用于有返回结果的任务。

**ForkJoinPool**：ForkJoinTask需要通过ForkJoinPool来执行。

被切分的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。

## Work−Stealing
工作窃取算法，指某个线程可以从其他线程执行队列中窃取任务来执行。被窃取任务线程从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

![Work−Stealing](https://github.com/alanzhang211/learning-note/raw/master/img/Work%E2%88%92Stealing.png)

每一个工作线程维护自己的调度队列中的可运行任务，工作队列是一个双端队列（不仅支持后进先出——LIFO的push和pop操作，还支持先进先出——FIFO的take操作）。

工作线程使用后进先出——LIFO(最早的优先)的顺序，通过弹出任务来处理队列中的任务；使用先进先出——FIFO规则用于获取别的任务。

**优点**：充份利用线程进行并行计算，减少了线程间的竞争（使用了双端队列）。

**缺点**：在某些情况下还是存在竞争，比如：只有一个任务时。该算法会消耗更多的系统资源，比如：创建了多个线程和双端队列。

# 源码分析
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/fork-join-class.png)

ForkJoinPool依赖于ForkJoinTask和ForkJoinWorkerThread。

ForkJoinTask实现Future接口，ForkJoinWorkerThread继承Thread，负责执行任务。

## 核心方法
#### fork
把新创建的子任务提交给当前线程，由当前线程push到它自身的队列数组中。
```
    public final ForkJoinTask<V> fork() {
        Thread t;
        //将任务加入到当前线程的队列中
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }

    //将任务加入ForkJoinTask数组，并唤醒或创建线程去执行任务
    final void push(ForkJoinTask<?> task) {
        ForkJoinTask<?>[] a; ForkJoinPool p;
        int b = base, s = top, n;
        if ((a = array) != null) {    // ignore if queue removed
            int m = a.length - 1;     // fenced write for task visibility
            U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
            U.putOrderedInt(this, QTOP, s + 1);
            if ((n = s - b) <= 1) {
                if ((p = pool) != null)
                    //唤醒或创建线程去只执行任务
                    p.signalWork(p.workQueues, this);
            }
            else if (n >= m)
                growArray();
        }
    }
```

## join
阻塞当前线程获得结果。
```
    public final V join() {
        int s;
        if ((s = doJoin() & DONE_MASK) != NORMAL)
            reportException(s);
        return getRawResult();
    }

    //获取结果
    private int doJoin() {
        int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
        //(s = status) < 0完成，直接返回；
        //否则从当前线程的任务数组中拿出该任务进行执行，如果正常结束，返回执行结果；
        //如果任务未完成，可能阻塞，等待执行
        //否则阻塞直到完成
        return (s = status) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (w = (wt = (ForkJoinWorkerThread)t).workQueue).
            tryUnpush(this) && (s = doExec()) < 0 ? s :
            wt.pool.awaitJoin(w, this, 0L) :
            externalAwaitDone();
    }
```

# 应用场景
适合计算密集型（CPU密集）的任务。

> 临界值选取过大，任务分割的不够细，则不能充分利用 CPU；临界值选取过小，则任务分割过多，可能产生过多的子任务，导致过多的线程间的切换和加重 GC 的负担从而影响了效率。

---
*参考资料：*
1. 《java并发编程的艺术》
2. http://gee.cs.oswego.edu/dl/papers/fj.pdf
