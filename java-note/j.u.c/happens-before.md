# 原理介绍
> happens-before作为JMM内存语义的基础，它是判断数据是否存在竞争、线程是否安全的主要依据，依靠这个原则，我们解决在并发环境下两操作之间是否可能存在冲突的所有问题。

## 定义
1. 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
2. 两个操作之间存在happens-before关系，并不意味着一定要按照happens-before原则制定的顺序来执行。如果重排序之后的执行结果与按照happens-before关系来执行的结果一致，那么这种重排序并不非法。

## happens-before与JMM的关系
![happens-before与JMM的关系](https://github.com/alanzhang211/learning-note/raw/master/img/happens-before-JMM.png)

## 原则
1. 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作。
2. 锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作。
3. volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作。
4. 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C。
5. 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作。
6. 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生。
7. 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行。
8. 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始。

> 从原生的原则，推演出如此下规则：
> 1. 将一个元素放入一个线程安全的队列的操作Happens-Before从队列中取出这个元素的操作。
> 2. 将一个元素放入一个线程安全容器的操作Happens-Before从容器中取出这个元素的操作。
> 3. 在CountDownLatch上的倒数操作Happens-Before CountDownLatch#await()操作。
> 4. 释放Semaphore许可的操作Happens-Before获得许可操作。
> 5. Future表示的任务的所有操作Happens-Before Future#get()操作。
> 6. 向Executor提交一个Runnable或Callable的操作Happens-Before任务开始执行操作。

## 应用场景
happens-before作为同步、竞争的判定依据。

## 实战分析
## 安全的单例模式
### 双重检锁（DCL）
> 双重检锁实现单例模式是存在问题的。使用happens-before原则进行分析。

```
class Foo {
    private Helper helper;
    public Helper getHelper() {
        if (helper == null) {
            synchronized(this) {
                if (helper == null) {
                    helper = new Helper();//问题根源
                }
            }
        }
        return helper;
    }

    // other functions and members...
}
```
#### 分析
*helper = new Helper();* 这里是创建初始化对象的过程。在jvm内存中会被分解为3个步骤：

```
memory = allocate();//1.分配内存
ctorInstance(memeory);//2.初始化对象
helper = memory;//3.设置helper指向刚分配的内存地址
```
上述2、3之间，可能会出现重排序。可能情况：
```
memory = allocate();//1.分配内存
helper = memory;//3.设置helper指向刚分配的内存地址
                //这里对象还没有初始化
ctorInstance(memeory);//2.初始化对象
```
### 解决方式
1. 不允许2、3重排。
> 使用volatile的内存语义：**volatile具有禁止重排序的语义;对一个volatile变量的写操作Happens-Before后面对这个变量的读操作，这里的“后面”指的是时间上的先后顺序。**

```
class Foo {
    private volatile Helper helper;
    public Helper getHelper() {
        if (helper == null) {
            synchronized(this) {
                if (helper == null) {
                    helper = new Helper();
                }
            }
        }
        return localRef;
    }

    // other functions and members...
}
```

另一种写法：

```
class Foo {
    private volatile Helper helper;
    public Helper getHelper() {
        Helper localRef = helper;
        if (localRef == null) {
            synchronized(this) {
                localRef = helper;
                if (localRef == null) {
                    helper = localRef = new Helper();
                }
            }
        }
        return localRef;
    }
    // other functions and members
}
```
> wiki里说这样有25%的性能提升（在这里，没有跑数据，依据《effective java》第71条：慎用延迟加载中说到）

![](https://github.com/alanzhang211/learning-note/raw/master/img/effective-Java-dcl.png)

2. 允许2、3重排，但是不允许其他线程看到重排，
> 基于类初始化解决方案（定义一个静态内部类）：jvm在类初始化阶段，会执行类的初始化，在此期间，jvm会获取锁，这个锁可以同步多线程对同一个类的初始化。
```
// Correct lazy initialization in Java
class Foo {
    private static class HelperHolder {
       public static final Helper helper = new Helper();
    }

    public static Helper getHelper() {
        return HelperHolder.helper;//出发类的初始化，jvm在类初始化时加入锁。
    }
}
```
> 以下条件，一个类或接口类型的T将被初始化
+ T是一个类，而且一个T类型的实例被创建。
+ T是一个类，且T中声明的一个静态方法被调用。
+ T中声明的一个静态字段被赋值。
+ T中声明的一个静态字段被使用，且这个字段不是一个常量。


### 对比
+ 基于类初始化方案，代码更加简洁。
+ 基于volatile的方案，除了对静态字段实施延迟初始化外，还可以对实例字段实现延迟初始化。
 > 字段延迟初始化降低了初始化类或创建实例的开销，但增加了访问被延迟初始化字段的开销。
> 如果确实要对实例字段使用线程安全的初始化，使用基于volatile的方案；如果对静态字段使用线程安全的初始化，请使用基于类初始化的方案。

---
*参考资料：*
+ https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java
+ http://www.importnew.com/23519.html
+ 《java并发编程的艺术》
