# 原理介绍
ReentrantLock 可重入锁。就是支持重进入的锁，它表示该锁能够支持一个线程对
资源的重复加锁。除此之外，该锁的还支持获取锁时的公平和非公平性选择。

对于ReentrantLock的实现来说，state可以用来表示当前线程获取锁的可重入次数。

> synchronized关键字隐式的支持重进入，比如一个synchronized修饰的递归方
法，在方法执行时，执行线程在获取了锁之后仍能连续多次地获得该锁。

ReentrantLock调用lock方法再次获取锁，以获取到锁的线程不会被阻塞。

## 两种锁
> 公平锁、非公平锁。默认是非公平锁。

```
    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

```

在绝对时间上，先对锁进行获取的请求一定先
被满足，那么这个锁是公平的。反之，是不公平的。公平的获取锁，也就是等待时间最长的线
程最优先获取锁，也可以说锁获取是顺序的

## 可重入性
重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞。

### 两个问题
1. **线程再次获取锁**。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再
次成功获取。

2. **锁的最终释放**。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到
该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁
被释放时，计数自减，当计数等于0时表示锁已经成功释放。

> 通过自定义同步器实现锁的获取和释放。

# 源码分析
ReentrantLock 内部维护了2个同步器：FairSync（公平）、NonfairSync（非公平）。

以上另个同步器均继承与Sync，而Sync继承与AQS同步器。这样，通过AQS内部的同步器机制，间接实现了公平和非公平。

> AQS是锁同步的基础组件。


## 非公平锁
### 非公平锁的获取
```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
            }
        }
        //当前线程是该锁持有者
        else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

### 公平锁释放
```
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())//当前线程未占有锁
        throw new IllegalMonitorStateException();
        boolean free = false;
        //如果当前可重入次数为0，则清空锁持有线程
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        //更新可重入次数
        setState(c);
        return free;
    }
```

## 公平锁
### 公平锁获取

```
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //公平策略（如果AQS如果当前 AQS 队列为空或者当前线程节点是 AQS 的第一个节点则）
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //当前线程是该锁持有者
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

与非公平锁不同之处：

```
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
> h == t 则说明当前队列为空则直接返回 false，如果 h!=t 并且 s==null 说明有一个元素将要作为 AQS 的第一个节点入队列，那么返回 true, 如果 h!=t 并且 s!=null 并且 s.thread != Thread.currentThread() 则说明队列里面的第一个元素不是当前线程则返回 true。

### 公平锁释放
 逻辑同非公平锁

> 释放锁不存在公平性和非公平性。

## 公平锁和非公平锁对比

公平锁 | 非公平锁
---|---
带来更多的线程切换|可能出现线程“饥饿”问题；但可以保证更大的吞吐量



# 应用场景
## 已经在执行中则不再执行（有状态执行）
1. 用在定时任务时，如果任务执行时间可能超过下次计划执行时间，确保该有状态任务只有一个正在执行，忽略重复触发。
2. 用在界面交互时点击执行较长时间请求操作时，防止多次点击导致后台重复执行（忽略重复触发）。

> 以上两种情况多用于进行非重要任务防止重复执行，（如：清除无用临时文件，检查某些资源的可用性，数据备份操作等）。

```
private ReentrantLock lock = new ReentrantLock();
...
if (lock.tryLock()) {  //如果已经被lock，则立即返回false不会等待，达到忽略操作的效果
    try {
        //操作
    } finally {
        lock.unlock();
       }

    }
```
## 同步执行
主要是防止资源使用冲突，保证同一时间内只有一个操作可以使用该资源。（如：文件操作，同步消息发送，有状态的操作等）。

```
try {
    lock.lock(); //如果被其它资源锁定，会在此等待锁释放，达到暂停的效果
        //操作
    } finally {
        lock.unlock();
    }
```

## 尝试等待执行
如果发现该操作已经在执行，则尝试等待一段时间，等待超时则不执行。
用来防止由于资源处理不当长时间占用导致死锁情况（大家都在等待资源，导致线程队列溢出）。

```
try {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {  //如果已经被lock，尝试等待5s，看是否可以获得锁，如果5s后仍然无法获得锁则返回false继续执行
         try {
             //操作
            } finally {
                lock.unlock();
            }
        }
    } catch (InterruptedException e) {
        e.printStackTrace(); //当前线程被中断时(interrupt)，会抛InterruptedException       
    }
```
## 等待执行，响应中断
synchronized与Lock在默认情况下是不会响应中断(interrupt)操作，会继续执行完。lockInterruptibly()提供了可中断锁来解决此问题。

这种情况主要用于**取消**某些操作对资源的占用。如：（取消正在同步运行的操作，来防止不正常操作长时间占用造成的阻塞）

```
try {
        lock.lockInterruptibly();
        //操作
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        lock.unlock();
    }
```

# 实战分析
实现一个简单的线程安全的 list。

```
public static class ReentrantLockList {
        //线程不安全的list
        private ArrayList<String> array = new ArrayList<String>();
        //独占锁
        private volatile ReentrantLock lock = new ReentrantLock();
        //添加元素
        public void add(String e) {
            lock.lock();
            try {
                array.add(e);
            } finally {
                lock.unlock();
            }
        }
        //删元素
        public void remove(String e) {
            lock.lock();
            try {
                array.remove(e);
            } finally {
                lock.unlock();
            }
        }
        //获取数据
        public String get(int index) {
            lock.lock();
            try {
                return array.get(index);
            } finally {
                lock.unlock();
            }
        }
```

# FAQ
## ReentrantLock与synchronized的区别
1. 与synchronized相比，ReentrantLock提供了更多，更加全面的功能，具备更强的扩展性。例如：时间锁等候，可中断锁等候，锁投票。
2. ReentrantLock还提供了条件Condition，对线程的等待、唤醒操作更加详细和灵活，所以在多个条件变量和高度竞争锁的地方，ReentrantLock更加适合（以后会阐述Condition）。
3. ReentrantLock提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而synchronized则一旦进入锁请求要么成功要么阻塞，所以相比synchronized而言，ReentrantLock会不容易产生死锁些。
4. ReentrantLock支持更加灵活的同步代码块，但是使用synchronized时，只能在同一个synchronized块结构中获取和释放。注：ReentrantLock的锁释放一定要在finally中处理，否则可能会产生严重的后果。
5. ReentrantLock支持中断处理，且性能较synchronized会好些。

---
*参考资料*
1. 《java并发编程的艺术》
2. https://my.oschina.net/noahxiao/blog/101558
3. http://gitbook.cn/books/5ac8a7e72a04fd6c83713956/index.html
