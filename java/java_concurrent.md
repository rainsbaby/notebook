线程

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

#### notify()与notifyAll()
notify随机唤醒一个wait的线程，notifyAll唤醒所有wait的线程，都无法实现精准唤醒。

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











![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_queue_uml.png)



LinkedBlockingQueue

LinkedBlockingQueue实现是线程安全的，实现了先进先出等特性，是作为生产者消费者的首选，



ConcurrentLinkedQueue 

ConcurrentLinkedQueue 的实现原理和AQS 内部的阻塞队列类似：同样是基于CAS，同样是通过head/tail指针记录队列头部和尾部，但还是有稍许差别。

但在ConcurrentLinkedQueue中，head/tail的更新可能落后于节点的入队和出队，因为它不是直接对head/tail指针进行CAS操作的，而是对Node中的item进行操作。

ConcurrentLinkedQueue的size方法要遍历所有元素，比较慢，尽量使用其isEmpty方法而不要使用其size方法。



ConcurrentHashMap
JDK1.7中基于Segment分段锁实现，每个segment是一个hashmap。

JDK1.8中去掉了分段锁，改用一个大的hashmap。


HashMap

基于红黑树实现，非线程安全。

基于modCount进行并发检测，modCount用来表示map内部发生结构性变化的次数。

如果在访问期间modCount发生变化，即map内部结构性变化（如发生了rehash/mapping数目变化），则会报ConcurrentModificationException。


ConcurrentSkipListMap

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_skiplistmap.png)

线程池

初始化参数：

线程池状态变化：
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/java_concurrent/java_concurrent_threadpool_state.png)

shutdown（）与shutdownNow（）的区别：

线程池的4种拒绝策略：


#### Executors工具类

在《阿里巴巴Java开发手册》中，明确禁止使用Executors创建线程池，并要求开发者直接使用ThreadPoolExector或ScheduledThreadPoolExecutor进行创建。

原因是Executors工具类返回的线程池对象的弊端：

1. FixedThreadPool和SingleThreadPool，允许的队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。
2. CachedThreadPool和ScheduledThreadPool，允许创建的线程数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。

