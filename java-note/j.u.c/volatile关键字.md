# 原理介绍
 volatile 变量可以被看作是一种 “程度较轻的 synchronized”，某种程度上可以实现同步的效果。这体现在“可见性”上。

 > 锁提供了两种主要特性：互斥（mutual exclusion） 和可见性（visibility）。

 ## 语义
 在JMM中，线程之间的通信采用共享内存来实现的.

 > 当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值立即刷新到主内存中。
当读一个volatile变量时，JMM会把该线程对应的本地内存设置为无效，直接从主内存中读取共享变量。

## 特性
1. 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。

2. 禁止进行指令重排序。
实现指令重排，原理是使用内存屏障。

3. 原子性
这里的原子性体现在单个volatile的读/写上，类似于volatile++（包括：读取，修改，保存操作）的复合操作是不具备原子性的。


## volatile内存屏障
+ 在每一个volatile写操作前面插入一个StoreStore屏障。
+ 在每一个volatile写操作后面插入一个StoreLoad屏障。
+ 在每一个volatile读操作后面插入一个LoadLoad屏障。
+ 在每一个volatile读操作后面插入一个LoadStore屏障。

### volatile写
![volatile写](https://github.com/alanzhang211/learning-note/raw/master/img/volatile-%E5%86%85%E5%AD%98%E5%86%99%E5%B1%8F%E9%9A%9C.png)

### volatile读
![volatile读](https://github.com/alanzhang211/learning-note/raw/master/img/volatile-neicundu.png)

## 正确使用volatile
### 条件
+ 对变量的写操作不依赖于当前值。
+ 该变量没有包含在具有其他变量的不变式中。

### 原因
+ 在某些情形下，使用 volatile 变量要比使用相应的锁简单得多。
+ 某些情况下，volatile 变量同步机制的性能要优于锁。

> volatile 操作不会像锁一样造成阻塞，因此，在能够安全使用 volatile 的情况下，volatile 可以提供一些优于锁的可伸缩特性。如果读操作的次数要远远超过写操作，与锁相比，volatile 变量通常能够减少同步的性能开销。

# 应用场景
## 轻量级的“读-写锁”策略。
> 结合使用 volatile 和 synchronized 实现 “开销较低的读－写锁”。

```
@ThreadSafe
public class CheesyCounter {
    // Employs the cheap read-write lock trick
    // All mutative operations MUST be done with the 'this' lock held
    @GuardedBy("this") private volatile int value;

    public int getValue() { return value; }

    //todo 这里可以使用原子变量实现synchronized原子性。如：AtomicInteger的incrementAndGet方法。
    public synchronized int increment() {
        return value++;
    }
}
```

## 状态标志
> 也许实现 volatile 变量的规范使用仅仅是使用一个布尔状态标志，用于指示发生了一个重要的一次性事件，例如完成初始化或请求停机。


```
volatile boolean shutdownRequested;

...

public void shutdown() { shutdownRequested = true; }

public void doWork() {
    while (!shutdownRequested) {
        // do stuff
    }
}
```

## 一次性安全发布

> 实现安全发布对象的一种技术就是将对象引用定义为 volatile 类型。**避免了“双重检锁”的问题：某个对象引用的更新值（由另一个线程写入）和该对象状态的旧值同时存在。**

```
public class BackgroundFloobleLoader {
    public volatile Flooble theFlooble;

    public void initInBackground() {
        // do lots of stuff
        theFlooble = new Flooble();  // this is the only write to theFlooble
    }
}

public class SomeOtherClass {
    public void doWork() {
        while (true) {
            // do some stuff...
            // use the Flooble, but only if it is ready
            if (floobleLoader.theFlooble != null)
                doSomething(floobleLoader.theFlooble);
        }
    }
}
```
> theFlooble = new Flooble();  语句编译成字节码指令一般是含有这两条指令：new，invokespecial（new指令顺序先于invokespecial指令），其中new用来分配对象内存空间并初始化默认值并返回堆对象的引用，而invokespecial指令用来调用对象自定义初始化方法<init>。


**分解指令**
1. memory = allocate(); // 分配内存空间 是new指令的一部分
2. <init>(memory); // 初始化对象 对应invokespecial调用对象自定义初始化方法
3. instance = memory ; // 返回分配的内存地址引用 是new指令的一部分

## 独立观察
> 定期 “发布” 观察结果供程序内部使用;如：应用程序就是收集程序的统计信息。

```
public class UserManager {
    public volatile String lastUser;

    public boolean authenticate(String user, String password) {
        boolean valid = passwordIsValid(user, password);
        if (valid) {
            User u = new User();
            activeUsers.add(u);
            lastUser = user;
        }
        return valid;
    }
}
```

## 防止对象逸出
> volatile bean 模式适用于将 JavaBeans 作为“荣誉结构”使用的框架。在 volatile bean 模式中，JavaBean 被用作一组具有 getter 和/或 setter 方法 的独立属性的容器。volatile bean 模式的基本原理是：很多框架为易变数据的持有者（例如 HttpSession）提供了容器，但是放入这些容器中的对象必须是线程安全的。

```
@ThreadSafe
public class Person {
    private volatile String firstName;
    private volatile String lastName;
    private volatile int age;

    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public int getAge() { return age; }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

# 实战分析
> 问题：调度系统中，如果系统停止，则任务队列中的任务也进行终止。

这里就使用了上面描述的场景“状态标志”实现。

## 代码概要
```
//停止调度的标志
private volatile boolean stop = false;
...

//任务线程run方法检测stop表识
@Override
public void run() {
        init();
        while(true){
            if(stop){
                ....//任务执行逻辑
            }

```

# FAQ
待续...

---
参考资料：
+ 《java并发编程的艺术》
+ [正确使用 Volatile 变量](https://www.ibm.com/developerworks/cn/java/j-jtp06197.html)
+ [Java内存模型之分析volatile](http://cmsblogs.com/?p=2148)
+ [volatile的实现原理以及应用场景](https://www.jianshu.com/p/d577c2817af8)
