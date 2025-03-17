# 为什么要引入CAS

- 1）解决传统锁的性能问题：

  - synchronized等锁机制在多线程竞争时会导致线程阻塞，上下文切换，以此产生性能损耗，尤其是高并发场景下可能会引发吞吐量下降

  无锁优化：CAS通过自旋（循环重试）实现无锁操作，避免了线程阻塞和唤醒的开销，适用于低竞争场景下的高并发

- 2）保证原子操作：

  - 非原子操作存在线程安全问题，简单的操作（如：i++）在多线程环境中并非是原子操作，需要拆分为"读取值-修改值-写入值"等步骤，可能会导致数据不一致

  原子性保障：如果多个线程同时尝试修改同一变量，只有一个线程的CAS操作会成功，其他线程需要重试或放弃

  - CAS通过硬件级原子指令（如：`lock cmpxchg`指令实现缓存锁定）确保【比较-交换值】过程的原子性，可直接用于解决多线程对共享变量的原子更新问题
  - 内存屏障：`lock cmpxchg`隐式包含`StoreLoad`屏障，强制刷新写缓冲区，增加总线的通信量

- 3）支持乐观锁：

  - 悲观锁的局限性：传统锁假设并发冲突频繁，通过独占资源来避免冲突，但这会增加等待时间

  CAS采用乐观锁的机制，假设并发冲突较少，仅在更新时检查变量是否被修改，失败后通过自旋重试，可减少线程阻塞

- 4）硬件性能优势：
  - 底层支持：CAS依赖于CPU原子指令（如：`cmpxchg`）实现，由JVM通过`Unsafe`类封装为本地方法，执行效率较高
  - 轻量级同步：相较于重量级锁，CAS操作无需进入内核态，可减少系统调用的开销
- 5）优化并发数据结构：
  - CAS是`java.util.concurrent`包中的原子类（如`AtomicInteger`）、无锁队列（如`ConcurrentLinkedQueue`）以及AQS框架的核心实现基础
  - 如：`AtomicInteger`对象调用`incrementAndGet`方法会通过CAS来保证线l程安全的原子递增，避免锁竞争
- 6）适用场景扩展：
  - 锁优化：对于`synchronized`的优化（锁升级），CAS可实现锁状态的快速切换，减少直接使用重量级锁的概率
  - 避免ABA问题：结合`AtomicStampedReference`等工具，可通过版本号机制扩展CAS适用性，解决对象引用更新时的中间状态问题

# 什么是CAS

- CAS（Compare And Swap）是硬件级别的原子操作，会比较内存中的某个值是否为预期值，如果是，则更新为新值，否则不做修改
- 工作原理：
  - 比较（Compare）：CAS会检查内存中的某个值是否和预期值相等
  - 交换（Swap）：如果相等，则将内存中的值更新为新值
  - 失败重试：如果不相等，则说明有其他线程已经修改了该值，CAS操作失败，一般会重新尝试修改值，直到成功

## CAS的优缺点

优点：

- 无锁并发：CAS操作不使用锁，不会导致线程阻塞，提高了系统的并发和性能
- 原子性：CAS操作是原子的，不可分割的，保证了线程安全

缺点：

- ABA问题：CAS操作中，如果一个变量值从A变为B，又变回为A，CAS无法检测到这种变化，可能会导致错误
- 自旋开销：CAS操作通过自旋实现，可能会导致CPU资源浪费，尤其在高并发的场景下
- 仅适用于对单变量的控制：CAS操作仅适用于对单个变量的更新，而如果要保证多个变量的原子更新和复杂逻辑，CAS需要结合循环重试、锁等其他机制

## ABA问题

- ABA问题：变量值从A变为B，再从B变为A，CAS操作无法检测到这种变化，解决ABA问题的常见方法是引入版本号或时间戳，每次更新变量时，同时更新版本号或时间戳，从而检测变量的变化

