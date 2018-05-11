# 原理介绍
不同于ReentrantLock的排他性，ReentrantReadWriteLock持有两把锁。一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

## 好处
1. 在同一时刻可以允许多个读线程访问。
2. 简化读写交互场景的编程方式。
3. 读写锁能够提供比排它锁更好的并发性和吞吐量。

## 特性

特性 | 说明
---|---
公平选择 | 支持非公平（默认）和公平的锁获取方式，吞吐量：非公平优于公平
重进入 |支持重入：读线程获取读锁，能够再次获取读锁；写线程获取写锁喉，可再次获取写锁，同时也可以获取读锁。最多支持65535个递归写入锁和65535个递归读取锁。
锁降级|遵循获取写锁、获取读锁再释放写锁的次序，写锁能够降级成为读锁。
锁获取中断|读取锁和写入锁都支持获取锁期间被中断。这个和独占锁一致。
条件变量|写入锁提供了条件变量(Condition)的支持，这个和独占锁一致，但是读取锁却不允许获取条件变量，将得到一个UnsupportedOperationException异常。

> **不支持锁升级**：读取锁是不能直接升级为写入锁的。因为获取一个写入锁需要释放所有读取锁，所以如果有两个读取锁视图获取写入锁而都不释放读取锁时就会发生死锁。

## 锁降级
锁降级指的是写锁降级成为读锁。如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。锁降级是指把持住（当前拥有的）写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。

降级逻辑：

```
//持有写锁
writeLock.lock();
try {

    业务逻辑

    //获取写锁
    readLock.lock();
} finally {
    //释放写锁
    writeLock.unlock();
}
// 锁降级完成，写锁降级为读锁
```

## 两把锁
ReentrantReadWriteLock使用一个state整型变量维护读写状态。“按位切分”，其高16位表示读状态也就是获取该读锁的线程个数，低16位表示获取到写锁的线程的可重入次数。并且通过CAS保证同时只有一个线程可以更新state的值。

