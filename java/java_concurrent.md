

线程
#### 进程与线程

进程是代码在数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位。

线程则是进程的一个执行路径，一个进程中至少有一个线程，进程中的多个线程共享进程的资源。

操作系统在分配资源时是把资源分配给进程的，但是CPU资源比较特殊，它是被分配到线程的。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_thread.png)

堆是一个进程中最大的一块内存，堆是被进程中的所有线程共享的，是进程创建时分配的，堆里面主要存放使用new操作创建的对象实例。

方法区则用来存放JVM加载的类、常量及静态变量等信息，也是线程共享的。

程序计数器，是为了记录该线程让出CPU时的执行地址的，待再次分配到时间片时线程就可以从自己私有的计数器指定地址继续执行。

每个线程都有自己的栈资源，用于存储该线程的局部变量，这些局部变量是该线程私有的，其他线程是访问不了的，除此之外栈还用来存放线程的调用栈帧。

#### 线程的优雅关闭：

一个线程一旦运行起来，就不要去强行打断它，合理的关闭办法是让其运行完（也就是函数执行完毕），干净地释放掉所有资源，然后退出。

在Java中，有stop（）、destory（）之类的函数，但这些函数都是官方明确不建议使用的。原因很简单，如果强制杀死线程，则线程中所使用的资源，例如文件描述符、网络连接等不能正常关闭。

#### 守护线程

Thread.setDaemon(true). 即可将一个线程设置为守护线程。

当在一个JVM进程里面开多个线程时，这些线程被分成两类：守护线程和非守护线程。默认开的都是非守护线程。

在Java中有一个规定：当所有的非守护线程退出后，整个JVM进程就会退出。意思就是守护线程“不算作数”，守护线程不影响整个JVM进程的退出。

例如，垃圾回收线程就是守护线程，它们在后台默默工作，当开发者的所有前台线程（非守护线程）都退出之后，整个JVM进程就退出了。

#### interrupt和InterruptedException

只有那些声明了会抛出InterruptedException 的函数才会抛出InterruptedException异常，如sleep/wait/join/park，表示从阻塞中被唤醒。

轻量级阻塞和重量级阻塞：

* 能够被中断的阻塞称为轻量级阻塞，对应的线程状态是WAITING或者TIMED_WAITING；
* 而像synchronized 这种不能被中断的阻塞称为重量级阻塞，对应的状态是BLOCKED。

t.interrupt（）的精确含义是“唤醒轻量级阻塞”，而不是字面意思“中断一个线程”。
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_thread_state.png)

初始线程处于NEW状态，调用start（）之后开始执行，进入RUNNING或者READY状态。

如果没有调用任何的阻塞函数，线程只会在RUNNING和READY之间切换，也就是系统的时间片调度。

这两种状态的切换是操作系统完成的，开发者基本没有机会介入，除了可以调用yield（）函数，放弃对CPU的占用。

**t.interrupt()：**
t.interrupt（）相当于给线程发送了一个唤醒的信号，所以如果线程此时恰好处于WAITING或者TIMED_WAITING状态，就会抛出一个InterruptedException，并且线程被唤醒。

而如果线程此时并没有被阻塞，则线程什么都不会做。但在后续，线程可以判断自己是否收到过其他线程发来的中断信号

**t.isInterrupted()与Thread.interrupted():**
两个方法都用于判断线程是否收到过中断信号。

t.isInterrupted()，为非静态函数，只读取线程中断状态，不修改中断状态。

Thread.interrupted(),为静态函数，读取线程中断中途，同时重置中断状态位。


#### synchronized

**实现原理：**

在Java对象头里，有一块数据叫Mark Word。在64位机器上，Mark Word是8字节（64位）的，这64位中有2个重要字段：锁标志位和占用该锁的thread ID。

若synchronized用于修饰某个对象或方法的非静态方法，则锁位于该对象的对象头内；

若synchronized用于修饰类的静态方法或静态成员变量，则锁位于该类的class变量的对象头内。

**加锁与释放：**

可以将wait/notify、Condition等与synchronized合作，实现加锁与锁释放时的通知。

以wait/notify为例：在wait（）的内部，会先释放锁obj1，然后进入阻塞状态，之后，它被另外一个线程用notify（）唤醒，去重新拿锁！其次，wait（）调用完成后，执行后面的业务逻辑代码，然后退出synchronized同步块，再次释放锁。
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_synchronized_wait.png)

#### wait

经典使用方式：