- Java中的`AtomicStampedReference`提供了版本号的解决方案，其内部提供一个Pair对象封装了引用和版本号，通过volatile保证了`Pair`对象的可见性

  ```java
  public class AtomicStampedReference<V> {
  
      private static class Pair<T> {
          final T reference;
          final int stamp;
          private Pair(T reference, int stamp) {
              this.reference = reference;
              this.stamp = stamp;
          }
          static <T> Pair<T> of(T reference, int stamp) {
              return new Pair<T>(reference, stamp);
          }
      }
  
      private volatile Pair<V> pair;
  
      /**
       * Creates a new {@code AtomicStampedReference} with the given
       * initial values.
       *
       * @param initialRef the initial reference
       * @param initialStamp the initial stamp
       */
      public AtomicStampedReference(V initialRef, int initialStamp) {
          pair = Pair.of(initialRef, initialStamp);
      }
      // ...
  }    
  ```

- 在内部CAS中，添加了版本号的对比：

  ![image-20250316153106397](C:\Users\xyl\AppData\Roaming\Typora\typora-user-images\image-20250316153106397.png)

- Java还提供了`AtomicMarkableReference`类，其原理和`AtomicStampedReference`类似，区别在于AtomicMarkableReference是用boolean类型的mark来标识数据是否被修改，而AtomicStampedReference则用int类型的stamp来标识数据的版本（数据被修改的次数）

![image-20250316153404557](C:\Users\xyl\AppData\Roaming\Typora\typora-user-images\image-20250316153404557.png)



# CAS在Java中的实现

- 在Java中，CAS操作由`sun.misc.Unsafe`类提供，具体是通过JNI（Java Native Interface）来调用这些底层的硬件指令来实现CAS操作，从而以保障CAS操作的原子性
- 在Java中，可以使用并发包中的原子类（如：AtomicInteger、AtomicLong等），这些类封装了CAS操作，提供了线程安全的原子操作
- 下面具体来说下AtomicInteger和LongAdder

## AtomicInteger

- 在JDK1.5引入（Lock、ReentrantLock等锁的实现也是在JDK1.5引入，synchronized在JDK1.0引入）

### 实现原理

### 使用场景

### 瓶颈







## LongAdder

- LongAdder在JDK1.8引入

- volatile可解决多线程内存不可见的问题，对于一写多读的场景，volatile可用于解决变量同步问题，但如果是多写，则无法解决线程安全问题，可通过原子类来保证线程安全

  - 如果是count++操作，使用如下类实现：

    ```java
    AtomicInteger count = new AtomicInteger(); 
    count.addAndGet(1);
    ```

  - 如果是JDK8，则推荐使用LongAdder，LongAdder通过减少乐观锁的重试次数，来获得比AtomicInteger更好的性能

    ```java
    LongAdder num = new LongAdder();
    num.add(2);
    ```

### 设计思想

#### 分段降低CAS失败的频次

- 在设计思想上，LongAdder通过"分段"（分散热点）来降低CAS失败的频次，将value值的更新操作分散到一个数组中，不同线程会命中到数组的不同槽中，各个线程只对自己槽中的value值进行CAS操作，如此热点就被分散了，冲突的概率就会减小很多
- LongAdder有一个`volatile long`修饰的全局变量`base`，当并发度不高时，都是通过CAS来直接操作base，如果CAS失败，则针对LongAdder中的`Cell[]`数组中的Cell进行CAS操作，如此以减少失败的概率

#### @Contended消除伪共享

- 在 `LongAdder` 的父类 `Striped64` 中存在一个 `volatile Cell[] cells;` 数组，其长度是**2 的幂次方**，每个`Cell`都使用 `@Contended` 注解进行修饰，而`@Contended`注解可以进行**缓存行填充**，从而解决**伪共享问题**。伪共享会导致缓存行失效，缓存一致性开销变大

- 伪共享：多个线程同时读写同一个缓存行的不同变量时导致的CPU缓存失效，尽管这些变量之间没有任何关系，但由于在主内存中邻近，存在于同一个缓存行中，变量之间的相互覆盖会导致缓存频繁未命中，进而引发性能下降
- 解决伪共享的方案：一般会使用直接填充，只需保证不同线程的变量存在于不同的缓存行即可，使用多余的字节来填充即可做到，如此就不会出现伪共享问题

#### 惰性求值

- `LongAdder`只有在使用`longValue()`获取当前累加值时才会真正的去结算计数的数据，`longValue()`方法底层就是调用`sum()`方法，对`base`和`Cell数组`的数据累加然后返回，做到数据写入和读取分离

