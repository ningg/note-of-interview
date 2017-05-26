## 什么叫线程安全？举例说明

几点吧：

1. 线程安全，是针对`对象`来说的；
2. 多线程，`并发访问` `对象`时，不需要额外的**同步机制**，对象始终保持可预期的输出
3. 不可变对象、无状态对象、自带锁机制的对象，都是`线程安全`的


##  进程和线程的区别

* 进程：Mem 资源分配的基本单元
* 线程：CPU 调度的基本单元
	* 线程依附于进程
	* 线程的上下文切换代价，远远小于进程的上下文切换


## volatile的理解

### volatile 修饰变量的作用：

volatile 变量，自身的特性，简单来说：

* 可见性：一个线程的更新操作，其他线程**立即可见**；
* 原子性：对 volatile 变量的`读`、`写`操作，是原子的，但`自增`操作不是原子的；

happen-before 原则：多线程并发的时候，有序性

* volatile 变量的`写-读`：`写`先发生
* 锁（monitor）的`释放-获取`：`释放`先发生

指令重排，优点？为什么有指令重排操作？

* **代码执行过程**，可以细分为：`取址`、`译码`、`执行`等过程，构成**流水**，指令重排，能够充分利用 CPU 资源；
* **多 CPU 情况下**：指令重排提升效率（Note：单 CPU 环境，指令重排失效）
* **指令重排**：包含`编译器指令重排`和 `CPU 指令重排`


详细来说：

1. **内存，可见性**：一个线程的更新操作，其他线程**立即可见**；
	* volatile 变量的更新仍然是发生在`工作内存`中；
	* volatile 约束：
		* **读取操作**：使用变量时，每次都从`主内存`获取最新值；read、load、use 连续
		* **更新操作**：更新变量后，立即写入到`主内存`；assign、store、write 连续
		* note：
			* **JMM 内存模型**：所有变量的更新操作，都发生在`工作内存`中；`工作内存`和`主内存`，存在`数据同步延时`，因此，数据不一致；
			* volatile 保证了变量在`主内存`和`工作内存`的一致性，同一时刻，所有的工作内存，都看到相同的取值；
			* volatile 约束编译器：将几个原子操作绑定在一起，保证`内存可见性` 
2. **变量读取和赋值操作的原子性**：
	* **非原子性协议**：JVM 规范，允许 VM 实现的时候，针对 64 bit 类型的变量赋值操作，拆分为 2 次，每次复制 32 bit；（long、float，64 bit）
	* **volatile 修饰的 64 bit**，会保证变量`读取`和`赋值`是`原子`的；
3. **线程间，有序性**：volatile 的读写操作，禁止指令重排序
	* **volatile 有序性**：本质是，`编译器`\`CPU`，通过JMM 底层的 lock 操作，设置`内存屏障`
		* **写操作之前的操作，先发生**：同一个线程内，volatile `赋值`操作，其前面的所有操作，一定先发生，不允许指令重排
		* **读操作之后的操作，后发生**：同一个线程内，volatile `读取`操作，其后的所有操作，一定后发生，不允许指令重排
	* **内存屏障**：`指令重排序`，无法跨过`内存屏障`
	* `指令重排`：保证`线程内`，`串行语义`；不保证`线程间`的语义。

### volatile 是轻量级的 synchronized？

* **两种方式，都是用于解决**：`多线程并发`，线程的`有序性`
* **volatile 本质**：使用 JMM 内存模型中 lock 原子操作，实现的线程间有序性；
* **synchronized 本质**：class 字节码 monitor enter 和 monitor exit 实现的线程间有序；
	* monitor enter 相对 lock 更高一层，涉及操作更多一些
	* 现状：synchronized 经过多次的`优化`，效率已经接近 volatile
