# 实现原理
一个使用优先级队列实现的无界阻塞队列。在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。

DelayQueue中内部使用的是PriorityQueue存放数据，出队/入队依赖于优先级队列，简化了代码逻辑；使用ReentrantLock实现线程同步；实现了Delayed接口，此接口定义了一个获取当前剩余时间的接口。

## Leader/Followers模式
Leader/Followers模式是多个工作线程轮流获得事件源集合，轮流监听、分发并处理事件的一种模式。在任意时间点，程序都仅有一个leader线程，n个处理中的线程，m个追随线程。如下图示：

![Leader/Followers](https://github.com/alanzhang211/learning-note/raw/master/img/Leader-Followers.jpg)

1. 线程有3种状态：领导leading，处理processing，追随following
2. 假设共N个线程，其中只有1个leading线程（等待任务），n个processing线程（处理），余下有N-1-n个following线程（空闲）
3. 有一把锁，谁抢到就是leading，它的任务是等待新任务。
4. 事件/任务来到时，leading线程会对其进行处理，从而转化为processing状态，处理完成之后，又转变为following
5. 丢失leading后，following会尝试抢锁，抢到则变为leading，否则保持following
6. following不干事，就是抢锁，力图成为leading

# 源码分析
## 类图
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/DelayQueue-class.png)

```
//同步锁
private final transient ReentrantLock lock = new ReentrantLock();
//使用优先级队列存储元素
private final PriorityQueue<E> q = new PriorityQueue<E>();
//leader线程（Leader/Followers模式）
private Thread leader = null;
//条件对象，当新元素到达，或新线程可能需要成为leader时被通知
private final Condition available = lock.newCondition();
```

## 核心方法
### 入队
因为队列是无界的，所以入队不会阻塞。

#### add(E e)
向延迟队列插入元素,成功返回true。
```
public boolean add(E e) {
        // 直接调用offer并返回
        return offer(e);
    }
```

#### put(E e)
向延迟队列插入元素，无返回值。

```
public void put(E e) {
        offer(e);
    }
```
#### offer(E e)
插入元素到队列，插入元素要实现Delayed接口。
```
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        // 获得锁
        lock.lock();
        try {
            // 向优先队列插入元素
            q.offer(e);
            // 若在此之前队列为空，则置空leader，并通知条件对象
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```

### 出队
#### poll()
获取并移除队头过期元素，否者返回null。
```
    public E poll() {
        //获得锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取队首元素
            E first = q.peek();
            //如果队首元素为null或则还没到过期时间，直接返回null
            if (first == null || first.getDelay(NANOSECONDS) > 0)
                return null;
            else
                //委托给优先级队列，出队列
                return q.poll();
        } finally {
            lock.unlock();
        }
    }
```
#### take()
获取并移除队首元素，该方法将阻塞，直到队列中包含达到延迟时间的元素。
```
public E take() throws InterruptedException {
        //获得锁，支持中断
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            循环获取过期的队首元素
            for (;;) {
                //获得队首元素
                E first = q.peek();
                //若元素为空，线程进入等待条件队列，在offer方法中会调用条件对象的通知方法
                if (first == null)
                    available.await();
                else {
                    //获得延迟时间
                    long delay = first.getDelay(NANOSECONDS);
                    //达到延迟时间，委派给优先级队列，进行出队处理
                    if (delay <= 0)
                        return q.poll();
                    //队首引用设为null
                    first = null; // don't retain ref while waiting
                    //如果leader不为null
                    if (leader != null)
                        //等待，leader移除
                        available.await();
                    else {
                        //否则，设置当前线程为leader，并设置延迟时间
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            //leader为null，并且队首元素不为null，通知等待队列中满足条件的线程
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```
### 获取
#### peek()
获取但不移除队首元素，如果队列为空,返回null。若队列不为空，该方法换回队首元素，不论是否达到延迟时间。
```
public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //委派给优先级队列，获取队首元素
            return q.peek();
        } finally {
            lock.unlock();
        }
    }
```

# 应用场景
1. 关闭空闲连接。服务器中，有很多客户端的连接，空闲一段时间之后需要关闭。
2. 缓存。缓存中的对象，超过了空闲时间，需要从缓存中移出。
3. 任务超时处理。在网络协议滑动窗口请求应答式交互时，处理超时未响应的请求。

---
*参考资料：*
1. https://zhuanlan.zhihu.com/p/30567371
2. https://blog.csdn.net/xiaoxufox/article/details/51880645