```
synchronized(obj) {
	while ( 条件不满足) {
		obj.wait();
	}
}

synchronized(obj) {
	obj.notify();
}

```

#### notify()与notifyAll()
notify随机唤醒一个wait的线程，notifyAll唤醒所有wait的线程，都无法实现精准唤醒。

被唤醒的线程不能马上从wait方法返回并继续执行，它必须在获取了共享对象的监视器锁后才可以返回。

利用Condition或park/unpark可以实现精准唤醒。

#### volatile与内存屏障

volatile的三重功效：64位写入的原子性、内存可见性和禁止重排序。

**内存可见性：**

因为存在CPU缓存一致性协议，例如MESI，多个CPU之间的缓存不会出现不同步的问题，不会有“内存可见性”问题。

但是，缓存一致性协议对性能有很大损耗，为了解决这个问题，CPU 的设计者们在这个基础上又进行了各种优化。例如，在计算单元和L1之间加了Store Buffer、Load Buffer（还有其他各种Buffer）.
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_volatile_jmm.png)
L1、L2、L3和主内存之间是同步的，有缓存一致性协议的保证，但是Store Buffer、Load Buffer和L1之间却是异步的。

也就是说，往内存中写入一个变量，这个变量会保存在Store Buffer里面，稍后才异步地写入L1中，同时同步写入主内存中。

多CPU，每个CPU多核，每个核上面可能还有多个硬件线程，对于操作系统来讲，就相当于一个个的逻辑CPU。每个逻辑CPU都有自己的缓存，这些缓存和主内存之间不是完全同步的。

**重排序：**

* 编译器重排序。对于没有先后依赖关系的语句，编译器可以重新调整语句的执行顺序。
* CPU指令重排序。在指令级别，让没有依赖关系的多条指令并行。
* CPU内存重排序。CPU有自己的缓存，指令的执行顺序和写入主内存的顺序不完全一致。StoreBuffer的延迟写入，即属于内存重排序。

无论什么语言，站在编译器和CPU的角度来说，不管怎么重排序，单线程程序的执行结果不能改变，这是单线程程序的重排序规则。

而在多线程环境下，程序运行结果的确定性无法保证。

**happen-before：**

为了明确定义在多线程场景下，什么时候可以重排序，什么时候不能重排序，Java 引入了**JMM（Java Memory Model）**，也就是Java内存模型（单线程场景不用说明，有as-if-serial语义保证）。

这个模型就是一套规范，对上，是JVM和开发者之间的协定；对下，是JVM和编译器、CPU之间的协定。

如果**A happen-before B**，意味着A的执行结果必须对B可见，也就是保证跨线程的内存可见性。

A happen-before B不代表A一定在B之前执行。因为，对于多线程程序而言，两个操作的执行顺序是不确定的。

基于happen-before的这种描述方法，JMM对开发者做出了一系列承诺：

* 单线程中的每个操作，happen-before 对应该线程中任意后续操作（也就是as-if-serial语义保证）。
* 对volatile变量的写入，happen-before对应后续对这个变量的读取。
* 对synchronized的解锁，happen-before对应后续对这个锁的加锁。
* 对final变量的写，happen-before于final域对象的读，happen-before于后续对final变量的读。
另外，happen-before还具有传递性，即若A happen-before B，Bhappen-before C，则A happen-before C。

**内存屏障：**

内存屏障是JMM和happen-before的底层基础。

1. 编译器内存屏障，告诉编译器不要对指令进行重排序。编译完成后，这种内存屏障就消失了。
2. CPU内存重排序，可以由开发者显式调用。

在理论层面，可以把基本的CPU内存屏障分成四种：

* LoadLoad：禁止读和读的重排序。
* StoreStore：禁止写和写的重排序。
* LoadStore：禁止读和写的重排序。
* StoreLoad：禁止写和读的重排序。

**volatile实现：**

由于不同的CPU架构的缓存体系不一样，重排序的策略不一样，所提供的内存屏障指令也就有差异。

为了实现volatile关键字的语义的一种参考做法：

1. 在volatile写操作的前面插入一个StoreStore屏障。保证volatile写操作不会和之前的写操作重排序。
2. 在volatile写操作的后面插入一个StoreLoad屏障。保证volatile写操作不会和之后的读操作重排序。
3. 在volatile读操作的后面插入一个LoadLoad屏障+LoadStore屏障。保证volatile读操作不会和之后的读操作、写操作重排序。

即volatile修饰的变量，在代码中被访问之处，会根据需要插入不同的内存屏障，来实现禁止重排序的目的。
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_volatile_base.png)

