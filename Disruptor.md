# Disruptor

常用的线程安全的内置队列有什么问题。Java的内置队列如下表所示
![-w408](media/15441768993203/15446158697945.jpg)
队列的底层一般分成三种：数组、链表和堆。
- 基于数组线程安全的队列，比较典型的是ArrayBlockingQueue，它主要通过加锁的方式来保证线程安全；
- 基于链表的线程安全队列分成LinkedBlockingQueue和ConcurrentLinkedQueue

通过不加锁的方式实现的队列都是`无界`的，加锁的方式，可以实现有界队列。

## ArrayBlockingQueue的问题

    加锁和伪共享等出现严重的性能问题  

**场景举例：**
1. 被阻塞线程的优先级较高，而持有锁的线程优先级较低，就会发生优先级反转。
2. 加锁通常会严重地影响性能
3. 线程在持有锁的情况下被延迟执行，例如发生了缺页错误、调度延迟或者其它类似情况，会一直持有锁

> 单线程情况下，不加锁的性能 > CAS操作的性能 > 加锁的性能.
> CAS是CPU的一个指令，由CPU保证原子性。

下面是ArrayBlockingQueue通过加锁的方式实现的offer方法，保证线程安全。
```java
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            return false;
        else {
            insert(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```
## LinkedBlockingQueue问题
JVM垃圾回收，和heap内存问题
### 共享
>L1、L2、L3分别表示一级缓存、二级缓存、三级缓存，越靠近CPU的缓存，速度越快.当CPU执行运算的时候，它先去L1查找所需的数据、再去L2、然后是L3，如果最后这些缓存中都没有，所需的数据就要去主内存拿。走得越远，运算耗费的时间就越长。

Cache是由很多个cache line组成的。每个cache line通常是64字节，并且它有效地引用主内存中的一块儿地址。一个Java的long类型变量是8字节，因此在一个缓存行中可以存8个long类型的变量。

CPU每次从主存中拉取数据时，会把相邻的数据也存入同一个cache line。在访问一个long数组的时候，如果数组中的一个值被加载到缓存中，它会自动加载另外7个。因此你能非常快的遍历这个数组。事实上，你可以非常快速的遍历在连续内存块中分配的任意数据结构。

### 什么是伪共享
ArrayBlockingQueue有三个成员变量：
takeIndex：需要被取走的元素下标
putIndex：可被元素插入的位置的下标
count：队列中元素的数量
这三个变量很容易放到一个缓存行中，但是之间修改`没有太多的关联`。所以每次修改，都会使之前缓存的数据失效，从而不能完全达到共享的效果。
![-w728](media/15441768993203/15446167453374.jpg)
这种无法充分使用缓存行特性的现象，称为伪共享。

对于伪共享，一般的解决方案是，增大数组元素的间隔使得由不同线程存取的元素位于不同的缓存行上，以空间换时间。

## Disruptor的设计方案
Disruptor通过以下设计来解决队列速度慢的问题：
- 环形数组结构
    为了避免垃圾回收，采用数组而非链表。同时，数组对处理器的缓存机制更加友好。

- 元素位置定位
    数组长度2^n，通过位运算，加快定位的速度。下标采取递增的形式。不用担心index溢出的问题。index是long类型，即使100万QPS的处理速度，也需要30万年才能用完。

- 无锁设计
    每个生产者或者消费者线程，会先申请可以操作的元素在数组中的位置，申请到之后，直接在该位置写入或者读取数据。

下面忽略数组的环形结构，介绍一下如何实现无锁设计。整个过程通过原子变量CAS，保证操作的线程安全。

多生成者多消费者场景（详见：https://tech.meituan.com/disruptor.html）


### 性能测试
![-w563](media/15441768993203/15446184109177.jpg)
依据并发竞争的激烈程度的不同，Disruptor比ArrayBlockingQueue吞吐量快4~7倍。
## Log4j 2应用场景
Log4j 2相对于Log4j 1最大的优势在于多线程并发场景下性能更优。该特性源自于Log4j 2的异步模式采用了Disruptor来处理。
在Log4j 2的配置文件中可以配置

### 参考资料 https://www.cnblogs.com/276815076/p/3503923.html
### 参考资料 https://tech.meituan.com/disruptor.html