* **volatile 和 synchronized** 是通过保证：`happen-before` 原则（`先行发生`），实现线程并发时，有序性的；
* **happen-before 原则**：先执行的线程A，产生的执行效果（变量赋值等），对于后执行的线程 B，是可见的。
	* lock：synchronized 的 monitor exit 发生在 monitor enter 之前；
		* 一个变量，只能被一个线程 lock；
		* 同一个线程，可以 lock 同一个变量多次，但，需要 unlock 相同次数；
		* unlock 变量之前，必须把变量值，写回到`主内存`；
		* 只有 unlock 变量后，其他线程才能 lock 这个变量；
	* volatile：volatile 变量的写操作，发生在读操作之前；
		* volatile 变量的更新操作，对其他线程的读取，是立即可见的；

		
## 原子性实现机制

**处理器**（CPU）提供`总线锁定`和`缓存锁定`两种方式，来保证复杂内存操作的**原子性**。

* **总线型**：就是使用处理器提供一个LOCK信号，当一个处理器在总线传输信号时，其他处理器的请求将被阻塞住，那么该处理独占内存。所以总线锁定开销大。
* **缓存锁定**：内存区域如果被缓存在缓存行中，且在在lock期间被锁定，当它执行锁操作写回内存时，处理器总线不在锁定而是通过修改内部的内存地址并使用缓存一致性制阻止同时修改保证操作的原子性。缓存一致性进制两个以上的处理器同时修改内存区域数据，其他处理器回写被锁定并且使其缓存行无效。

## Java 原子性操作实现原理

`AtomicInteger` 使用`循环CAS`实现**原子自增**操作，CAS是在操作期间先比较旧值，如果旧值没有发生改变，才交换成新值，发生了变化则不交换。

无锁的原子性操作，性能更好：

* 避免了锁竞争和加解锁的损耗。

这种方式会产生以下几种问题：

1. **ABA问题**，通过加**版本号**解决；
	* `AtomicInteger` 未添加版本号
	* `AtomicStampedReference<V>` 中，
		* 使用 `Pair` 对象，封装 value 和 stamp
		* `Pair` 对象，为 volatile
2. **循环时间过长**开销大，一般采用`自旋`方式实现；
3. 只能保证一个**共享变量**的`原子操作`。 

AtomicInteger 的操作：

1. 内部属性：
	1. 取值：`volatile int value`，volatile 保证`读`&`写`操作的**原子性**；
	2. 偏移量：`final long valueOffset`，静态代码块中，获取对象的 value 偏移量；
2. 具体操作：
	1. 自增：`getAndIncrement()`，**CAS 循环**，利用 valueOffset，底层使用 CAS 原子指令（Unsafe 类）
3. 补充说明：
	1. `volatile`：内存可见，`读`&`写`操作是**原子**的，但`自增`操作不是原子的；
	2. 使用`循环 CAS`机制，乐观锁，实现原子`自增`；

AtomicInteger 的自增操作，代码示例：

```
    /**
     * Atomically increments by one the current value.
     *
     * @return the previous value
     */
    public final int getAndIncrement() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return current;
        }
    }
```



## Java内存模型

Java内存模型控制线程之间的通信，决定了一个线程对共享变量的写入何时对另一个线程可见。它属于语言及的内存模型，它确保在不同编译器和不同的处理平台之上，通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证。JMM的核心目标是找到一个好的平衡点，一方面是为程序员提供足够强的内存可见性保证（提供happens-before规则），另一方面对编译器和处理器的限制尽可能地放松（只要不改变程序结果，怎么优化都可以）

**可见性保证**
 为了提供内存可见性保证，JMM向程序员保证了以下hapens-before规则:
