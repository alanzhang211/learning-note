# 原理介绍
一个支持优先级排序的无界阻塞队列。底层采用二叉堆来实现。默认情况下元素采取自然顺序升序排列。也可以自定义类实现compareTo()方法来指定元素排序规则，或者初始化PriorityBlockingQueue时，指定构造参数Comparator来对元素进行排。

内部使用数组作为元素存储的数据结构，这个数组是可扩容的，当当前元素个数>=最大容量时候会通过算法扩容，出队时候始终保证出队的元素是堆树的根节点，而不是在队列里面停留时间最长的元素。

## 二叉堆
二叉堆是一种特殊的堆，就结构性而言就是完全二叉树或者是近似完全二叉树，满足树结构性和堆序性。树机构特性就是完全二叉树应该有的结构，堆序性则是：父节点的键值总是保持固定的序关系于任何一个子节点的键值，且每个节点的左子树和右子树都是一个二叉堆。它有两种表现形式：最大堆、最小堆。

+ 最大堆：父节点的键值总是大于或等于任何一个子节点的键值。
+ 最小堆：父节点的键值总是小于或等于任何一个子节点的键值。

PriorityBlockingQueue采用最小堆实现。

![最小堆](https://github.com/alanzhang211/learning-note/raw/master/img/%E4%BA%8C%E5%8F%89%E5%A0%86-%E6%9C%80%E5%B0%8F%E5%A0%86.png)

# 源码分析
## 类图
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/PriorityBlockingQueue.png)

```
    //默认容量
    private static final int DEFAULT_INITIAL_CAPACITY = 11;
    //最大容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    //队列数组（二叉堆）
    private transient Object[] queue;
    //队列中元素个数
    private transient int size;
    //比较器，如果空，则使用素的compareTo方法进行比较来确定元素的优先级，这意味着队列元素必须实现了Comparable接口。
    private transient Comparator<? super E> comparator;
    //同步锁
    private final ReentrantLock lock;
    //出队条件队列
    private final Condition notEmpty;
    //自旋锁，用 CAS 操作来保证同时只有一个线程可以扩容队列，状态为0或者1，其中0表示当前没有在进行扩容，1标示当前正在扩容。
    private transient volatile int allocationSpinLock;
    //优先队列：主要用于序列化，这是为了兼容之前的版本。只有在序列化和反序列化才非空。
    private PriorityQueue<E> q;
```
> 一把锁，一个条件队列，一个数组（二叉堆），一个比较器。

## 核心方法
### 入队
#### put(E e)
将指定元素插入此优先级队列。PriorityBlockingQueue是无界的，所以不可能会阻塞。
```
public void put(E e) {
        //调用offer方法处理
        offer(e);
    }
```

#### offer(E e)
将指定元素插入此优先级队列。PriorityBlockingQueue是无界的，所以不会放回false。
```
    public boolean offer(E e) {
        //空元素返回空指针异常
        if (e == null)
            throw new NullPointerException();
        //获得同步锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        int n, cap;
        Object[] array;
        //如果当前元素个数大于队列容量，进行扩容
        while ((n = size) >= (cap = (array = queue).length))
            tryGrow(array, cap);
        //插入元素
        try {
            //获得比较器
            Comparator<? super E> cmp = comparator;
            //比较器未空，使用默认自然顺
            if (cmp == null)
                siftUpComparable(n, e, array);
            else
                //使用自定义比较器
                siftUpUsingComparator(n, e, array, cmp);
            //元素个数+1
            size = n + 1;
            //唤醒出队条件队列中等待的线程
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
        return true;
    }

    //采用自然排序
    private static <T> void siftUpComparable(int k, T x, Object[] array) {
        Comparable<? super T> key = (Comparable<? super T>) x;
        //队列元素大于0
        while (k > 0) {
            //获得父节点（数组索引-1后，右移1位，即（n-1）/2）
            int parent = (k - 1) >>> 1;
            //拿到父元素
            Object e = array[parent];
            //如果插入元素大于父元素，退出循环
            if (key.compareTo((T) e) >= 0)
                break;
            //交互
            array[k] = e;
            k = parent;
        }
        //插入数据
        array[k] = key;
    }
    //采用所指定的比较器
    private static <T> void siftUpUsingComparator(int k, T x, Object[] array,
                                       Comparator<? super T> cmp) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = array[parent];
            //使用比较器比较
            if (cmp.compare(x, (T) e) >= 0)
                break;
            array[k] = e;
            k = parent;
        }
        array[k] = x;
    }
```
#### tryGrow
扩容。
```
    private void tryGrow(Object[] array, int oldCap) {
        //释放锁
        lock.unlock(); // must release and then re-acquire main lock
        Object[] newArray = null;
        //使用cas扩容，只有一个线程可以进行扩容
        if (allocationSpinLock == 0 &&
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                     0, 1)) {
            try {
                //分配这么多新的空间，oldGap<64则扩容新增oldcap+2,否者扩容50%，并且最大为MAX_ARRAY_SIZE
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) : // grow faster if small
                                       (oldCap >> 1));
                if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                    int minCap = oldCap + 1;
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();
                    newCap = MAX_ARRAY_SIZE;
                }
                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];
            } finally {
                allocationSpinLock = 0;
            }
        }
        //第一个线程cas成功后，第二个线程会进入这个地方，然后第二个线程让出cpu，尽量让第一个线程获取锁，但是这得不到肯定的保证。
        if (newArray == null) // back off if another thread is allocating
            Thread.yield();
        //获取锁
        lock.lock();
        //复制原来的数组的值到新的数组中
        if (newArray != null && queue == array) {
            queue = newArray;
            System.arraycopy(array, 0, newArray, 0, oldCap);
        }
    }
```


