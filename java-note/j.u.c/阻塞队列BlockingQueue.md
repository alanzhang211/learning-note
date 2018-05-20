# 原理介绍
阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。

## 两钟阻塞操作
 1. 支持阻塞的插入方式：队列会阻塞插入元素的线程，直到队列不满。
 2. 支持阻塞的移除方式：是在队列为空时，获取元素的线程会等待队列变为非空。

## 分类
![分类](https://github.com/alanzhang211/learning-note/raw/master/img/blockingqueue-all.jpg)

> [脑图源文件](https://github.com/alanzhang211/learning-note/blob/master/img/%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.xmind)

+ ArrayBlockingQueue：此队列按照先进先出（FIFO）的原则对元素进行排序。默认是非公平访问队列。
+ LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列。此队列的默认和最大长度为Integer.MAX_VALUE。此队列按照先进先出的原则对元素进行排序。
+ PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。默认情况下元素采取自然顺序升序排列。也可以自定义类实现compareTo()方法来指定元素排序规则，或者初始化PriorityBlockingQueue时，指定构造参数Comparator来对元素进行排。注意的是不能保证同优先级元素的顺序。
+ DelayQueue：一个使用优先级队列实现的无界阻塞队列。在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。
+ SynchronousQueue：一个不存储元素的阻塞队列。它支持公平访问队列。默认情况下线程采用非公平性策略访问队列。
+ LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。相对于其他阻塞队列，LinkedTransferQueue多了tryTransfer和transfer方法。
+ LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。在多线程同时入队时，也就减少了一半的竞争。

## 通知模式实现

所谓通知模式，就是当生产者往满的队列里添加元素时会阻塞住生产者，当消费者消费了一个队列中的元素后，会通知生产者当前队列可用。如下图示：生产线程Thread1，消费线程Thread2。

![生产-消费线程](https://github.com/alanzhang211/learning-note/raw/master/img/BlockingQueue.jpg)


# 源码分析

![类图](https://github.com/alanzhang211/learning-note/raw/master/img/BlockingQueue-class.jpg)

## 核心方法

功能|抛出异常|特殊值|阻塞|超时
---|---|---|---|---
插入|add(e)|offer(e) |put(e)|offer(e,time,unit)
移除|remove| pool |take|poll(time,unit)
检测|element()|peek()|-|-

1. 抛出异：如果操作不能马上进行，则抛出异常。
2. 特殊值：如果操作不能马上进行，将会返回一个特殊的值，一般是true或者false。
3. 阻塞:如果操作不能马上进行，操作会被阻塞。
4. 超时:如果操作不能马上进行，操作会被阻塞指定的时间，如果指定时间没执行，则返回一个特殊值，一般是true或者false。


# 应用场景
生产者和消费者场景。

---
*参考资料：*
1. 《java并发编程的艺术》
2. https://blog.csdn.net/suifeng3051/article/details/48807423