1.程序顺序规则：一个线程的每个操作happen-before与该线程的任意后续操作。
2.监视器锁规则：一个锁的解锁，happens-before于随后这个锁的加锁。
3.Volatile变量规则：对一个volatile域的写，happens-before于任意后续这个域的读。
4.传递性, 如果A happens-before B, 且B happens-before C 那么A happens-before C
5.线程启动规则：如果线程A执行操作ThreadB.start().那么线程A中的任意操作happens-before与线程B中的任意操作。
6.线程结束规则: 线程中的任何操作都必须在其线程检测到该线程已经结束之前执行，或者从Thread.join中成功返回，或者在调用Thread.isAlive时返回false.
7.中断规则:当一个线程在另一个线程上调用interrupt时，必须在被中断线程检测到interrupt调用之前执行(通过抛出InterruptedException,或者调用isInterrupted和interrupted)
8.终结器规则: 对象的构造函数必须在启动该对象的终结器之前执行完成。
 
**禁止重排序**
为了保证内存可见性，java编辑器在生成指令序列的适当位置插入内存屏障指令来禁止特定类型的处理器重排序。
重排序：编译器和处理器为了优化程序性能对指令进行重新排序的一种手段。
1)编译器优化的重排序: 编译器在不改变单线程程序语义的前提下可以重新安排语句顺序。
2)指令级并行的重排序.现代处理器采用指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变对应指令的执行顺序。
3)内存系统重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能在乱序执行。

## Final域的内存语义

对于final域编译器和处理器要遵守两个重排序规则：

1.在构造器函数内对final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。（保证了对象引用为任何线程可见之前，对象的final域已经被正确初始化过）
2.初次读一个包含final域的对象引用，与随后初次读这个final域这两个操作不能重排序。
为何保证其内存语义：可以为java程序员提供安全保证，只要对象是正确构造的，那么不需要使用同步就可以保证线程都能看到这个fianal域在构造函数中被初始化之后的值。

## 死锁的必要条件？怎么克服？

答：产生死锁的四个必要条件：