#### 无锁编程

* 一写一读的无锁队列：内存屏障
* 一写多读的无锁队列：volatile关键字
* 多读多写的无锁队列：CAS
* 无锁栈：对head指针进行CAS操作
* 无锁链表：例子如ConcurrentSkipListMap



### Atomic

#### AtomicInteger和AtomicLong

AtomicInteger和AtomicLong都基于Unsafe实现CAS操作，以此进行update。update时，自旋更新value，避免使用悲观锁。

```
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    private volatile int value;
    
    public final long accumulateAndGet(long x,
                                       LongBinaryOperator accumulatorFunction) {
        long prev, next;
        do {
            prev = get();
            next = accumulatorFunction.applyAsLong(prev, x);
        } while (!compareAndSet(prev, next)); //自旋CAS，更新value
        return next;
    }
```

#### AtomicBoolean和AtomicReference

在Unsafe类中，只支持int、long、Object三种类型的CAS操作。AtomicBoolean的实现，通过int（0/1）和boolean互转来实现。

#### AtomicStampedReference和AtomicMarkableReference

AtomicStampedReference中有一个版本号stamp，用于解决ABA问题。

```
public class AtomicStampedReference<V> {

    private static class Pair<T> {
        final T reference;
        final int stamp;	 //版本号
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }

    private volatile Pair<V> pair;
```    

当expectedReference==对象当前的reference时，再进一步比较expectedStamp是否等于对象当前的版本号，以此判断数据是否被其他线程修改过。

AtomicMarkableReference与AtomicStampedReference原理类似，只是Pair里面的版本号是boolean类型的，而不是整型的累加变量。

#### AtomicIntegerFieldUpdater、AtomicLongFieldUpdater和AtomicReferenceFieldUpdater

如果是一个已经有的类，在不能更改其源代码的情况下，要想实现对其成员变量的原子操作，就需要AtomicIntegerFieldUpdater。

AtomicIntegerFieldUpdater基于**反射**获取类的成员，也利用CAS实现修改操作。

#### AtomicIntegerArray、AtomicLongArray和Atomic-ReferenceArray

同样基于Unsafe CAS，实现对数组中单个元素的原子操作。

#### Striped64与LongAdder

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_strip64.png)

AtomicLong内部是一个volatile long型变量，由多个线程对这个变量进行CAS操作。多个线程同时对一个变量进行CAS操作，在高并发的场景下仍不够快。

而在LongAdder中，把一个变量拆成多份，变为多个变量，有些类似于ConcurrentHashMap 的分段锁的例子。

把一个Long型拆成**一个base变量外加多个Cell**，每个Cell包装了一个Long型变量。

当多个线程并发累加的时候，如果并发度低，就直接加到base变量上；如果并发度高，冲突大，平摊到这些Cell上。在最后取值的时候，再把base和这些Cell求sum运算。

在sum求和函数中，并没有对cells[]数组加锁。也就是说，一边有线程对其执行求和操作，一边还有线程修改数组里的值，也就是**最终一致性**，而不是强一致性。

**伪共享与缓存行填充：**

在LongAdder的Cell类的定义中，用了一个独特的注解@sun.misc.Contended。

这是JDK 8之后才有的，之所以这个地方要用缓存行填充，是为了不让Cell[]数组中相邻的元素落到同一个缓存行里，避免缓存刷新时互相影响。

** LongAccumulator：**

 LongAccumulator比LongAdder功能更强大，可以定义初始值，还可以自定义二元操作。
 
 
### Lock与Condition

#### ReentrantLock

 ReentrantLock基于内部类Sync实现，主要逻辑位于AQS，Sync有FairSync和NonFairSync两个子类。默认为非公平锁，对应NonFairSync。
 
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_reentrant_uml.png) 

 ReentrantLock主要内容：
 
 * private volatile int state; 标记锁的状态。
 * private transient Thread exclusiveOwnerThread; 记录当前持有锁的线程。
 * 基于park/unpark原语实现对一个线程的阻塞/精准唤醒操作。
 * 基于CAS实现一个线程安全的无锁队列，维护所有阻塞的线程。

ReentrantLock中state可以为0，表示当前没有加锁；也可以为正数，表示当前线程重入次数，每重入lock一次state加一，每unlock一次state减一。


#### ReentrantReadWriteLock

从表面来看，ReentrantReadWriteLock 有ReadLock和WriteLock是两把锁，实际上它只是同一把锁的两个视图而已。

