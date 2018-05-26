# 原理介绍
一个由链表(单向链表)结构组成的有界阻塞队列。此队列的默认和最大长度为Integer.MAX_VALUE。此队列按照先进先出的原则对元素进行排序。

> 双锁（ReentrantLock）：takeLock、putLock，允许读写并行.

![原理图](https://github.com/alanzhang211/learning-note/raw/master/img/LinkedBlockingQueue-item.jpg)

# 源码分析
## 类图
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/LinkedBlockingQueue-class.jpg)

```
    //塞队列所能存储的最大容量，默认的最大容量为Integer.MAX_VALUE
    private final int capacity;
    //当前阻塞队列中的元素数量
    private final AtomicInteger count = new AtomicInteger();
    //链表的头部,节点内的元素为null
    transient Node<E> head;
    //链表的尾部，节点的后继节点为null
    private transient Node<E> last;
    //执行出队操作时需要获取该锁
    private final ReentrantLock takeLock = new ReentrantLock();
    //当队列为空时候执行出队操作的线程会被放入这个条件队列进行等待
    private final Condition notEmpty = takeLock.newCondition();
    /** 执行入队操作时需要获取该锁*/
    private final ReentrantLock putLock = new ReentrantLock();
    //当队列满时候执行进队操作的线程会被放入这个条件队列进行等待
    private final Condition notFull = putLock.newCondition();

    //链表节点
    static class Node<E> {
        //使用item来保存元素本身
        E item;
        //保存当前节点的后继节点
        Node<E> next;
        Node(E x) { item = x; }
    }

```
## 核心方法
### 入队
#### offer(E e)
向队列尾部插入一个元素，如果队列有空闲容量则插入成功后返回 true，如果队列已满则丢弃当前元素然后返回false。该方法是非阻塞的，直接返回。

```
    public boolean offer(E e) {
        //空元素，返回空指针
        if (e == null) throw new NullPointerException();
        //获得队列元素个数
        final AtomicInteger count = this.count;
        //如果队列满，返回false
        if (count.get() == capacity)
            return false;
        int c = -1;
        //构造新节点
        Node<E> node = new Node<E>(e);
        //获得入队锁
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            //队列不满
            if (count.get() < capacity) {
                //入队
                enqueue(node);
                //个数+1
                c = count.getAndIncrement();
                //还有空闲空间，则唤醒notFull的条件队列里面因为调用了notFull.await操作被阻塞的一个线程,j进行入队操作。
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        //c==0表示有一个元素（c初值为-1）
        if (c == 0)
            signalNotEmpty();
        //有元素，入队成功，返回true
        return c >= 0;
    }

    //通知出队阻塞队列中的线程
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }

    //插入元素，尾指针调整
    private void enqueue(Node<E> node) {
        last = last.next = node;
    }
```
#### put(E e)
向队列尾部插入一个元素，如果队列有空闲则插入后直接返回true，如果队列已满则阻塞当前线程直到队列有空闲插入成功后返回true，如果在阻塞的时候被其它线程设置了中断标志，则被阻塞线程会抛出 InterruptedException 异常而返回，该方法是阻塞的。

```
public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            //如果队列满，则线程进行notFull条件队列（此处，同样使用while循环处理，避免notFull.await() 被虚假唤醒了）
            while (count.get() == capacity) {
                notFull.await();
            }
            //入队
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
```
### 出队
#### poll()
从队列头部获取并移除一个元素，如果队列为空则返回 null，该方法是不阻塞的。
```
public E poll() {
        //获取队列元素个数
        final AtomicInteger count = this.count;
        //队列为空，返回null
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        //获得出队锁
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            //如果队列不空
            if (count.get() > 0) {
                //
                x = dequeue();
                //元素个数减1
                c = count.getAndDecrement();
                //前线程移除掉队列里面的一个元素后队列不为空，唤醒调用notEmpty.await被阻塞线程。
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        //当前队列是满的，移除队头元素后队列当前至少有一个空闲位置，可以调用signalNotFull激活notFull的条件队列里的一个线程，进行入队操作
        if (c == capacity)
            signalNotFull();
        return x;
    }


    //出队处理，调整头指针
    private E dequeue() {
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }

    //唤醒notFull等待队列中的线程，入队
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }
```
#### take()
获取当前队列头部元素并从队列里面移除，如果队列为空则阻塞调用线程,直到队列不为空然后返回元素.如果在阻塞的时候被其它线程设置了中标志，则被阻塞线程会抛出InterruptedException 异常而返回。

```
public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            //如果队列为空，阻塞挂起。把当前线程放入notEmpty的条件队列。
            while (count.get() == 0) {
                notEmpty.await();
            }
            //出队
            x = dequeue();
            c = count.getAndDecrement();
            //队列不空，唤醒notEmpty条件队列中等待的线程，进行出队处理。
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        //出队前满，出队后至少有一个空位。
        if (c == capacity)
            signalNotFull();
        return x;
    }
```
#### remove(Object o)
删除队列里面指定元素，有则删除返回 true，没有则返回false。
```
    public boolean remove(Object o) {
        if (o == null) return false;
        //获取两把锁
        fullyLock();
        try {
            //遍历队列
            for (Node<E> trail = head, p = trail.next;
                 p != null;
                 trail = p, p = p.next) {
                 //找到相同元素
                if (o.equals(p.item)) {
                    unlink(p, trail);
                    return true;
                }
            }
            return false;
        } finally {
            fullyUnlock();
        }
    }

    //获取双锁
    void fullyUnlock() {
        takeLock.unlock();
        putLock.unlock();
    }

    //释放双锁
    void fullyUnlock() {
        takeLock.unlock();
        putLock.unlock();
    }

    //移除节点
    void unlink(Node<E> p, Node<E> trail) {
        p.item = null;
        trail.next = p.next;
        if (last == p)
            last = trail;
        //移除后，如果队列有空闲空间，唤醒notFull的条件队列中一个线程，进行入队操作。
        if (count.getAndDecrement() == capacity)
            notFull.signal();
    }
```


### 获取
#### peek()
获取队列头部元素但是不从队列里面移除，如果队列为空则返回 null，该方法是不阻塞的。

```
    public E peek() {
        //队列空，返回null
        if (count.get() == 0)
            return null;
        //获得出队锁
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            //获取第一个节点
            Node<E> first = head.next;
            //first是否为null（2次检查）
            if (first == null)
                return null;
            else
                return first.item;
        } finally {
            takeLock.unlock();
        }
    }
```
# FAQ
1. peek()中获取到first节点后，还要判断first是否为null?

判断队列是否为空和获取锁的过程不是原子操作，可能在获取takeLock之前，其他线程执行了poll或take操作，导致队列为空。

2. LinkedBlockingQueue与ArrayBlockingQueue的比较
+ ArrayBlockingQueue中在入队列和出队列操作过程中，使用的是同一个lock，读取和操作的都无法做到并行；LinkedBlockingQueue读取和插入操作所使用的锁是两个不同的lock，它们之间的操作互相不受干扰，因此两种操作可以并行完成。
+ LinkedBlockingQueue相比ArrayBlockingQueue具有更好的扩容性。ArrayBlockingQueue容量固定，可以更好的对吸能进行预测；LinkedBlockingQueue不指定，默认Integer.MAX_VALUE。创建节点是动态地，节点出队列会被GC回收，一种程度上可以理解为“无界队列”。任务多时，入队处理，可能会出现内存溢出的情况。

---
*参考资料:*
+ https://blog.csdn.net/u010887744/article/details/73010691
+ https://www.jianshu.com/p/cc2281b1a6bc