![读写锁](https://github.com/alanzhang211/learning-note/raw/master/img/ReentrantReadWriteLock-wei.jpg)

### 读锁
读锁是共享锁；一个线程对资源加了读锁，其他线程可以继续加读锁；持有读锁，其他的写锁被阻塞。

### 写锁
写锁是独占锁，持有写锁的线程，可以对资源加读锁（锁降级），其他写请求被阻塞。

### 读写锁进入条件

读锁 | 写锁
---|---
没有其他线程的写锁 | 没有其他线程的读锁
没有写请求或者有写请求，但调用线程和持有锁的线程是同一个 | 没有其他线程的写锁

### 与Condition的关系
读锁不允许newConditon获取Condition接口，而写锁的newCondition接口实现方法同ReentrantLock。

![](https://github.com/alanzhang211/learning-note/raw/master/img/ReentrantReadWriteLock-condition.png)

### 特点

锁情况 | 互斥性
---|---
读-读 | 不互斥
读-写 | 互斥
写-写 | 互斥



# 源码分析
## 写锁
写锁是使用的 WriteLock 来实现的。

### 获取
void lock() 写锁是个独占锁，同时只有一个线程可以获取该锁。

```
public void lock() {
       sync.acquire(1);
   }

    public final void acquire(int arg) {
        // sync重写的tryAcquire方法
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
> lock() 内部调用了AQS的acquire方法，其中的tryAcquire是ReentrantReadWriteLock内部sync类重写的。

```
protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            //c!=0说明读锁或者写锁已经被某线程获取
            if (c != 0) {
                //w=0说明已经有线程获取了读锁或者w!=0并且当前线程不是写锁拥有者，则返回false
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
               //说明某线程获取了写锁，判断可重入个数
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //设置可重入数量
                setState(c + acquires);
                return true;
            }

            //c==0,表示目前没有线程获取到读锁和写锁；第一个写线程获取写锁
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

非公平锁下的writerShouldBlock（）实现；

```
final boolean writerShouldBlock() {
       return false; // writers can always barge
   }
```
对于非公平锁来说固定返回false，则说明抢占式执行CAS尝试获取写锁，获取成功则设置当前锁的持有者为当前线程返回true,否者返回false。

对于公平锁的实现：
```
final boolean writerShouldBlock() {
  return hasQueuedPredecessors();
}
```
使用hasQueuedPredecessors来判断当前线程节点是否有前驱节点,如果有则当前线程放弃获取写锁的权限直接返回false。

### 释放
void unlock() 尝试释放锁。如果当前线程持有该锁，调用该方法会让该线程对该线程持有的AQS状态值减一，如果减去1后当前状态值为0则当前线程会释放对该锁的持有，否者仅仅减一而已。如果当前线程没有持有该锁调用了该方法则会抛出 IllegalMonitorStateException异常。

释放过程即AQS的state低16位同步递减为0的过程，当state的高16位都为0表示写锁释放完毕，唤醒head节点后下一个SIGNAL状态的节点的线程。如果该写锁占有线程未释放写锁前还占用了读锁，那么写锁释放后该线程就完全转换成了读锁的持有线程。

```
public void unlock() {
            sync.release(1);
        }

        public final boolean release(int arg) {
        //调用ReentrantReadWriteLock中sync实现的tryRelease方法
        if (tryRelease(arg)) {
            //激活阻塞队列里面的一个线程
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

protected final boolean tryRelease(int releases) {
            //看是否是写锁拥有者调用的unlock
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
           //获取可重入值，这里没有考虑高16位，因为写锁时候读锁状态值肯定为0
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
           //如果写锁可重入值为0则释放锁，否者只是简单更新状态值。
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```


## 读锁
ReentrantReadWriteLock 中写锁是使用的 ReadLock 来实现的。

### 获取
void lock() 获取读锁，如果当前没有其它线程持有写锁，则当前线程可以获取读锁，AQS的高16位的值会增加1，然后方法返回。否者如果其它有一个线程持有写锁，则当前线程会被阻塞。

```
public void lock() {
       sync.acquireShared(1);
   }

    public final void acquireShared(int arg) {
        //调用ReentrantReadWriteLock中的sync的tryAcquireShared方法
        if (tryAcquireShared(arg) < 0)
           //调用AQS的doAcquireShared方法，加入阻塞队列。
            doAcquireShared(arg);
    }

//重写tryAcquireShared方法
protected final int tryAcquireShared(int unused) {
    //获取当前状态值
    Thread current = Thread.currentThread();
    int c = getState();
    //判断是否写锁被占用,写锁是排他锁，如果被占有直接返回，无法获取读锁。
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;

    //获取读锁计数
    int r = sharedCount(c);
    //尝试获取锁，多个读线程只有一个会成功，不成功的进入下面fullTryAcquireShared进行重试
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        //第一个线程获取读锁
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        //如果当前线程是第一个获取读锁的线程
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            //记录最后一个获取读锁的线程或记录其它线程读锁的可重入数
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != current.getId())
                //readHolds记录当前线程获取读锁的可重入次数。
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    //类似tryAcquireShared，但是是自旋获取
    return fullTryAcquireShared(current);
}
```
非公平锁的 readerShouldBlock 实现：
```
final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
   }

   final boolean apparentlyFirstQueuedIsExclusive() {
   Node h, s;
   return (h = head) != null &&
       (s = h.next)  != null &&
       !s.isShared()         &&
       s.thread != null;
    }
```
公平锁的readerShouldBlock实现：
```
final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
```
同写锁。

### 释放
void unlock()：读锁的释放过程即AQS的state高16位同步递减为0的过程，当state的高16位都为0表示读锁释放完毕。

```
public void unlock() {
        sync.releaseShared(1);
    }

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

//重写tryReleaseShared方法
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    //如果当前线程是第一个获取读锁线程
    if (firstReader == current) {
        //如果可重入次数为1
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else//否者可重入次数减去1
            firstReaderHoldCount--;
    } else {
        //如果当前线程不是最后一个获取读锁线程，则从threadlocal里面获取
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != current.getId())
            rh = readHolds.get();
        //如果可重入次数<=1则清除threadlocal,避免垃圾堆积。
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        //可重入次数减去一
        --rh.count;
    }

    //循环直到自己的读计数-1 cas更新成功
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```


# 应用场景
1. 多线程环境。
2. 并且共同访问同一个资源数据。
3. 要求可以共享读数据,同时读。
4. 不能同时写数据。

# 实战分析
## 缓存
较为精确的控制缓存，使用ReentrantReadWriteLock。

```
class CachedData {
   Object data;
   volatile boolean cacheValid;
   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock();
     if (!cacheValid) {
       // Must release read lock before acquiring write lock
       rwl.readLock().unlock();
       rwl.writeLock().lock();
       try {
         // Recheck state because another thread might have
         // acquired write lock and changed state before we did.
         if (!cacheValid) {
           data = ...
           cacheValid = true;
         }
         // Downgrade by acquiring read lock before releasing write lock
         rwl.readLock().lock();
       } finally {
         rwl.writeLock().unlock(); // Unlock write, still hold read
       }
     }

     try {
       use(data);
     } finally {
       rwl.readLock().unlock();
     }
   }
 }}
```

# FAQ
1. 锁降级中读锁的获取是否必要？

答案是必要的。主要是为了保证数据的可见性，如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，那么当前线程无法感知线程T的数据更新。如果当前线程获取读锁(还未释放写锁，其他线程获取写锁将被阻塞)，即遵循锁降级的步骤，则线程T将会被阻塞，直到当前线程使用数据并释放写锁之后，线程T才能获取写锁进行数据更新。

---
*参考资料：*
1. 《java并发编程的艺术》
2. http://gitbook.cn/books/5ac8a7e72a04fd6c83713956/index.html
