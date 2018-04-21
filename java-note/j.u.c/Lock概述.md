# 介绍
Lock是JDK1.5 之后新增的用来实现锁功能。同java内置锁synchronized一样，实现锁的同能。

## 结构
![类图](https://github.com/alanzhang211/learning-note/raw/master/img/Lock.png)

主要核心类，见上图。

### 3个接口
#### Lock
Lock接口，提供了获取过和释放锁的方法。

```
public interface Lock {
    void lock();//用来获取锁。如果锁已被其他线程获取，则进行等待（try之前获得锁）。
    void lockInterruptibly() throws InterruptedException;  // 可以响应中断
    boolean tryLock();//尝试获取锁，如果获取成功，则返回true；如果获取失败（即锁已被其他线程获取），则返回false，也就是说，这个方法无论如何都会立即返回（在拿不到锁时不会一直在那等待）。
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;  // 类似于tryLock(),拿不到锁等待一段时间。可以响应中断
    void unlock();//释放锁（finally释放锁）。
    Condition newCondition();//返回绑定到Lock的新的Condition实例，用于线程间的通信
}
```
相比较synchronized提供了更加灵活的方式。

> 1.一个线程获取锁时，可以设置一个等待时间，不会让其他线程无限期的等待下去。
>
> 2.取得前线程是否获得锁的状态，tryLock() 。
>
> 3.在读写操作，读线程之间不需要进行同步操作，以提高性能 。
>
> 4.Lock可以让等待锁的线程响应中断（在获取不到锁时终止等待状态）。



#### Condition
```
public interface Condition {
    void await() throws InterruptedException;//当前线程进入等待队列知道signal或中断。
    void awaitUninterruptibly;//当前线程进入等待状态直到被通知，对中断不敏感
    long awaitNanos(long nanosTimeout) throws InterruptedException;//当前线程进入等待状态，知道被通知、中断或超时。返回值是剩余时间
    boolean await(long time, TimeUnit unit) throws InterruptedException;//当前线程进入等待状态，知道被通知、终止或超时。
    boolean awaitUntil(Date deadline) throws InterruptedException;//进入等待状态直到被通知、中断或到某个时间。
    void signal();//唤醒一个等待在condition上的线程
    void signalAll();//唤醒所有等待在condition上的线程
}
```
Condition接口提供了类似于Object的监控方法，与Lock配合使用实现等待/通知模式。

Condition依赖Lock对象（通过Lock对象的newCondition方法获的）。

需要在调用方法前获得锁；一般将Condition对象作为成员变量。

#### 实现分析
ConditionObject作为同步器的内部类；每个Condition都包含一个等待队列。是实现等待/通知的关键。

1. 等待队列
一个Condition包含一个等待队列。一个Lock拥有一个同步队列和多个等待队列（调用Lock.newCondition）
插入节点时不需要CAS，await操作必须获取锁，线程安全由锁保护。

2. 等待
调用Condition的await方法，会是当前线程进入等待队列饼释放锁，同时线程变为等待状态。

3. 通知
调用condition的signal方法，将会唤醒在等待队列时间最长的节点（首节点），在唤醒节点之前，会将节点移动到同步队列中。

#### ReadWriteLock

```
public interface ReadWriteLock {
    Lock readLock();//获取读锁
    Lock writeLock();//获取写锁
}
```
接口只定义了readLock和writeLock方法。

ReentrantReadWriteLock实现中接口外，提供了一些便于外界监控器内部的方法。

### 3个抽象类
#### AbstractOwnableSynchronizer
用于表示锁与持有者之间的关系；
AbstractOwnableSynchronizer不管理这种关系；主要是其子类(AbstractQueuedSynchronizer/AbstractQueuedLongSynchronizer)，用于控制或监视锁的状态，提供帮助。
#### AbstractQueuedSynchronizer
同步器，AQS全称，它是构建锁或者其他同步组件的基础框架。内部依赖LockSupport。

AQS的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态。

> 同步器，基于模版模式设计,同步器提供的模版方法：
>
> 1、独占式获取与释放同步状态
>
> 2、共享式获取与释放同步状态
>
> 3、查询同步队列中的等待线程情况

AQS的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态。

AQS通过内置的FIFO同步队列来完成资源获取线程的排队工作，如果当前线程获取同步状态失败（锁）时，AQS则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，则会把节点中的线程唤醒，使其再次尝试获取同步状态。

#### AbstractQueuedLongSynchronizer
AbstractQueuedLongSynchronizer和AbstractQueuedSynchronizer的区别在于acquire和release的arg参数是long而不是int类型。



### 4个具体类
#### LockSupport
当需要阻塞或唤醒一个线程的时候，都会使用LockSupport工具类来完成相应的工作。

定义了一组（以park开头的方法阻塞当前线程；unpark（Thread thread）唤醒一个阻塞线程）公共静态方法，提供了最基本的线程阻塞和唤醒功能。也称为构建同步组件的基本工具类。


#### ReentrantLock
eentrantLock，可重入锁，是一种递归无阻塞的同步机制。它可以等同于synchronized的使用，但是ReentrantLock提供了比synchronized更强大、灵活的锁机制，可以减少死锁发生的概率。

> 1、支持一个线程对资源重读加锁
>
> 2、支持获取锁时的公平和非公平性选择(内部通过Sync内部类实现，Sync继承AQS；公平锁FairSync和非公平锁NonfairSync)

ReentrantLock里面大部分的功能都是委托给Sync来实现的，同时Sync内部定义了lock()抽象方法由其子类去实现，默认实现了nonfairTryAcquire(int acquires)方法，可以看出它是非公平锁的默认实现方式。


#### ReentrantReadWriteLock
一对锁，一个读锁，一个写锁。


> 1. 支持公平和非公平的方式获取锁；吞吐量非公平优于公平。
> 2. 重入性：读锁获取后可以再次获取读锁；写锁获取后，可以再次后去写锁，同时也可以获取读锁。
> 3. 锁降级：支持获取写锁、获取读锁再释放写锁；写锁能够降级为读锁。
> 4. 分离出读锁和写锁，相比其他排他锁有更好的升。
> 5. 减化读写的交互场景和编程方式

#### StampedLock
java8在java.util.concurrent.locks新增的一个API

ReentrantReadWriteLock 在沒有任何读写锁时，才可以取得写入锁，这可用于实现了悲观读取（Pessimistic Reading），即如果执行中进行读取时，经常可能有另一执行要写入的需求，为了保持同步，ReentrantReadWriteLock 的读取锁定就可派上用场。

然而，如果读取执行情况很多，写入很少的情况下，使用 ReentrantReadWriteLock 可能会使写入线程遭遇饥饿（Starvation）问题，也就是写入线程迟迟无法竞争到锁定而一直处于等待状态。

StampedLock控制锁有三种模式（写，读，乐观读），一个StampedLock状态是由版本和模式两个部分组成，锁获取方法返回一个数字作为票据stamp，它用相应的锁状态表示并控制访问，数字0表示没有写锁被授权访问。在读锁上分为悲观锁和乐观锁。

所谓的乐观读模式，也就是若读的操作很多，写的操作很少的情况下，你可以乐观地认为，写入与读取同时发生几率很少，因此不悲观地使用完全的读取锁定，程序可以查看读取资料之后，是否遭到写入执行的变更，再采取后续的措施（重新读取变更信息，或者抛出异常） ，这一个小小改进，可大幅度提高程序的吞吐量。

后续篇章会针对核心几个类进行介绍，敬请期待。

---
*参考资料：*
1. 《java并发编程实战》
2. 《java并发编程的艺术》
3. http://www.importnew.com/14941.html