1. **互斥条件**/**临界资源**存在：一个资源每次只能被一个进程使用。
1. **请求与保持资源**：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
1. **不抢占资源**：进程已获得的资源，在末使用完之前，不能强行剥夺。
1. **循环等待资源**：若干进程之间形成一种头尾相接的循环等待资源关系。

这四个条件是死锁的`必要条件`，只要系统发生死锁，这些条件必然成立，而只要上述条件之一不满足，就不会发生死锁。

如何避免，死锁发生：

1. 设计上，避免线程同时占用多个资源，尽量保证只占用一个资源；
2. 使用**定时锁**，`lock.tryLock(timeout)` 替代 synchronized 的循环等待锁；

死锁已经发生，解决方法:

1. 撤消陷于死锁的全部进程；
2. 逐个撤消陷于死锁的进程，直到死锁不存在；
3. 从陷于死锁的进程中逐个强迫放弃所占用的资源，直至死锁消失。
4. 从另外一些进程那里强行剥夺足够数量的资源分配给死锁进程，以解除死锁状态

## CountDownLatch(闭锁) 与CyclicBarrier(栅栏)的区别

CountDownLatch: 允许一个或多个线程等待其他线程完成操作. CyclicBarrier：阻塞一组线程直到某个事件发生。 

1. 闭锁用于等待事件、栅栏是等待线程.
2. 闭锁CountDownLatch做减计数，而栅栏CyclicBarrier则是加计数。
3. CountDownLatch是一次性的，CyclicBarrier可以重用。
4. CountDownLatch一个线程(或者多个)，等待另外N个线程完成某个事情之后才能执行。CyclicBarrier是N个线程相互等待，任何一个线程完成之前，所有的线程都必须等待。 
CountDownLatch 是计数器, 线程完成一个就记一个,就像报数一样, 只不过是递减的.
而CyclicBarrier更像一个水闸, 线程执行就像水流, 在水闸处都会堵住, 等到水满(线程到齐)了, 才开始泄流.

## execute 和submit的区别

线程池 `ExecutorService` 接口，提供 2 种方法：

1. `execute()`：无任务返回值；
1. `submit()`：捕获任务返回值，Future 对象。

## shutdown和shutdownNow的区别

**shutdown()**：标记关闭线程池，会执行完所有已提交的 task

1. 线程池，标记为 SHUTDOWN 状态，不接收新的 task；
2. 所有已接受的 task，都会执行；
3. 关闭空闲状态的线程；

**shutdownNow()**：立即关闭线程池，终止正在执行的 task，清空等待执行的 task 列表

1. 线程池，标记为 STOP，不接收新的 task；
2. 中断正在执行的 task（调用 `interrupt()` 方法）；
3. 清除等待执行的 task；
4. 返回等待执行 task 的列表；

如何关闭指定的 task？

* ExecutorService 中，`submit()` task，跟 task 绑定了一个 Future 对象；
* 调用 Future 对象的 `cancel()` 方法；

特别说明：

> interrupt() 只是一个中断信号：
> 
> 1. 当线程状态为 `IO 阻塞` 或者 `synchronized 同步` 状态时，无法中断；
> 2. 线程为：sleep、wait 等状态时，会立即执行；
> 
> ShutdownNow() 并不代表线程立即终止执行，需要看线程的具体状态。


## ThreadLocal的设计理念与作用

ThreadLocal 是做什么用的？

* 每个 Thread，都维护一个变量副本；

具体实现细节：

* ThreadLocal，线程的一个本地变量，线程独占
* ThreadLocal 变量，存放在跟具体 Thread 内部的 ThreadLocalMap 中
* ThreadLocalMap 中，key 就是 ThreadLocal 对象，value 是具体的取值；
* ThreadLocalMap，作为一个 Map，底层是 `Entry[]` 数组，解决 Hash 冲突的方式是：**避让**策略（开放寻址）
* ThreadLocalMap，当 key 数量超过阈值，会 2 倍扩容
* ThreadLocalMap 的 `Entry[]` 数组，ThreadLocal 变量是`弱引用`，每次 GC 时，会尝试回收 ThreadLocal 变量；
* ThreadLocalMap，仍然有内存泄漏的风险，因为，只有在 ThreadLocalMap 调用 get()、set() 方法时，才会尝试回收 `Entry[]` 对象，但，可能一直不会调用 get\set 方法；

![](/assets/java/threadLocal-in-mem.png)

补充：

关于强引用（Strong）、软引用（Soft）、弱引用（Weak）、虚引用（Phantom）：

* **强引用**：我们平时使用的引用，Object obj = new Object(); 只要引用存在，GC 时，就不会回收对象；
* **软引用**：还有一些用，但非必需的对象，系统发生**内存溢出之前**，会回收软引用指向的对象；
* **弱引用**：非必需的对象，**每次 GC** 时，都会回收弱引用指向的对象；
* **虚引用**：不影响对象的生命周期，**每次 GC** 时，**都会回收**，虚引用用于在 GC 回收对象时，获取一个**系统通知**。


## 线程的状态

Java 中，定义了 5 种线程状态：

1. **New 新建**：创建后，尚未启动
2. **RUNNABLE 可运行状态**：对应 OS 中 `运行`与`就绪` 2 种状态
4. **WAITING 无限等待**：不占用 CPU 资源，需要`显式唤醒`，notify()、notifyAll()；
5. **TIME_WAITING 超时等待**：不占用 CPU 资源，一定时间后，可以自动唤醒；
6. **BLOCKED 阻塞状态**：等待`对象锁`，也有`超时阻塞`
6. **TERMINATED	 结束状态**：线程已经执行结束

![](/assets/java/thread-state-flow.png)

## 同步

同步就是协同步调，按预定的先后次序进行运行。

## sleep() 和 wait() 区别

几点：

1. **所属对象不同**：
	* `sleep()`，是 Thread 的静态方法
	* `wait()`，是 Object 的方法
2. **释放资源不同**：
	* `sleep()`，释放 `CPU 资源`；
	* `wait()`，释放`对象锁`，引发当前线程释放 `CPU 资源`，需要同一个对象执行 notify() 或 notifyAll() 唤醒；


## sleep() 和 yield() 区别

几方面：

1. 线程优先级：
	* sleep()，不考虑线程的优先级，释放 CPU 资源
	* yeild()，只给相同优先级或更高优先级线程，让出 CPU 资源
2. 可移植性：sleep() 通用性更强，不同 OS 上支持更友好




## 线程同步相关的方法


1. **wait()**:使一个线程处于等待（阻塞）状态，并且释放所持有的对象的锁，InterruptedException 异常；
1. **sleep()**:使一个正在运行的线程处于睡眠状态，是一个静态方法，调用此方法要捕捉InterruptedException 异常；
1. **notify()**:唤醒一个处于等待状态的线程，当然在调用此方法的时候，并不能确切的唤醒某一个等待状态的线程，而是由JVM确定唤醒哪个线程，而且与优先级无关；
1. **notityAll()**:唤醒所有处入等待状态的线程，注意并不是给所有唤醒线程一个对象的锁，而是让它们竞争；

## 什么是线程池（thread pool）

答：在面向对象编程中，创建和销毁对象是很费时间的，因为创建一个对象要获取内存资源或者其它更多资源。在 Java 中更是如此，虚拟机将试图跟踪每一个对象，以便能够在对象销毁后进行垃圾回收。所以提高服务程序效率的一个手段就是尽可能减少创建和销毁对象的次数，特别是一些很耗资源的对象创建和销毁，这就是"池化资源"技术产生的原因。线程池顾名思义就是事先创建若干个可执行的线程放入一个池（容器）中，需要的时候从池中获取线程不用自行创建，使用完毕不需要销毁线程而是放回池中，从而减少创建和销毁线程对象的开销。

http://www.cnblogs.com/dolphin0520/p/3932921.html

## ConcurrentHashMap实现原理

HashMap：非线程安全的

ConcurrentHashMap 和 Hashtable 主要区别：

* `锁的粒度`
	* ConcurrentHashMap：分段锁，锁住 segment；
	* Hashtable：锁住整个 segment；
* `如何锁`

Hashtable 在竞争激烈的环境下表现效率低下的原因是一把锁锁住整张表，导致所有线程同时竞争一个锁。
ConcurrentHashMap采用分段锁，每把锁锁住容器中的一个Segment。那么多线程访问容器里不同的Segment的数据时线程就不会存在竞争，从而有效提高并发访问效率。首先是将数据分层多个Segment存储，并为每个Segment分配一把锁，当一个线程范围其中一段数据时，其他线程可以访问其他段的数据。

数据结构：

ConcurrentHashMap内部是有Segment数组和HashEntry数组组成。一个ConcurrentHashMapMap里包含一个Segment数组，而Segment的结构和HashMap一样，里面是由一个数组和链表结构组成，所以一个Segment内部包含一个HashEntry数组。每个HashEntry是一个链表结构，对于HashEntry数组进行修改时首先需要获取与它对应的Segment锁。默认情况下有16个Segment

Segment的定位:
使用Wang/Jenkins hash变种算法对元素的hashCode进行一次再散列，目的是为了减少散列冲突。

ConcurrentHashMap 的操作:

### get 

1. 不加锁：原因
	* segment部分：通过 Unsafe 读取了最新的（volitale） segments 列表
	* segment内部：volitale 的 `HashEntry[] table`
	* HashEntry部分：HashEntry 中 `value` 和 `next` 是 volatile 的（key 是 final 的）
	* 新元素，追加在`拉链`的头部
2. 数组拉链，逐个遍历

get操作实现非常简单高效。先经过一次在散列，然后用这个散列值通过散列运算定位到Segment，再通过散列算法定位到元素。get之所以高效是因为整个get过程不需要加锁，除非读到空值才会加锁重读。实现该技术的技术保证是保证HashEntry是不可变的。

第一步是访问count变量，这是一个volatile变量，由于所有的修改操作在进行结构修改时都会在最后一步写count 变量，通过这种机制保证get操作能够得到几乎最新的结构更新。对于非结构更新，也就是结点值的改变，由于HashEntry的value变量是 volatile的，也能保证读取到最新的值。接下来就是根据hash和key对hash链进行遍历找到要获取的结点，如果没有找到，直接访回null。

对hash链进行遍历不需要加锁的原因在于链指针next是final的。但是头指针却不是final的，这是通过getFirst(hash)方法返回，也就是存在 table数组中的值。这使得getFirst(hash)可能返回过时的头结点，例如，当执行get方法时，刚执行完getFirst(hash)之后，另一个线程执行了删除操作并更新头结点，这就导致get方法中返回的头结点不是最新的。这是可以允许，通过对count变量的协调机制，get能读取到几乎最新的数据，虽然可能不是最新的。要得到最新的数据，只有采用完全的同步。

2.) put
该方法也是在持有段锁(锁定当前segment)的情况下执行的，这当然是为了并发的安全，修改数据是不能并发进行的，必须得有个判断是否超限的语句以确保容量不足时能够rehash。首先根据计算得到的散列值定位到segment及该segment中的散列桶中。接着判断是否存在同样一个key的结点，如果存在就直接替换这个结点的值。否则创建一个新的结点并添加到hash链的头部，这时一定要修改modCount和count的值，同样修改count的值一定要放在最后一步。

3）remove
HashEntry中除了value不是final的，其它值都是final的，这意味着不能从hash链的中间或尾部添加或删除节点，因为这需要修改next 引用值，所有的节点的修改只能从头部开始。对于put操作，可以一律添加到Hash链的头部。但是对于remove操作，可能需要从中间删除一个节点，这就需要将要删除节点的前面所有节点整个复制一遍，最后一个节点指向要删除结点的下一个结点。
首先定位到要删除的节点e。如果不存在这个节点就直接返回null，否则就要将e前面的结点复制一遍，尾结点指向e的下一个结点。e后面的结点不需要复制，它们可以重用。

4) size()

每个Segment都有一个count变量，是一个volatile变量。当调用size方法时，首先先尝试2次通过不锁住segment的方式统计各个Segment的count值得总和，如果两次值不同则将锁住整个ConcurrentHashMap然后进行计算。

参见《java并发编程的艺术》P156
http://www.cnblogs.com/ITtangtang/p/3948786.html


## 同步方法和同步代码块的区别是什么
1. 同步方法只能锁定当前对象或class对象， 而同步方法块可以使用其他对象、当前对象及当前对象的class作为锁。
2. 从反编译后的结果看，对于同步块使用了monitorenter和monitorexit指令，而同步方法则是依靠方法上的修饰符ACC_SYNCHRONIZED来完成，但它们的本质都是对一个对象监视器进行获取，而这个获取过程是排他的。
	
### 显示锁ReentrantLock与内置锁synchronized的相同与区别
相同：显示锁与内置锁在加锁和内存上提供的语义相同(互斥访问临界区)
不同：
1.使用方式，内置无需指定释放锁，简化锁操作。显示锁拥有锁获取和释放的可操作性。
2.功能上：显示锁提供了其他很多功能如定时锁等待、可中断锁等待、公平性、尝试非阻塞获取锁、以及实现非结构化的加锁。（一个线程获取不到锁而被阻塞在synchronized之外时，对该线程进行中断操作，此时该线程的中断表示为会被修改，但线程依旧会被阻塞在synchronized上，等待获取锁。）
3.对死锁的处理：内置只能重启，显示可以通过设置超时获取锁来避免
4.性能上：java1.5 显示远超内置，java1.6 显示锁稍微比内置好

7. atomicinteger和Volatile等线程安全操作的关键字的理解和使用

SOF你遇到过哪些情况。

21. 实现多线程的3种方法：Thread与Runable。
**1)继承Tread类，重写run函数**
**2)实现Runnable接口**
**3)实现Callable接口**


22. 线程同步的方法：sychronized、lock、reentrantLock等。

23. 锁的等级：方法锁、对象锁、类锁。
26. ThreadPool用法与优势。

26 Callable和Runnable的区别
27. Concurrent包里的其他东西：ArrayBlockingQueue、CountDownLatch等等。

29. foreach与正常for循环效率对比。
31. 反射的作用于原理。

32. 泛型常用特点，List<String>能否转为List<Object>。

36. 设计模式：单例、工厂、适配器、责任链、观察者等等。
37. JNI的使用。
java的代理是怎么实现的  
35. Java1.7与1.8新特性。
lmbda表达式
Java8新特性
连接池使用使用什么数据结构实现
实现连接池
结束一条 Thread 有什么方法？ interrupt 底层实现有看过吗？线程的状态是怎么样的？如果给你实现会怎么样做？
Java 中有内存泄露吗？是怎么样的情景？为什么不用循环计数？
java都有哪些加锁方式
AIO与BIO的区别
生产者与消费者，手写代码
1. Java创建线程之后，直接调用start()方法和run()的区别
2. 常用的线程池模式以及不同线程池的使用场景
3. newFixedThreadPool此种线程池如果线程数达到最大值后会怎么办，底层原理。
4. 多线程之间通信的同步问题，synchronized锁的是对象，衍伸出和synchronized相关很多的具体问题，例如同一个类不同方法都有synchronized锁，一个对象是否可以同时访问。或者一个类的static构造方法加上synchronized之后的锁的影响。
5. 了解可重入锁的含义，以及ReentrantLock 和synchronized的区别
6. 同步的数据结构，例如concurrentHashMap的源码理解以及内部实现原理，为什么他是同步的且效率高
8. 线程间通信，wait和notify

9. 定时线程的使用
10. 场景：在一个主线程中，要求有大量(很多很多)子线程执行完之后，主线程才执行完成。多种方式，考虑效率。
14. 并发、同步的接口或方法
16. J.U.C下的常见类的使用。 ThreadPool的深入考察； BlockingQueue的使用。（take，poll的区别，put，offer的区别）；原子类的实现。
17. 简单介绍下多线程的情况，从建立一个线程开始。然后怎么控制同步过程，多线程常用的方法和结构
19. 实现多线程有几种方式，多线程同步怎么做，说说几个线程里常用的方法


## 写出生产者消费者模式。

    public class ProducerConsumerPattern {
        public static void main(String args[]){
         BlockingQueue sharedQueue = new LinkedBlockingQueue();
         Thread prodThread = new Thread(new Producer(sharedQueue));
         Thread consThread = new Thread(new Consumer(sharedQueue));
         prodThread.start();
         consThread.start();
        }
    }
    
    //Producer Class in java
    class Producer implements Runnable {
        private final BlockingQueue sharedQueue;
        public Producer(BlockingQueue sharedQueue) {
            this.sharedQueue = sharedQueue;
        }
        @Override
        public void run() {
            for(int i=0; i<10; i++){
                try {
                    System.out.println("Produced: " + i);
                    sharedQueue.put(i);
                } catch (InterruptedException ex) {
                    Logger.getLogger(Producer.class.getName()).log(Level.SEVERE, null, ex);
                }
            }
        }
    }
     
    //Consumer Class in Java
    class Consumer implements Runnable{
        private final BlockingQueue sharedQueue;
        public Consumer (BlockingQueue sharedQueue) {
            this.sharedQueue = sharedQueue;
        }
        @Override
        public void run() {
            while(true){
                try {
                    System.out.println("Consumed: "+ sharedQueue.take());
                } catch (InterruptedException ex) {
                    Logger.getLogger(Consumer.class.getName()).log(Level.SEVERE, null, ex);
                }
            }
        }
    }