可以理解为是一把锁，线程分成两类：读线程和写线程。读线程和读线程之间不互斥（可以同时拿到这把锁），读线程和写线程互斥，写线程和写线程也互斥。
 
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_reentrantreadwrite_uml.png)  

ReentrantReadWriteLock中，state被拆成两半：

* 低16位，用于记录写锁。为0时，表示没有加写锁；为正数时，表示写线程重入次数；
* 高16位，用于记录读锁。

为什么要把一个int类型变量拆成两半，而不是用两个int型变量分别表示读锁和写锁的状态呢？这是因为无法用一次CAS 同时操作两个int变量，所以用了一个int型的高16位和低16位分别表示读锁和写锁的状态。

#### Condition

Condition本身也是一个接口，其功能和wait/notify类似。

Condition必须与Lock一起使用，如ReentrantLock或ReentrantReadWriteLock的WriteLock等。

Condition基于AQS中ConditionObject实现，其中有一个双向链表组成的队列，记录阻塞的线程。


#### StampedLock

JDK8新增StampedLock，实现了读写不互斥，可以提高并发度。

三种锁的并发度对比：

* ReentrantLock，无论读写，均互斥；
* ReentrantReadWriteLock，读与读不互斥，读写互斥，写写互斥；
* StampedLock，读读不互斥，读写不互斥，写写互斥；

StampedLockn内部不是基于AQS实现，而是重新实现了一个阻塞队列。

其中state被分为三部分，低七位表示读锁状态，第八位表示写锁状态，剩余表示版本号。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_stampedlock_class.png)

* 乐观读tryOptimisticRead时，返回版本号，相当于读之前给数据做了一次快照；
* 之后，使用时再通过validate对比版本号；
* 如果版本号不一致，则通过readLock升级为悲观读。
 
 另外，StampedLock中将读线程串联在一起，唤醒读线程时会一起唤醒所有的读线程，从而实现读线程间不互斥。

 
### 同步工具类
 
#### Semaphore

基于AQS实现，支持一定资源数量的并发访问控制。

```
Semaphore semaphore = new Semaphore(10);
semaphore.acquire();
semaphore.release();
```
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_semaphore_uml.png)

#### CountDownLatch

基于AQS实现，支持state递减，直到state为0时唤醒所有阻塞的线程。

```
CountDownLatch latch = new CountDownLatch(1);
latch.await();
latch.countDown();
```
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_countdownlatch_uml.png)

#### CyclicBarrier
CyclicBarrier基于ReentrantLock+Condition实现。

如下所示，累计10个线程调用await阻塞时，即可唤醒所有线程。CyclicBarrier可重复使用。

```
CyclicBarrier cb = new CyclicBarrier(10);
cb.await();
```

#### Exchanger

用于线程间交换数据，基于CAS+park/unpark实现。


#### Phaser

Phaser可代替CountDownLatch和CyclicBarrier使用，同时有新功能：

* 可动态调整要同步的线程个数。
* 可构建有层次的Phaser。

不基于AQS实现，但是也基于CAS+state变量+阻塞队列来实现。

### 并发容器

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_queue_uml.png)

ArrayBlockingQueue，一个用数组实现的环形队列。

LinkedBlockingQueue，一种基于单向链表的阻塞队列。

LinkedBlockingQueue实现是线程安全的，实现了先进先出等特性，是作为生产者消费者的首选。

PriorityQueue，队列通常是先进先出的，而PriorityQueue是按照元素的优先级从小到大出队列的。

DelayQueue，即延迟队列，也就是一个按延迟时间从小到大出队的PriorityQueue。

SynchronousQueue，先调put（..），线程会阻塞；直到另外一个线程调用了take（），两个线程才同时解锁，反之亦然。对于多个线程而言，例如3个线程，调用3次put（..），3个线程都会阻塞；直到另外的线程调用3次take（），6个线程才同时解锁，反之亦然。

BlockingDeque，一个阻塞的双端队列接口。

CopyOnWriteArrayList：

CopyOnWrite，指在“写”的时候，不是直接“写”源数据，而是把数据拷贝一份进行修改，再通过悲观锁或者乐观锁的方式写回。那为什么不直接修改，而是要拷贝一份修改呢？这是为了在“读”的时候不加锁。


**ConcurrentLinkedQueue**

ConcurrentLinkedQueue 的实现原理和AQS 内部的阻塞队列类似：同样是基于CAS，同样是通过head/tail指针记录队列头部和尾部，但还是有稍许差别。