### 出队
#### take()
获取队列内部堆树的根节点元素，如果队列为空则阻塞。
```
public E take() throws InterruptedException {
    //获取锁，可被中断
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {

        //如果队列为空，则阻塞，把当前线程放入notEmpty的条件队列(使用while循环，防止假唤醒)
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```

#### poll()
获取队列内部堆树的根节点元素，如果队列为空，则返回 null。
```
    public E poll() {
        //获得锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //出队
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    private E dequeue() {
        // 队列位空，返回null
        int n = size - 1;
        if (n < 0)
            return null;
        else {
            Object[] array = queue;
            //获取堆的根节点（数组第一个元素）
            E result = (E) array[0];
            //获取队尾元素
            E x = (E) array[n];
            //将队尾设置为null
            array[n] = null;
            Comparator<? super E> cmp = comparator;
            //将队尾元素插入数组下标为0的位置，与两个子节点值进行比较，调整堆
            if (cmp == null)
                //使用自然排序处理
                siftDownComparable(0, x, array, n);
            else
                //使用自定义比较器
                siftDownUsingComparator(0, x, array, n, cmp);
            size = n;
            return result;
        }
    }
    //采用默认排序
    private static <T> void siftDownComparable(int k, T x, Object[] array,
                                               int n) {
        if (n > 0) {
            Comparable<? super T> key = (Comparable<? super T>)x;
            //最后一个叶节点父节点位置
            int half = n >>> 1;           // loop while a non-leaf
            while (k < half) {
                //待调整位置左节点位置
                int child = (k << 1) + 1; // assume left child is least
                //左节点元素
                Object c = array[child];
                //右节点
                int right = child + 1;
                //左右比较，取最小
                if (right < n &&
                    ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                    c = array[child = right];
                //如果待调整key最小，那就退出，直接赋值
                if (key.compareTo((T) c) <= 0)
                    break;
                //如果key不是最小，那就取左右节点小的那个放到调整位置，然后小的那个节点位置开始再继续调整
                array[k] = c;
                k = child;
            }
            array[k] = key;
        }
    }

    //采用自定义比较器
    private static <T> void siftDownUsingComparator(int k, T x, Object[] array,
                                                    int n,
                                                    Comparator<? super T> cmp) {
        if (n > 0) {
            int half = n >>> 1;
            while (k < half) {
                int child = (k << 1) + 1;
                Object c = array[child];
                int right = child + 1;
                if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                    c = array[child = right];
                if (cmp.compare(x, (T) c) <= 0)
                    break;
                array[k] = c;
                k = child;
            }
            array[k] = x;
        }
    }
```
#### remove(Object o)
有2个调整，先自顶向下调整，保证最小，然后再向上调整。
```
    public boolean remove(Object o) {
        //获得锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取删除元素下标
            int i = indexOf(o);
            if (i == -1)
                return false;
            //移除元素
            removeAt(i);
            return true;
        } finally {
            lock.unlock();
        }
    }
    //获得对象下标
    private int indexOf(Object o) {
        if (o != null) {
            Object[] array = queue;
            int n = size;
            for (int i = 0; i < n; i++)
                if (o.equals(array[i]))
                    return i;
        }
        return -1;
    }
    //删除指定下标元素
    private void removeAt(int i) {
        Object[] array = queue;
        int n = size - 1;
        //如果是尾节点，直接移除
        if (n == i) // removed last element
            array[i] = null;
        else {
            //获取尾节点
            E moved = (E) array[n];
            array[n] = null;
            Comparator<? super E> cmp = comparator;
            //指定位置下调
            if (cmp == null)
                siftDownComparable(i, moved, array, n);
            else
                siftDownUsingComparator(i, moved, array, n, cmp);
            //上调
            if (array[i] == moved) {
                if (cmp == null)
                    siftUpComparable(i, moved, array);
                else
                    siftUpUsingComparator(i, moved, array, cmp);
            }
        }
        size = n;
    }
```

### 获取
#### peek()
获取第一个元素，不移除。
```
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (size == 0) ? null : (E) queue[0];
        } finally {
            lock.unlock();
        }
    }
```
# FAQ
1. 为什么只有一个notEmpty的Condition？
在于PriorityBlockingQueue是一个无界队列，插入总是会成功，除非消耗尽了资源导致服务器挂。

2. 为什么扩容的时候，先释放锁。
了提高性能，使用 CAS 控制只有一个线程可以进行扩容，并且在扩容前释放了锁，让其它线程可以进行入队出队操作。

---
*参考资料：*
1. https://www.cnblogs.com/wangzhongqiu/p/8522485.html
2. http://gitbook.cn/books/5ad09cdf7e92727030f21eec/index.html
3. https://blog.csdn.net/chenssy/article/details/76409396
