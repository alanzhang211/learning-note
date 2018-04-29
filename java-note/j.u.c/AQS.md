# 原理介绍
AQS是AbstractQueuedSynchronizer的缩写，AQS是Java并包里大部分同步器的基础构件，利用AQS可以很方便的创建锁和同步器。它封装了一个状态，提供了一系列的获取和释放操作，这些获取和释放操作都是基于状态的。

![类图](https://github.com/alanzhang211/learning-note/raw/master/img/AbstractQueuedSynchronizer.png)

### CLHLock
与标准CLHLock实现不同的是，AQS不是一个自旋锁，它提供了更加丰富的语意：

1. 提供了独享(exclusive)方式和共享(share)方式来获取/释放，比如锁是独占方式的，信号量semaphore是共享方式的，可以有多个线程进入临界区

2. 支持可中断和不可中断的获取/释放

3. 支持普通的和具有时间限制的获取/释放

4. 提供了自旋和阻塞的切换，可以先自旋，如果等待时间长，可以阻塞。


# 源码分析
### 一个状态state
AQS 中维持了一个单一的状态信息state,可以通过getState,setState,compareAndSetState 函数修改其值。

```
protected final int getState() {
        return state;
    }

    protected final void setState(int newState) {
        state = newState;
    }

    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

1. 对于ReentrantLock的实现来说，state可以用来表示当前线程获取锁的可重入次数。

2. 对应读写锁ReentrantReadWriteLock来说state的高16位表示读状态也就是获取该读锁的次数；低16位表示获取到写锁的线程的可重入次数。

3. 对于semaphore来说state用来表示当前可用信号的个数。

4. 对于FutuerTask来说，state用来表示任务状态（例如还没开始，运行，完成，取消）。

5. 对应CountDownlatch和CyclicBarrie来说state用来表示计数器当前的值。

## 两个类
### Node
```
static final class Node {
        /**
         * 线程已被取消
         */
        static final int CANCELLED = 1;
        /**
         * 当前线程的后继线程需要被unpark(唤醒)
         */
        static final int SIGNAL = -1;
        /**
         * 线程(处在Condition休眠状态)在等待Condition唤醒
         */
        static final int CONDITION = -2;
        /**
         * 共享锁
         */
        static final Node SHARED = new Node();
        /**
         * 独占锁
         */
        static final Node EXCLUSIVE = null;
        /**
         * 等待状态
         */
        volatile int waitStatus;
        /**
         * 前继节点
         */
        volatile Node prev;
        /**
         * 后继节点
         */
        volatile Node next;
        /**
         * 获取同步状态的线程
         */
        volatile Thread thread;
        /**
         * 等待队列中的后继节点
         */
        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * 获取前继节点
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        /**
         * 三个构造函数
         */
        Node() {
        }

        Node(Thread thread, Node mode) {
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) {
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

Node定义了队列中的节点。

1. 维护了一个Node SHARED引用表示共享模式

2. 维护了一个Node EXCLUSIVE引用表示独占模式

3. 维护了几种节点等待的状态 waitStatus, 其中CANCELLED = 1是正数，表示取消状态，SIGNAL = -1，CONDITION = -2， PROPAGATE = -3都是负数，表示节点在条件队列的某个状态，SIGNAL表示后续节点需要被唤醒

4. 维护了Node prev引用，指向队列中的前一个节点，通过Tail的CAS操作来创建

5. 维护了Node next引用，指向队列中的下一个节点，也是在通过Tail入队列的时候设置的，这样就维护了一个双向队列

6. 维护了一个volatile的Thread引用，把一个节点关联到一个线程

7. 维护了Node nextWaiter引用，指向在条件队列中的下一个正在等待的节点，是给条件队列使用的。值得注意的是条件队列只有在独享状态下才使用


### ConditionObject
是Condition接口的实现类，负责管理条件队列。

条件队列是通过Node来构件的一个单向链表结构。底层的条件操作(等待和唤醒)使用LockSupport类来实现。

1. 维护了一个Node firstWaiter引用指向条件队列的队首节点

2. 维护了一个Node lastWaiter引用指向条件队列的队尾节点

3. 条件队列支持节点的取消退出机制，CANCELLED节点来表示这种取消状态

4. 支持限时等待机制

5. 支持可中断和不可中断的等待。

### 2个队列

![2个队列](https://github.com/alanzhang211/learning-note/raw/master/img/AQS%E9%98%9F%E5%88%97.png)

#### 同步队列
![同步队列](https://github.com/alanzhang211/learning-note/raw/master/img/%E5%90%8C%E6%AD%A5%E9%98%9F%E5%88%97.png)

AQS 是一个 FIFO 的双向队列，内部通过节点 head 和 tail 记录队首和队尾元素，队列元素类型为 Node。其中 Node 中 thread 变量用来存放进入 AQS 队列里面的线程；Node 节点内部 SHARED 用来标记该线程是获取共享资源时候被阻塞挂起后放入AQS队列，EXCLUSIVE标示线程是获取独占资源时候被挂起后放入AQS队列；waitStatus记录当前线程等待状态，分别为 CANCELLED（线程被取消了），SIGNAL（线程需要被唤醒），CONDITION（线程在条件队列里面等待），PROPAGATE（释放共享资源时候需要通知其它节点）；prev记录当前节点的前驱节点，next 记录当前节点后继节点。

##### 队列是如何维护
###### 入队列
在线程尝试获取锁的时候，如果失败了需要将该线程加入到CLH队列。

```
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }    
```
> 注意在这个过程中有一个CAS操作，采用自旋方式直到成功为止。

###### 出队列
当线程释放锁时，需要进行“出列”，出列的主要工作则是唤醒其后继节点（一般来说就是head节点）
```
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

#### 等待队列/条件队列
通过 lock.newCondition() 作用其实是new了一个 AQS内部声明的 ConditionObject 对象，ConditionObject是AQS的内部类，可以访问到 AQS 内部的变量（例如状态变量 status 变量）和方法。对应每个条件变量内部维护了一个条件队列，用来存放当调用条件变量的await()方法被阻塞的线程。

> 注意的是一个 Lock 对象可以创建多个条件变量.

### 两种模式
在AQS维护的CLH队列锁中，每个节点（Node）代表着一个需要获取锁的线程。该Node中有两个常量SHARE、EXCLUSIVE。其中SHARE代表着共享模式，EXCLUSIVE代表着独占模式。

```
在AQS维护的CLH队列锁中，每个节点（Node）代表着一个需要获取锁的线程。该Node中有两个常量SHARE、EXCLUSIVE。其中SHARE代表着共享模式，EXCLUSIVE代表着独占模式。
```

#### 独占模式的获取与释放资源流程
```
void acquire(int arg)
void acquireInterruptibly(int arg)
boolean release(int arg)
```
1. 当一个线程调用 acquire(int arg) 方法获取独占资源时候，会首先使用 tryAcquire 尝试获取资源，具体是设置状态变量 state 的值，成功则直接返回。失败则将当前线程封装为类型为 Node.EXCLUSIVE 的 Node 节点后插入到 AQS 阻塞队列尾部，并调用 LockSupport.park(this) 挂起当前线程。

```
 public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

2. 当一个线程调用 release(int arg) 时候会尝试使用,tryRelease操作释放资源，这里是设置状态变量state的值，然后调用LockSupport.unpark(thread)激活AQS队列里面最早被阻塞的线程 (thread)。被激活的线程则使用tryAcquire尝试看当前状态变量state的值是否能满足自己的需要，满足则该线程被激活然后继续向下运行，否者还是会被放入AQS队列并被挂起。

```
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
> 注意的 AQS 类并没有提供可用的 tryAcquire 和 tryRelease，正如 AQS 是锁阻塞和同步器的基础框架，tryAcquire 和 tryRelease 需要有具体的子类来实现。子类在实现 tryAcquire 和 tryRelease 时候要根据具体场景使用 CAS 算法尝试修改状态值 state, 成功则返回 true, 否者返回 false。子类还需要定义在调用 acquire 和 release 方法时候 state 状态值的增减代表什么含义。

#### 共享资模式的获取与释放流程
```
void acquireShared(int arg)
void acquireSharedInterruptibly(int arg)
boolean releaseShared(int arg)
```
1. 当线程调用 acquireShared(int arg) 获取共享资源时候，会首先使用tryAcquireShared尝试获取资源，具体是设置状态变量state的值，成功则直接返回。失败则将当前线程封装为类型为 Node.SHARED 的 Node 节点后插入到 AQS 阻塞队列尾部，并使用 LockSupport.park(this) 挂起当前线程。

```
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

2. 当一个线程调用 releaseShared(int arg) 时候会尝试使用, tryReleaseShared 操作释放资源，这里是设置状态变量state的值，然后使用LockSupport.unpark（thread）激活 AQS 队列里面最早被阻塞的线程 (thread)。被激活的线程则使用tryReleaseShared尝试看当前状态变量state的值是否能满足自己的需要，满足则该线程被激活然后继续向下运行，否者还是会被放入 AQS 队列并被挂起。

```
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
> 需要注意的 AQS 类并没有提供可用的 tryAcquireShared和tryReleaseShared，正如AQS是锁阻塞和同步器的基础框架，tryAcquireShared和tryReleaseShared需要有具体的子类来实现。子类在实现 tryAcquireShared 和 tryReleaseShared 时候要根据具体场景使用 CAS 算法尝试修改状态值state,成功则返回true,否者返回false。

---
*参考资料：*
+ 《java并发编程艺术》
+ http://cmsblogs.com/?p=1741
+ http://gitbook.cn/books/5ac8a7e72a04fd6c83713956/index.html