但在ConcurrentLinkedQueue中，head/tail的更新可能落后于节点的入队和出队，因为它不是直接对head/tail指针进行CAS操作的，而是对Node中的item进行操作。

ConcurrentLinkedQueue的size方法要遍历所有元素，比较慢，尽量使用其isEmpty方法而不要使用其size方法。

**ConcurrentHashMap**

HashMap通常的实现方式是“数组+链表”，这种方式被称为“拉链法”。ConcurrentHashMap在这个基本原理之上进行了各种优化，在JDK 7和JDK 8中的实现方式有很大差异。

1. JDK1.7中基于Segment**分段锁**实现，每个segment是一个hashmap。

每个Segment都继承自ReentrantLock，Segment的数量等于锁的数量，这些锁彼此之间相互独立，即所谓的“分段锁”。

Segment的个数不能扩容，但每个Segment的内部可以扩容。

好处：

* 减少Hash冲突，避免一个槽里有太多元素。
* 提高读和写的并发度。段与段之间相互独立。
* 提供扩容的并发度。扩容的时候，不是整个ConcurrentHashMap 一起扩容，而是每个Segment独立扩容。

2. JDK1.8中去掉了分段锁，所有数据都放在一个大的HashMap中；其次是引入了红黑树。

如果头节点是Node类型，则尾随它的就是一个普通的链表；如果头节点是TreeNode类型，它的后面就是一颗红黑树，TreeNode是Node的子类。

链表和红黑树之间可以相互转换：初始的时候是链表，当链表中的元素超过某个阈值时，把链表转换成红黑树；反之，当红黑树中的元素个数小于某个阈值时，再转换为链表。

特点：

* 使用红黑树，当一个槽里有很多元素时，其查询和更新速度会比链表快很多，Hash冲突的问题由此得到较好的解决。
* 加锁的粒度，并非整个ConcurrentHashMap，而是对每个头节点分别加锁，即并发度，就是Node数组的长度，初始长度为16，和在JDK 7中初始Segment的个数相同。
* **并发扩容**，这是难度最大的。在JDK 7中，一旦Segment的个数在初始化的时候确立，不能再更改，并发度被固定。之后只是在每个Segment内部扩容，这意味着每个Segment独立扩容，互不影响，不存在并发扩容的问题。但在JDK 8中，相当于只有1个Segment，当一个线程要扩容Node数组的时候，其他线程还要读写，因此处理过程很复杂。
* 初始化时，多个线程的竞争通过对sizeCtl进行CAS操作实现，保证只有一个线程负责初始化。

如下图为扩容的基本图示。每次扩容，会将数组长度扩大到原来的两倍。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_concurrenthashmap_kuorong1.png)

扩容时可以并发扩容，每个线程负责一段。线程在put元素时，检测到正在扩容，就可以帮助扩容。

如下为多个线程并行扩容-任务划分示意图。旧数组的长度是N，每个线程扩容一段，一段的长度用变量stride（步长）来表示，transferIndex表示了整个数组扩容的进度。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_concurrenthashmap_kuorong2.png)

transferIndex是ConcurrentHashMap的一个成员变量，记录了扩容的进度。初始值为n，从大到小扩容，每次减stride个位置，最终减至n＜=0，表示整个扩容完成。


**HashMap**

基于红黑树实现，非线程安全。

基于modCount进行并发检测，modCount用来表示map内部发生结构性变化的次数。

如果在访问期间modCount发生变化，即map内部结构性变化（如发生了rehash/mapping数目变化），则会报ConcurrentModificationException。


**ConcurrentSkipListMap**

提供的key有序的HashMap，基于SkipList（跳查表）实现的。

SkipList基于无锁链表实现节点的增加、删除。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_skiplistmap.png)



### 线程池

线程池的实现原理：调用方不断地向线程池中提交任务；线程池中有一组线程，不断地从队列中取任务，这是一个典型的生产者—消费者模型。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_threadpool_uml.png)

**初始化参数：**

* corePoolSize：在线程池中始终维护的线程个数。
* maxPoolSize：在corePooSize已满、队列也满的情况下，扩充线程至此值。
* keepAliveTime/TimeUnit：maxPoolSize 中的空闲线程，销毁所需要的时间，总线程数收缩回corePoolSize。
* blockingQueue：线程池所用的队列类型。
* threadFactory：线程创建工厂，可以自定义，也有一个默认的。
* RejectedExecutionHandler：corePoolSize 已满，队列已满，maxPoolSize 已满，最后的拒绝策略。

