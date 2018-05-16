# 原理介绍
Exchanger提供了一个同步点，在这个同步点，两个线程可以交换数据。

当线程A调用Exchanger对象的exchange()方法后，他会陷入阻塞状态，直到线程B也调用了exchange()方法，然后以线程安全的方式交换数据，之后线程A和B继续运行。

# 源码分析
![类图](https://github.com/alanzhang211/learning-note/blob/master/img/(https://github.com/alanzhang211/learning-note/blob/master/img/slotExchanger.jpg))

## 核心变量
```
//每个线程保留唯一的一个Node节点
private final Participant participant;
//arena为数组槽
private volatile Node[] arena;
//单个槽
private volatile Node slot;
```
## 内部类
#### Node

```
@sun.misc.Contended static final class Node {
        int index;              // arena的下标
        int bound;              // 上一次记录的Exchanger.bound
        int collides;           // 在当前bound下CAS失败的次数
        int hash;               // 伪随机数，用于自旋
        Object item;            // 线程的当前项，也就是需要交换的数据
        volatile Object match;  // releasing操作的线程传递的项
        volatile Thread parked; // 挂起时设置线程值，其他情况下为null
    }
```

#### Participant
为每个线程保留唯一的一个Node节点，它继承ThreadLocal，同时在Node节点中记录在arena中的下标index。

```
static final class Participant extends ThreadLocal<Node> {
        public Node initialValue() { return new Node(); }
    }
```
## 核心方法
### exchange(V x)
等待其他线程到达交换点，然后与其进行数据交换，支持中断。

```
public V exchange(V x) throws InterruptedException {
        Object v;
        Object item = (x == null) ? NULL_ITEM : x; // translate null args
        //arena为数组槽，如果为null，则执行slotExchange()方法，否则判断线程是否中断;没有中断则执行arenaExchange()方法.
        if ((arena != null ||
             (v = slotExchange(item, false, 0L)) == null) &&
            ((Thread.interrupted() || // disambiguates null return
              (v = arenaExchange(item, false, 0L)) == null)))
            throw new InterruptedException();
        return (v == NULL_ITEM) ? null : (V)v;
    }
```
### slotExchange(Object item, boolean timed, long ns)

```
private final Object slotExchange(Object item, boolean timed, long ns) {
        //获取当前线程的节点
        Node p = participant.get();
        Thread t = Thread.currentThread();
        //如果当前线程被中断，直接返回
        if (t.isInterrupted()) // preserve interrupt status so caller can recheck
            return null;
        //自旋
        for (Node q;;) {
            //slot不为空，表示有数据进行交换
            if ((q = slot) != null) {
                //CAS替换
                if (U.compareAndSwapObject(this, SLOT, q, null)) {
                    Object v = q.item;//需要交换的数据
                    q.match = item;//做releasing操作的线程传递的项
                    Thread w = q.parked;//挂起时设置线程值
                    //挂起线程不为null
                    if (w != null)
                        U.unpark(w);//挂起线程
                    return v;//返回交换数据
                }
                //如果失败了，则创建arena
                //bound 则是上次Exchanger.bound
                if (NCPU > 1 && bound == 0 &&
                    U.compareAndSwapInt(this, BOUND, 0, SEQ))
                    arena = new Node[(FULL + 2) << ASHIFT];
            }
            //如果arena != null，说明有线程竞争slot，直接返回，进入arenaExchange逻辑处理
            else if (arena != null)
                return null; // caller must reroute to arenaExchange
            else {
                //第一次循环,给pnode的item 赋值
                p.item = item;
                //将slot赋值赋值为 p
                if (U.compareAndSwapObject(this, SLOT, null, p))
                //赋值成功跳出循环
                    break;
                //如果CAS失败,将p的值清空,重来
                p.item = null;
            }
        }

        //走到这里的时候,说明 slot 是 null, 且arena不是 null(没有多线程竞争使用 slot),并且成功将 item 放入了slot中。
        //这个时候要做的就是阻塞自己,等待对方取出 slot 的数据项,然后重置slot的数据和池化对象的数据。
        //伪随机数
        int h = p.hash;
        //超时时间
        long end = timed ? System.nanoTime() + ns : 0L;
        //自旋次数，默认1024
        int spins = (NCPU > 1) ? SPINS : 1;
        Object v;
        //如果这个值不是 null, 说明数据被其他线程拿走了, 并且其他线程将数据赋值给match属性,完成了一次交换。
        while ((v = p.match) == null) {
            //自旋
            if (spins > 0) {
                //计算伪随机数
                h ^= h << 1; h ^= h >>> 3; h ^= h << 10;
                //如果算出来的是0,就使用线程id
                if (h == 0)
                    h = SPINS | (int)t.getId();
                //如果小于0,就将自旋数减一
                else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0)
                    //让出 CPU
                    Thread.yield();
            }
            //如果自旋数不够了,且 slot 还没有得到,就重置自旋数。
            else if (slot != p)
                spins = SPINS;
            //slot == p 了,说明对 slot 赋值成功
            //判断，如果没有中断并且数据arena为null（无多线程竞争），并且没有超时
            else if (!t.isInterrupted() && arena == null &&
                     (!timed || (ns = end - System.nanoTime()) > 0L)) {
                //为线程中的 parkBlocker 属性赋值为Exchange自己。
                U.putObject(t, BLOCKER, this);
                //node 节点的阻塞线程为当前线程。
                p.parked = t;
                //如果这个数据还没有被拿走,阻塞自己
                if (slot == p)
                    U.park(false, ns);
                //线程苏醒后,将p的阻塞线程属性清空
                p.parked = null;
                U.putObject(t, BLOCKER, null);
            }
            //如果有超时限制,使用CAS将slot从p变成null,取消这次交换
            else if (U.compareAndSwapObject(this, SLOT, p, null)) {
                //如果CAS成功，并且时间到了并且线程没有中断
                v = timed && ns <= 0L && !t.isInterrupted() ? TIMED_OUT : null;
                //跳出内层循环
                break;
            }
        }
        //将p的match属性设置成null,表示初始化状态,没有任何匹配。
        U.putOrderedObject(p, MATCH, null);
        //重置item
        p.item = null;
        //保留伪随机数,供下次种子数字
        p.hash = h;
        //返回交换数据
        return v;
    }
```
### 算法逻辑
![算法逻辑](https://github.com/alanzhang211/learning-note/blob/master/img/slotExchanger.jpg)

> 1. Exchange使用了对象池的技术,将对象保存在 ThreadLocal 中,这个对象(Node)封装了数据项,线程对象等关键数据。
> 2. 当第一个线程进入的时候,会将数据放到池化对象中,并赋值给slot的item.并阻塞自己(通常不会立即阻塞,而是使用yield自旋一会儿),等待对方取值。
> 3. 当第二个线程进入的时候,会拿出存储在slot item 中的值,然后对slot的match赋值,并唤醒上次阻塞的线程。
> 4. 当第一个线程阻塞被唤醒后,说明对方取到值了,就获取slot的match值,并重置slot的数据和池化对象的数据,并返回自己的数据。
> 5.如果超时了,就返回 Time_out 对象。
> 6. 如果线程中断了,就返回 null。

在该方法中,会返回2种结果,一是有效的item, 二是null:要么是线程竞争使用slot了,创建了arena数组,要么是线程中断了。

当slot被别是线程使用了，那么就需要创建一个arena的数组了。通过操纵数组里面的元素来实现数据交换。

# 应用场景
1. Exchanger可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。
2. Exchanger也可以用于校对工作。
3. 生产者消费者

# 实战分析

```
public class ExchangerTest {
    static class Producer implements Runnable{
        //生产者、消费者交换的数据结构
        private List<String> buffer;
        //步生产者和消费者的交换对象
        private Exchanger<List<String>> exchanger;
        Producer(List<String> buffer,Exchanger<List<String>> exchanger){
            this.buffer = buffer;
            this.exchanger = exchanger;
        }
        @Override
        public void run() {
            for(int i = 1 ; i < 5 ; i++){
                System.out.println("生产者第" + i + "次提供");
                for(int j = 1 ; j <= 3 ; j++){
                    System.out.println("生产者装入" + i  + "--" + j);
                    buffer.add("buffer：" + i + "--" + j);
                }
                System.out.println("生产者装满，等待与消费者交换...");
                try {
                    exchanger.exchange(buffer);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    static class Consumer implements Runnable {
        private List<String> buffer;
        private final Exchanger<List<String>> exchanger;
        public Consumer(List<String> buffer, Exchanger<List<String>> exchanger) {
            this.buffer = buffer;
            this.exchanger = exchanger;
        }
        @Override
        public void run() {
            for (int i = 1; i < 5; i++) {
                //调用exchange()与消费者进行数据交换
                try {
                    buffer = exchanger.exchange(buffer);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("消费者第" + i + "次提取");
                for (int j = 1; j <= 3 ; j++) {
                    System.out.println("消费者 : " + buffer.get(0));
                    buffer.remove(0);
                }
            }
        }
    }

    public static void main(String[] args){
        List<String> buffer1 = new ArrayList<String>();
        List<String> buffer2 = new ArrayList<String>();

        Exchanger<List<String>> exchanger = new Exchanger<List<String>>();

        Thread producerThread = new Thread(new Producer(buffer1,exchanger));
        Thread consumerThread = new Thread(new Consumer(buffer2,exchanger));

        producerThread.start();
        consumerThread.start();
    }
}
```
# FAQ
1. slot作为Exchanger交换数据的场景，应该只需要一个就可以了啊？为何还多了一个Participant 和数组类型的arena呢？

一个slot交换场所原则上来说应该是可以的，但实际情况却不是如此，多个参与者使用同一个交换场所时，会存。在严重伸缩性问题。既然单个交换场所存在问题，那么我们就安排多个，也就是数组arena。通过数组arena来安排不同的线程使用不同的slot来降低竞争问题，并且可以保证最终一定会成对交换数据。但是Exchanger不是一来就会生成arena数组来降低竞争，只有当产生竞争是才会生成arena数组。

2. 如何来判断会有竞争？
CAS替换slot失败，如果失败，则通过记录冲突次数来扩展arena的尺寸，我们在记录冲突的过程中会跟踪“bound”的值，以及会重新计算冲突次数在bound的值被改变时。

3. Exchanger和SynchronousQueue交换数据的区别？

Exchanger 使用了一个对象里的两个属性，item和match而多线程并发时，使用数组来控制，每个线程访问数组中不同的槽

SynchronousQueue 使用的是同一个属性，通过不同的isData来区分，多线程并发时，使用了队列进行排队。

SynchronousQueue交换一份数据，Exchanger交换两份。

---
*参考资料：*
1. 《java并发编程的艺术》
2. https://letcheng.github.io/2016/05/26/java-exchanger.html
3. http://brokendreams.iteye.com/blog/2253956
4. https://juejin.im/post/5ae7554ff265da0b86360880
