# 原理介绍
## 特点
synchronized相比volatile，不仅保证了“可见性”，同时也保证了”原子性“。
> 总结：
+ 确保线程互斥的访问同步代码。
+ 保证共享变量的修改能够及时可见。
+ 有效解决重排序问题。

## 使用方式
+ 普通同步方法，锁是当前实例对象
+ 静态同步方法，锁是当前类的class对象
+ 同步方法块，锁是括号里面的对象

> 任何一个对象都一个Monitor与之关联，当且一个Monitor被持有后，它将处于锁定状态。Synchronized在JVM里的实现都是基于进入和退出Monitor对象来实现方法同步和代码块同步，虽然具体实现细节不一样，但是都可以通过成对的MonitorEnter和MonitorExit指令来实现。MonitorEnter指令插入在同步代码块的开始位置，当代码执行到该指令时，将会尝试获取该对象Monitor的所有权，即尝试获得该对象的锁，而monitorExit指令则插入在方法结束处和异常处，JVM保证每个MonitorEnter必须有对应的MonitorExit。

## 代码
```
/**
 *
 * @author alanzhang
 * @date 2018/3/13
 */
public class SynTest {

    /**
     * 锁静态方法(类锁)
     * 1.和所有其他静态方法加锁的进行互斥
     * 2.和synchronized（xx.class)锁效果一样
     */
    public static synchronized void m1() {

    }

    /**
     * 锁普通方法（对象锁）
     *
     */
    public synchronized void m2 () {

    }

    /**
     * 锁代码块（对象锁，锁当前实例对象this）
     */
    public void m3() {
        synchronized (this) {

        }

    }
}
```
## 结论
#### **对象锁**
> 可以防止多个线程同时访问这个对象的synchronized方法（如果一个对象有多个synchronized方法，只要一个线程访问了其中的一个synchronized方法，其它线程不能同时访问这个对象中任何一个synchronized方法）。这时，不同的对象实例的synchronized方法是不相干扰的。也就是说，其它线程照样可以同时访问相同类的另一个对象实例中的synchronized方法。

#### **类锁**
> 防止多个线程中不同的实例对象（或者同一个实例对象）同时访问这个类中的synchronized static 方法。它可以对类的所有对象实例起作用。

#### **synchronized(this)**
> 当两个并发线程访问同一个对象object中的这个synchronized(this)同步代码块时，一个时间内只能有一个线程得到执行。另一个线程必须等待当前线程执行完这个代码块以后才能执行该代码块。

 > 当一个线程访问object的一个synchronized(this)同步代码块时，另一个线程仍然可以访问该object中的非synchronized(this)同步代码块。

 > 当一个线程访问object的一个synchronized(this)同步代码块时，其他线程对object中所有其它synchronized(this)同步代码块的访问将被阻塞。

 > 第三个例子同样适用其它同步代码块。也就是说，当一个线程访问object的一个synchronized(this)同步代码块时，它就获得了这个object的对象锁。结果，其它线程对该object对象所有同步代码部分的访问都被暂时阻塞。

## Java对象头
> synchronized使用的锁是存放在Java对象头里面，具体位置是对象头里面的MarkWord，MarkWord里默认数据是存储对象的HashCode等信息，但是会随着对象的运行改变而发生变化，不同的锁状态对应着不同的记录存储方式，可能值如下所示。

![Java对象头](https://github.com/alanzhang211/learning-note/raw/master/img/java-obj-head.png)

## Monitor
> 什么是Monitor？我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。
与一切皆对象一样，所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，因为在Java的设计中 ，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁。
Monitor 是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联（对象头的MarkWord中的LockWord指向monitor的起始地址），同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。其结构如下：

![Monitor](https://github.com/alanzhang211/learning-note/raw/master/img/monitor.png)

**Owner**：初始时为NULL表示当前没有任何线程拥有该monitor record，当线程成功拥有该锁后保存线程唯一标识，当锁被释放时又设置为NULL；

**EntryQ**:关联一个系统互斥锁（semaphore），阻塞所有试图锁住monitor record失败的线程。

**RcThis**:表示blocked或waiting在该monitor record上的所有线程的个数。

**Nest**:用来实现重入锁的计数。

**HashCode**:保存从对象头拷贝过来的HashCode值（可能还包含GC age）。

**Candidate**:用来避免不必要的阻塞或等待线程唤醒，因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。Candidate只有两种可能的值0表示没有需要唤醒的线程1表示要唤醒一个继任线程来竞争锁。


# 源码分析
## 机制
### 测试代码
见上文：代码

### class文件信息

javap 查看class文件信息

![javap](https://github.com/alanzhang211/learning-note/raw/master/img/javap.png)


### 结论
> **同步代码块**：monitorenter指令插入到同步代码块的开始位置，monitorexit指令插入到同步代码块的结束位置，JVM需要保证每一个monitorenter都有一个monitorexit与之相对应。任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，他将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor所有权，即尝试获取对象的锁。

> **同步方法**：synchronized方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示Klass做为锁对象。

# 应用场景
synchronized形式 |锁|锁类型
---|---|---
synchronized(obj){}| obj| 对象锁
synchronized(A.class){}| A.class| 类锁
synchronized(this){}|当前类对象| 对象锁
synchronized void memberMethod(){} | 当前类对象| 对象锁
static synchronized void staticMethod(){}|当前类| 类锁

# FAQ
*期待你的加入...*

---
参考资料：
+ 《java并发编程的艺术》
+ http://blog.csdn.net/u012465296/article/details/53022317
+ http://cmsblogs.com/?p=2071