**提交任务的处理流程：**

* step1：判断当前线程数是否大于或等于corePoolSize。如果小于，则新建线程执行；如果大于，则进入step2。
* step2：判断队列是否已满。如未满，则放入；如已满，则进入step3。
* step3：判断当前线程数是否大于或等于maxPoolSize。如果小于，则新建线程执行；如果大于，则进入step4。
* step4：根据拒绝策略，拒绝任务。

线程池状态变化：
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_threadpool_state.png)

shutdown（）与shutdownNow（）的区别：

* 前者不会清空任务队列，会等所有任务执行完成，后者再清空任务队列。
* 前者只会中断空闲的线程，后者会中断所有线程。

线程池的关闭需要一个过程，在调用shutDown（）或者shutdownNow（）之后，线程池并不会立即关闭，接下来需要调用awaitTermination 来等待线程池关闭。


**线程池的4种拒绝策略：**

* CallerRunsPolicy，由调用者直接在自己的线程里执行，线程池不做处理；
* AbortPolicy，线程池抛出异常；
* DiscardPolicy，线程池直接丢弃任务；
* DiscardOldestPolicy，线程池删除队列里最老的任务，把新任务放入队列；

#### Callable与Future

Callable就是一个有返回值的Runnable。

ThreadPoolExecutor中利用FutureTask作为一个Adapter对象，将Callable转换为Runnable来执行。

#### ScheduledThreadPoolExecutor

按时间调度来执行任务。

* AtFixedRate：按固定频率执行，与任务本身执行时间无关
* WithFixedDelay：按固定间隔执行，与任务本身执行时间有关。

延迟执行任务依靠的是DelayQueue。

#### Executors工具类

可用于创建不同类型的线程池。

在《阿里巴巴Java开发手册》中，明确禁止使用Executors创建线程池，并要求开发者直接使用ThreadPoolExector或ScheduledThreadPoolExecutor进行创建。

原因是Executors工具类返回的线程池对象的弊端：

1. FixedThreadPool和SingleThreadPool，允许的队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。
2. CachedThreadPool和ScheduledThreadPool，允许创建的线程数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。


### ForkJoinPool

使用：
```
@Test
public void testForkJoinPool() throws ExecutionException, InterruptedException {
    ForkJoinPool pool = ForkJoinPool.commonPool();
    Future<Integer> future = pool.submit(new AddAction(1, 100));
    System.out.println("sum: " + future.get());
}

class AddAction extends RecursiveTask<Integer> {
    int l;
    int h;

    public AddAction( int l, int h) {
        this.l = l;
        this.h = h;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        if (l + 5 >= h){
            for (int i = l; i<=h; i++){
                sum  += i;
            }
        } else {
            int mid = (l + h) >>> 1;
            AddAction left = new AddAction(l, mid);
            AddAction right = new AddAction(mid+1, h);
            left.fork(); //把自己放入当前线程所在的局部队列中
            right.fork(); 
            sum = left.join() + right.join(); // join会导致线程的层层嵌套阻塞
        }
        return sum;
    }
}
```    

ForkJoinTask

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_forkjointask.png)

ForkJoinPool内部结构如下图。除全局队列外，每个线程还有自己的局部队列。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_forkjoinpool.png)

**工作窃取队列**

ForkJoinPool内部队列不是基于BlockingQueue实现，而是基于一个普通的数组，名为工作窃取队列，为工作窃取算法服务。

工作窃取算法，是指一个Worker线程在执行完毕自己队列中的任务之后，可以窃取其他线程队列中的任务来执行，从而实现负载均衡，以防有的线程很空闲，有的线程很忙。这个过程要用到工作窃取队列。

（1）整个队列是环形的，也就是一个数组实现的RingBuffer
（2）当队列满了之后会扩容，所以被称为是动态的

### CompletableFuture

**主要方法：**

* get，阻塞在这里，直到获取返回结果；
* 提交任务：runAsync与supplyAsyn
* 链式的CompletableFuture：thenRun、thenAccept和thenApply
* CompletableFuture的组合：thenCompose与thenCombine
* 任意个CompletableFuture的组合：allOf 和anyOf

CompletableFuture不仅实现了Future接口，还实现了CompletableStage接口。

CompletableFuture中任务的执行同样依靠ForkJoinPool。

**任务的网状执行：有向无环图**

任何一个多元操作，都能被转换为多个二元操作的叠加

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_completablefuture_dag.png)


### 参考：

《Java并发实现原理》

《Java并发编程之美》