- 而`AtomicLong`使用`incrementAndGet()`每次都会返回`long`类型的计数值，每次递增后还会伴随着数据返回，增加了额外的开销

### 实现原理

### 使用场景



# CAS总线风暴

- CAS总线风暴是一种在高并发场景下，因频繁使用CAS操作和volatile变量，导致**CPU缓存一致性协议**（如：MESI）的通信流量激增，进而引发系统性能下降的现象，其本质上是**硬件层面的缓存行竞争**和**总线资源争用的综合结果**

## 核心原因

- 1）CAS操作的硬件机制保证（前面为什么引入CAS中已说明）

- 2）volatile变量的可见性机制

  - 写操作：JVM会在volatile写操作前插入StoreStore屏障（防止之前的普通写和当前volatile写重排序），在写操作后插入StoreLoad屏障（防止当前volatile写与之后的读操作重排序）
  - 读操作：JVM会在volatile读操作前插入LoadLoad屏障（防止之后的普通读与当前volatile读重排序），在读操作后插入LoadStore屏障（防止之后的普通写与当前volatile读重排序）

- 3）缓存行竞争（Cache Line Contention）

  - 伪共享：多个变量位于同一缓存行，不同线程独立修改这些变量时，会让缓存行失效

- 4）总线嗅探和一致性协议开销：

  - 嗅探机制：CPU需要每时每刻监听总线来维护缓存一致性，高频的CAS和volatile操作会生成大量的嗅探请求
  - 协议优化限制：尽管MESI协议通过批量失效和目录协议来减少流量，但在极端高并发场景下仍然可能会超过总线的承载能力

  补充：

  - 为实现CPU、缓存的一致性，需要引入写传播和事务的串行化来保证

    写传播（Write Propagation）：某个CPU核心中的Cache在更新数据时，必须要传播到其他CPU核心的Cache

    事务的串行化（Transaction Serialization）：某个CPU核心对数据的操作顺序，必须在其他核心看来，其顺序是一致的

  - 为实现写传播和事务的串行化，需要分别引入总线嗅探和MESI协议

    总线嗅探：CPU每时每刻监听总线上的一切活动，不管其他CPU核心是否缓存相同的数据，都要发出一个广播事件，但这会加重总线的负载

    MESI协议为保证事务的串行化，通过【状态机】机制降低了总线带宽，用Modified、Exclusive、Shared、Invalidate来标识Cache Line可能处于的四个状态，这在一定程度上减少了总线带宽压力

## 解决方案

1）内存布局优化：

- 使用`@Contended`注解（需`-XX:-RestrictContended`）

  ```java
  // 手动填充可能因对象头（12字节）和压缩指针失效
  class IneffectivePadding {
      private long head;          // 12字节对象头 + 4字节（压缩指针）
      private long p1, p2, p3;    // 24字节填充（实际需填充7个long）
      private volatile long data;
  }
  ```

  

- 手动填充确保跨缓存行（结合`jol`工具分析对象布局）

2）分散竞争：

- 使用分片计数器（如`LongAdder`）
- 将全局变量拆分为线程本地变量（Thread-Local）

3）环境适配

- 在NUMA系统中绑定线程至本地节点（`taskset`或`numactl`）

4）替代同步机制

- 在极端高竞争下使用锁（`ReentrantLock`）或`Thread.yield()`降低自旋强度



## 如何验证‌

- 1）**性能监测工具**

  - **Intel VTune**‌：分析缓存行竞争、总线锁定事件及NUMA跨节点流量

  - Perf：监测总线周期（`bus-cycles`）、缓存失效（`cache-misses`）及指令分布

    ```bash
    perf stat -e bus-cycles,cache-misses,instructions ./my_program  
    ```

- 2）**对象布局分析**

  ‌**JOL（Java Object Layout）**‌：精确计算对象内存布局，验证填充效果。

  ```java
  System.out.println(ClassLayout.parseClass(MyClass.class).toPrintable());
  ```

- 3）**微基准测试**

  - ‌**JMH（Java Microbenchmark Harness）**‌：对比CAS、锁及分片计数器的吞吐量差异

    ```java
    @Benchmark  
    public void testCAS(Blackhole bh) {  
        bh.consume(atomicInt.compareAndSet(0, 1));  
    }  
    ```











