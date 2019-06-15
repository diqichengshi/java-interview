## 1.AQS简单介绍

AQS的全称为(AbstractQueueSynchronizer)，这个类在java.util.concurrent.lock包下面。

AQS是一个用来构建锁和同步器的框架，使用AQS能简单快速的构建出应用广泛的同步器，比如我们提到的reentrantLock、Semaphore，其他诸如ReentranReadWriteLock，SynchronousQueue，FutureTask等等皆是基于AQS的。当然，我们自己也能利用AQS轻松非常轻松容易的构建出符合我们自己需求的同步器。

## 2.AQS原理

在面试中被问到的并发知识的时候，大多都会被问道"请你说一下自己对AQS原理的理解"。下面给大家一个示例供大家参考，面试不是背题，大家一定要加入自己的思想，即使加入不了自己的思想也要保证自己能够通俗的讲出啦而不是背出来。

### 2.1.AQS原理概览

**AQS的核心思想是：如果请求的共享资源空闲，则将请求当前资源的线程设置为有效的工作线程，并将共享资源设置为锁定状态。如果请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程放入队列。**

看个AQS(AbstractQueuedSynchronizer)原理图：

![enter image description here](https://camo.githubusercontent.com/55090fdc22963d41a3e56d5dadb17dfa7ad6379f/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f4a6176612532302545372541382538422545352542412538462545352539312539382545352542462538352545352541342538372545462542432539412545352542392542362545352538462539312545372539462541352545382541462538362545372542332542422545372542422539462545362538302542422545372542422539332f434c482e706e67)

AQS使用一个int成员变量来表示同步状态，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。

```java
private volatile int state;//共享变量，使用volatile修饰保证线程可见性

// 状态信息通过protected类型的getState，setState，compareAndSetState进行操作
//返回同步状态的当前值
protected final int getState() {  
        return state;
}
 // 设置同步状态的值
protected final void setState(int newState) { 
        state = newState;
}
//原子地（CAS操作）将同步状态值设置为给定值update如果当前同步状态的值等于expect（期望值）
protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

### 2.2.AQS对资源的共享方式

AQS定义两种资源共享方式

1.Exclusive(独占)：只有一个线程执行，入ReentrantLock。又可分为公平锁和非公平锁。

2.Share(共享)：多个线程可同时执行，如Semaphore/CountDownLatch。Semaphore、CountDownLatCh、 CyclicBarrier、ReadWriteLock 我们都会在后面讲到。

不同的自定义同步器争抢共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取和释放方式即可，至于具体线程等待队列的维护(如获取资源失败入队/唤醒出队等)，AQS已经在上层帮我们实现好了。

### 2.3.AQS底层使用了模板方法模式

同步器的涉及是基于模板方法模式的，如果需要自定义同步器一般的方式是这样(模板方法模式很经典的一个应用)：

1.使用者继承AbstractQueueSynchronized并重写指定的方法(这些重写方法很简单，无非是对于共享资源state的获取和释放)。

2.将AQS组合在自定义同步组件的实现中，并调用其模板方法，而这模板方法会调用使用者重写的方法。

**AQS使用了模板方法模式，自定义同步器时需要重写下面几个AQS提供的模板方法：**

```java
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryAcquireShared(int)
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
```

默认情况下，每个方法都抛出UnsupportedOperationException。这些方法的实现必须是内部线程安全的，并且通常应该简单而不是阻塞。AQS的其他方法都是final，无法被其他类使用，只有这几个方法可以被其他类使用。

**以reentrantLock为例：state初始化为0，表示为锁定状态。**。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

**再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）**。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS(Compare and Swap)减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

## 3.Semaphore(信号量)-允许多个线程同时访问

**synchronized和ReentrantLock都是一次只允许一个线程访问某个资源，Semaphore(信号量)可以指定多个线程同时访问某个资源。**

实例代码如下：

```java
public class SemaphoreExample1 {
  // 请求的数量
  private static final int threadCount = 550;

  public static void main(String[] args) throws InterruptedException {
    // 创建一个具有固定线程数量的线程池对象（如果这里线程池的线程数量给太少的话你会发现执行的很慢）
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    // 一次只能允许执行的线程数量。
    final Semaphore semaphore = new Semaphore(20);

    for (int i = 0; i < threadCount; i++) {
      final int threadnum = i;
      threadPool.execute(() -> {// Lambda 表达式的运用
        try {
          semaphore.acquire();// 获取一个许可，所以可运行线程数量为20/1=20
          test(threadnum);
          semaphore.release();// 释放一个许可
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }

      });
    }
    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadnum) throws InterruptedException {
    Thread.sleep(1000);// 模拟请求的耗时操作
    System.out.println("threadnum:" + threadnum);
    Thread.sleep(1000);// 模拟请求的耗时操作
  }
}
```

执行acquire()方法阻塞，直到有一个许可证可以获得然后拿走一个许可证；每个release()方法增加了一个许可证，这可能会释放一个阻塞的acquire()方法。然而，其实并没有实际的许可证这个对象，Semaphore只是维持了可获得许可证的数量。Semaphore经常用于限制获取某种资源的线程数量。

当然一次也可以一次拿取和释放多个许可，不过一般没有必要这样做：

除了acquire()方法外，另外一个比较的常用的与之对应的方法是tryAcquire()方法，该方法如果获取不到许可立即返回false。

Semaphore有两种模式：公平模式和非公平模式。

1.公平模式：调用acquire()的顺序就是获取许可证的顺序，遵循FIFO。

2.非公平模式：抢占式。

**Semaphore 对应的两个构造方法如下：**

```java
ublic Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

**这两个构造方法，都必须提供许可的数量，第二个构造方法可以指定是公平模式还是非公平模式，默认非公平模式。**

## 4.CountDownLatch(倒计时器)

CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行。在Java并发中，CountDownLatch的概念是一个常见的面试题，所以一定要确保你很好的了解了它。

### 4.1.CountDownLatch的三种典型用法

1.某一线程在开始运行前等待n个线程执行完毕。个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。

2.实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的 CountDownLatch 对象，将其计数器初始化为 1 ：new CountDownLatch(1) ，多个线程在开始执行任务前首先 coundownlatch.await()，当主线程调用 countDown() 时，计数器变为0，多个线程同时被唤醒。

3.死锁检测：一个非常方便的使用场景是，你可以使用n个线程访问共享资源，在每次测试阶段的线程数目是不同的，并尝试产生死锁。

### 4.2.CountDownLatch的使用实例

```java
public class CountDownLatchExample1 {
  // 请求的数量
  private static final int threadCount = 550;

  public static void main(String[] args) throws InterruptedException {
    // 创建一个具有固定线程数量的线程池对象（如果这里线程池的线程数量给太少的话你会发现执行的很慢）
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    final CountDownLatch countDownLatch = new CountDownLatch(threadCount);
    for (int i = 0; i < threadCount; i++) {
      final int threadnum = i;
      threadPool.execute(() -> {// Lambda 表达式的运用
        try {
          test(threadnum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } finally {
          countDownLatch.countDown();// 表示一个请求已经被完成
        }

      });
    }
    countDownLatch.await();
    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadnum) throws InterruptedException {
    Thread.sleep(1000);// 模拟请求的耗时操作
    System.out.println("threadnum:" + threadnum);
    Thread.sleep(1000);// 模拟请求的耗时操作
  }
}
```

上面的代码中，我们定义了请求的数量为550，当这550个请求被处理完成之后，才会执行System.out.println("finish");。

与CountDownLatch的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用CountDownLatch.await()方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

其他N个对象必须引用闭锁对象，因为它们需要通知CountDownLatch对象，它们已经完成了各自的任务。这种通知机制是通过CountDownLatch.countDown()方法来完成的；每调用一次这个方法，在构造函数中初始化的count的值就减1。所以当N个线程调用这个方法，count()的值等于0，然后主线程就能通过await()，恢复执行自己的任务。

### 4.3.CountDownLatch的不足

CountDownLatch是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当CountDownLatch使用完毕后，它不能再次被使用。

### 4.4.CountDownLatch的常见面试题

解释一下CountDownLatch概念？

CountDownLatch 和CyclicBarrier的不同之处？

给出一些CountDownLatch使用的例子？

CountDownLatch 类中主要的方法？

## 5.CyclicBarrier(循环栅栏)

CyclicBarrier和CountDownLatch非常类似，它也可以实现线程间的技术等待，但是它的功能比CountDownLatch更加复杂和强大。主要应用场景和CountDownLatch类似。

CyclicBarrier的子面意思是可循环使用的屏障。它要做的事情是，让一组线程到达一个屏障(也叫做同步点)的时候被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。CyclicBarrier默认的构造方法是CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用await()方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。

### 5.1.CyclicBarrier的应用场景

CyclicBarrier可以用于多线程计算数据，最后合并计算结果的应用场景。比如我们用一个excel保存了用户所有银行流水，每个Sheet保存了一个账户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个Sheet里的银行流水，都执行完后，得到每个Sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个excel的日均银行流水。

### 5.2.CyclicBarrier的使用示例

```java
public class CyclicBarrierExample2 {
  // 请求的数量
  private static final int threadCount = 550;
  // 需要同步的线程数量
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5);

  public static void main(String[] args) throws InterruptedException {
    // 创建线程池
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadnum:" + threadnum + "is ready");
    try {
      /**等待60秒，保证子线程完全执行结束*/  
      cyclicBarrier.await(60, TimeUnit.SECONDS);
    } catch (Exception e) {
      System.out.println("-----CyclicBarrierException------");
    }
    System.out.println("threadnum:" + threadnum + "is finish");
  }

}
```

### 5.3.CyclicBarrier和CountDownLatch的区别

CountDownLatch是计数器，只能使用一次，而CyclicBarrier的计数器提供reset功能，可以多次使用。但是我不认为它们之间的区别仅仅就是那么简单的一点。

对于CountDownLatch来说，重点是"一个线程(多个线程)等待"，而其他的N个线程在完成"某件事情"之后，可以终止，也可以等待。而对于CyclicBarrier，重点是多个线程，在任意一个线程没有完成，所有的线程都必须等待。

CountDownLatch是计数器，线程完成一个记录一个，只不过计数不是递增而是递减，而CyclicBarrier更像是一个阀门，需要所有线程都达到，阀门才能打开，然后继续执行。

CyclicBarrier和CountDownLatch的区别：

1.CyclicBarrier加计数方式。CountDownLatch是减计数方式。

2.CyclicBarrier计数达到指定值时释放所有等待线程。CountDownLatch计算为0时释放所有等待的线程。

3.CyclicBarrier计数达到指定值的时候，计数置为0重新开始。CountDownLatch计数为0时，无法重置。

4.CyclicBarrier调用await()方法技术加1，若加1后的值不等于构造方法的值，则线程阻塞。CountDownLatch调用countDown()方法计数减1，调用await()方法只进行阻塞，对计数没任何影响。

5.CyclicBarrier可重复利用。CountDownLatch不可重复利用。

## 6.ReentrantLock和ReentrantReadWriteLock

ReentrantLock 和 synchronized 的区别在上面已经讲过了这里就不多做讲解。另外，需要注意的是：读写锁 ReentrantReadWriteLock 可以保证多个线程可以同时读，所以在读操作远大于写操作的时候，读写锁就非常有用了