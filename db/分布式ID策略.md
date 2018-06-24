# 前言
继续讨论下一个分库分表的问题：**分布式ID问题**。这里的ID不一定是指数据库主键id，而是泛指指业务系统中能够识别唯一一条信息的标识。本文就以数据库主键为背景进行介绍。

那么，问题来了。分库后，不能依赖单个数据库主键唯一性约束来保证记录的唯一。

# 解决方案
在分布式的条件下，据需要一个“ID发号机”来实现。这个发号机可以用一下方式实现。

## 数据库实现
简单的思路，使用数据库的`auto_increment`生成全局唯一的。每个实例使用`auto_increment_increment`等长的步长，然后`auto_increment_offset`设置不同的初始值。

**优点**
1. 数据库实现，无需引入其他组件。

**缺点**
1. 强依赖于数据库，每次业务系统取号，都要访问数据库，增加了db的压力。
2. id不能保证绝对的递增。

## 使用Redis
为了解决数据库性能的压力，可以使用Redis来处理。使用Redis的`incr` 和 `incrby`来实现。同样每个实例设置不同的初始值，然后，设置相同的步长。

**优点**
1. 不依赖数据库，性能优于数据库。

**缺点**
1. 需要引入Redis，增加系统复杂度。
2. 增加了代码维护成本。

## 使用UUID
另一种简单的实现，就是用全球唯一的UUID进行标识。

 > UUID(Universally Unique Identifier)的标准型式包含32个16进制数字，以连字号分为五段，形式为8-4-4-4-12的36个字符。
 
 > UUID由以下几部分的组合：
（1）当前日期和时间，UUID的第一个部分与时间有关，如果你在生成一个UUID之后，过几秒又生成一个UUID，则第一个部分不同，其余相同。
（2）时钟序列。
（3）全局唯一的IEEE机器识别号，如果有网卡，从网卡MAC地址获得，没有网卡以其他方式获得。
 
 如：java中使用类UUID获取。
 ```
 String uuid = UUID.randomUUID().toString();
 ```
 **优点**
 1. 性能非常高：本地生成，没有网络消耗。
 
**缺点**
1. 不易于存储：UUID太长，16字节128位，通常以36长度的字符串表示。
2. 不利于作为主键：uuid的无序性，使得主键索引变更频繁。
3.不安全：包含了mac地址信息。

## snowflake方案
是一种以划分命名空间来划分的算法。这种方案把64-bit分别划分成多段，分开来标示机器、时间等，比如在snowflake中的64-bit分别表示如下图：
![snowflake](https://github.com/alanzhang211/learning-note/raw/master/img/db/snowflakepng)

> 上面提到的UUID也可以认为是这种思路。

> Mongdb objectID也是类似的算法。
![Mongdb objectID](https://github.com/alanzhang211/learning-note/raw/master/img/db/Mongdb-objectID.png)

**优点**
1. 毫秒数在高位，自增序列在低位，整个ID都是趋势递增的。
2. 不依赖数据库等第三方系统，以服务的方式部署，稳定性更高，生成ID的性能也是非常高的。
3. 以根据自身业务特性分配bit位，非常灵活。

**缺点**
1. 强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务会处于不可用状态。

## 利用zookeeper生成唯一ID
zookeeper主要通过其znode数据版本来生成序列号，可以生成32位和64位的数据版本号，客户端可以使用这个版本号来作为唯一的序列号。

**优点**
1. 分布式下的一般都会使用zookeeper，如果使用可以直接依赖zookeeper实现。无需引入其他组件。

**缺点**
1. 依赖zookeeper，并且是多步调用API，如果在竞争较大的情况下，需要考虑使用分布式锁。

## 其他第三方开源
1. Vesta：https://github.com/cloudatee/vesta-id-generator
2. 美团的Leaf

> 其中，Vesta开源地址：https://github.com/cloudatee/vesta-id-generator有兴趣的可以看看。

---
参考：

 [Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/MT_Leaf.html)
 
 [细聊分布式ID生成方法](https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=403837240&idx=1&sn=ae9f2bf0cc5b0f68f9a2213485313127&scene=21#wechat_redirect)