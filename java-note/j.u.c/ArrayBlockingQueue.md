# 原理介绍
一个由数组实现的有界阻塞队列。该队列采用FIFO的原则对元素进行排序添加的。

ArrayBlockingQueue为有界且固定，其大小在构造时由构造函数来决定，确认之后就不能再改变了。ArrayBlockingQueue支持对等待的生产者线程和使用者线程进行排序的可选公平策略，但是在默认情况下不保证线程公平的访问，在构造时可以选择公平策略（fair = true）。公平性通常会降低吞吐量，但是减少了可变性和避免了“不平衡性”。

![item](https://github.com/alanzhang211/learning-note/raw/master/img/ArrayBlockingQueue-item.jpg)

# 源码分析
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/ArrayBlockingQueue.jpg)

```
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable {
        //一个定长数组，维护ArrayBlockingQueue的元素
        final Object[] items;
        //出队下标(对首)
        int takeIndex;
        //标示入队元素下标(对尾)
        int putIndex;
        //元素个数
        int count;
        // 重入锁(保证入队出队的原子性)
        final ReentrantLock lock;
        // 出对条件队列
        private final Condition notEmpty;
        // 入列条件队列
        private final Condition notFull;
        //队列内部迭代器，是有一定弹性的弱一致性迭代器
        transient ArrayBlockingQueue.Itrs itrs;
    }
```
> 一把锁，两个条件队列。默认是，非公平锁阻塞队列。

```
 public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
  }

    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

## 核心方法
### 插入
#### add(e)
将指定的元素插入到此队列的尾部（如果立即可行且不会超过该队列的容量），在成功时返回 true，如果此队列已满，则抛出 IllegalStateException。

```
 public boolean add(E e) {
        return super.add(e);
    }

    public boolean add(E e) {
        //调用offer方法
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }
```

#### offer(E e)
向队列尾部插入一个元素，如果队列有空闲容量则插入成功后返回true，如果队列已满则丢弃当前元素然后返回false，如果e元素为null则抛出NullPointerException异常，另外该方法是不阻塞的。
```
public boolean offer(E e) {
        //null检测。
        checkNotNull(e);
        //获取独占锁同步
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //队列满，返回false，不阻塞
            if (count == items.length)
                return false;
            else {
                //否则入队
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
```

#### enqueue(E x)
元素放入队列。

```
private void enqueue(E x) {
        //元素入队
        final Object[] items = this.items;
        items[putIndex] = x;
        //计算下一个元素应该存放的下标
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        //激活 notEmpty 的条件队列中被阻塞的线程
        notEmpty.signal();
    }
```

#### put(E e)
将指定的元素插入此队列的尾部，如果该队列已满，则等待可用的空间。

```
    public void put(E e) throws InterruptedException {
        //检测是否为null
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        //获得锁，可被中断
        lock.lockInterruptibly();
        try {
            //如果队列满，则把当前线程放入notFull管理的条件队列，阻塞当前线程
            while (count == items.length)
                notFull.await();
            //入队，插入原色
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

### 出队
#### remove(Object o)
从队列中移除一个元素，如果有多个，只移除第一个。
```
public boolean remove(Object o) {
        if (o == null) return false;
        //拿到队列
        final Object[] items = this.items;
        final ReentrantLock lock = this.lock;
        //获得锁
        lock.lock();
        try {
            //队列非空
            if (count > 0) {
                //入队索引
                final int putIndex = this.putIndex;
                //出队遍历游标
                int i = takeIndex;
                do {
                    //如果找到相同的元素，移除并返回true
                    if (o.equals(items[i])) {
                        //移除原色
                        removeAt(i);
                        return true;
                    }
                    if (++i == items.length)
                        i = 0;
                } while (i != putIndex);
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
```

#### poll()
从队列头部获取并移除一个元素，如果队列为空则返回 null，该方法是不阻塞的。

```
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //当前队列为空返回null，否则，出队列。
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
```
#### dequeue()
该方法主要是从列头（takeIndex 位置）取出元素，同时如果迭代器itrs不为null，则需要维护下该迭代器。最后调用notFull.signal()唤醒入列线程。

```
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        //获取元素
        E x = (E) items[takeIndex];
        //数组元素设为null
        items[takeIndex] = null;
        //更新队列头
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        //迭代器itrs不为null，则需要维护下该迭代器
        if (itrs != null)
            itrs.elementDequeued();
        //发送信号激活notFull条件队列里面的一个线程
        notFull.signal();
        return x;
    }
```

#### take()
获取并移除此队列的头部，在元素变得可用之前一直等待。

```
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        //获得锁，支持中断
        lock.lockInterruptibly();
        try {
            //队列为空，则等待，阻塞当前线程，一直到队列有元素
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```
### 获取
#### peek()
获取队列头部元素，但是不从队列里面移除，如果队列为空则返回 null，该方法是不阻塞的。

```
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //返回元素值
            return itemAt(takeIndex);
        } finally {
            lock.unlock();
        }
    }

    final E itemAt(int i) {
        return (E) items[i];
    }
```

# FAQ
1. offer(E e) 和put(E e)的区别？
+ 如果队列满，offer不阻塞当前执行线程，直接放回；而put会使用notFull.await阻塞当前线程。
+ put支持中断。

2. put(E e)中判断队列是否为空，使用while而不是if？为什么？

考虑到当前线程被虚假唤醒的问题，也就是其它线程没有调用notFull的singal方法时候notFull.await()在某种情况下会自动返回。使用while循环假如notFull.await()被虚假唤醒了，那么循环在检查一下当前队列是否是满的，如果是则再次进行等待。

---
*参考资料：*
1. http://gitbook.cn/books/5ad09cdf7e92727030f21eec/index.html
2. http://cmsblogs.com/?p=2381
