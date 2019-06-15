## 1.synchronized关键字

### 1.1.说一说自己对于synchronized关键字的理解

synchronized关键字解决的是多个线程之间访问资源的同步性，synchronized关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。

另外，在Java早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁(monitor)是依赖于底层的操作系统的Mutex Lock来实现的，Java的线程是映射到操作系统的原生线程之上的。如果要挂起或者唤醒一个线程，都需要操作系统来帮忙完成，而操作系统实现线程之间的切换需要从用户状态转换到内核态，这个状态之间的切换需要相对较长的时间，时间成本相对较高，这也是为什么早期synchronized效率比较低的原因。幸运的是在Java6以后官方从JVM方面对synchronized做了较大优化，所以现在的synchronized锁效率也优化得很不错了。JDK1.6对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗话、偏向锁、轻量级锁等技术来减少操作的开销。

### 1.2.说说自己是怎么使用synchronized关键字的

synchronized关键字最主要的三种使用方式：

1.**修饰实例方法：**作用于当前对象实例加锁，进入同步代码前要获得当前对象实例的锁。

2.**修饰静态方法：**也就是给当前类加锁，会作用于类的所有实例对象，因为静态成员不属于任何一个实例对象是类成员(static表明这是该类的一个静态资源，不管new了多少个对象，只有一份)。所以如果一个线程A调用一个实例对象的非静态synchronize方法，而线程B需要调用这个实例对象所属类的静态synchronized方法，是允许的，不会发生，**因为访问静态synchronized方法占用的锁是当前类的锁，而访问非静态synchronized方法占用的锁是当前实例对象锁**。

3.**修饰代码块：**指定加锁对象，对给定对象加锁，进入同步代码块前要获得给定对象的锁。

**总结：**synchronized关键字加到static静态方法和synchronized(class)代码块上都是给Class类上锁。synchronized关键字加到静态方法上是给实例对象上锁。尽量不要使用synchronized(String s)在JVM中，字符串常量池具有缓存功能。

下面我以一个常见的面试题为例讲解一下synchronized关键字的具体应用。

面试中面试官经常会说"单例模式了解吗？来帮我手写以下！给我解释一下双重检验锁方式实现单例模式的原理"

**双重检验锁实现对象单例(线程安全)**

```java
public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

另外，需要注意uniqueInstance才用volatile关键字修饰也是很有必要。

uniqueInstance才有volatile关键字修饰也是很有必要的，uniqueInstance = new Singleton()；这段代码是分三步执行的。

1.为uniqueInstance分配内存空间。

2.初始化uniqueInstance。

3.将uniqueInstance指向分配的内存地址。

但是由于JVM具有指令重排的特性，执行顺序有可能编程1->3->2。指令重排在单线程环境下不会出现问题，但是在多线程环境下会导致一个线程获得还没有初始化的实例。例如，线程T1执行了1和3，线程T2调用getInstance()方法后发现uniqueInstance不为空，因此返回uniqueInstance，但此时uniqueInstance还未被初始化。

使用volatile可以禁止JVM的指令重排，保证在多线程环境下也能正常运行。

### 1.3.讲一下synchronized关键字的底层原理

**synchronized关键字底层原理属于JVM层面。**

#### 1.3.1.synchronized同步代码块(静态方法或实例)的情况

```java
public class SynchronizedDemo {
	public void method() {
		synchronized (this) {
			System.out.println("synchronized 代码块");
		}
	}
}
```

通过JDK自带的javap命令查看SynchronizedDemo类的相关字节码信息：首先切换到类的对应目录执行javac SynchronizedDemo.java命令生产编译后的.class文件，然后执行javap -c -s -v -l SynchronizedDemo.class。

![synchronized å³é®å­åç](https://camo.githubusercontent.com/8b5d297bde46c94ba37f6a050fcbb5c057b7f319/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31302f32362f313636616464363136613239326263663f773d39313726683d36333326663d706e6726733d3231383633)

从上面我们可以看出：

**synchronized同步代码块的实现使用的是monitorenter和monitorexit指令，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置。**当执行monitorenter指令时，线程试图获取锁也就是获取**monitor**(monitor对象存在于每个Java对象的对象头，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因)的持有权。当计数器为0则可以成功获取，获取后将锁计数器设为1也就是加1。相应的在执行monitorexit指令后，将锁计数器设为0，表明锁被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放。

#### 1.3.2.synchronized修饰方法的情况

```java
public class SynchronizedDemo2 {
	public synchronized void method() {
		System.out.println("synchronized 方法");
	}
}
```

![synchronized å³é®å­åç](https://camo.githubusercontent.com/38d5caa094b8646f676804e394fc9b6a0bd21cb8/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31302f32362f313636616464363136396663323036643f773d38373526683d34323126663d706e6726733d3136313134)

synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取而代之的确实是ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法，**JVM通过ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。**

### 1.4.说说JDK1.6之后synchronized关键字底层做了哪些优化

JDK1.6对锁的实现引入了大量的优化，如偏向锁、轻量级锁、自旋锁、适应性自旋锁、锁消除、锁粗化等技术来减少锁操作的开销。

锁主要存在四种状态，依次是无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态，他们会随着竞争的激烈而逐渐升级。注意锁可以升级不可降级，这种策略是为了提高获得锁的释放锁的效率。

关于这几种优化的详细信息可以查看笔主的这篇文章：<https://gitee.com/SnailClimb/JavaGuide/blob/master/docs/java/Multithread/synchronized.md>

### 1.5.谈谈synchronized和ReentrantLock的区别

**1.两者都是可重入锁**

两者都是可重入锁。可重入锁的概念是：自己可以再次获取自己的内部锁。比如一个线程获得了某个对象的锁，此时这个对象锁还没有释放，当其再次想要获取这个对象的锁的时候还是可以获取的，如果不可重入锁的话，就会造成死锁。同一个线程每次获取锁，锁的计数器都自增1，所以要等到锁的计数器降为0的时候才能释放锁。

**2.synchronized依赖于JVM而ReentrantLock依赖于API**

synchronized是依赖于JVM实现的，前面我们也讲到了虚拟机团队在JDK1.6后为synchronized关键字进行了很多优化，但是这些优化都是虚拟机层面实现的，并没有直接暴漏给我们。ReentrantLock是JDK层面实现的(也就是API层面，需要lock()和unlock()方法配合try/finally语句块来完成)，所以我们可以通过查看它的源代码，来看他是如何实现的。

**3.ReentrantLock比synchronized增加了一些高级功能**

相比synchronized，ReentrantLock增加了一些高级功能。主要有三点：**1.等待可中断；2.可实现公平锁；3.可实现选择性通知(锁可以绑定多个条件)。**

1.**ReentrantLock提供了一种能够中断等待锁的线程的机制**，通过lock.lockInterruptibly()来实现这个机制。也就是锁正在等待的线程可以选择放弃等待。改为处理其他事情。

2.**ReentrantLock可以指定是公平锁还是非公平锁。而synchronized只能是非公平锁。所谓公平锁就是先等待的线程先获得锁**。ReentrantLock默认情况是非公平的，可以通过ReentrantLock类的ReentrantLock(boolean fair)构造方法来指定是否是公平的。

3.synchronized关键字与wait()和notify()/notifyAll()方法想结合可以实现等待通知机制，ReentrantLock类当然也可以实现，但是需要借助Condition接口与newCondition()方法。**线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。在使用notify()/notifyAll()方法进行通知时，被通知的线程由JVM选择的，用ReentrantLock结合Condition实例可以实现"选择性通知"，**这个功能非常重要，而且是Condition接口默认提供的。而synchronized关键字就相当于整个Lock对象中只有一个Condition实例，所有的线程都注册在它一个身上。如果执行notifyAll()方法就会通知所有处于等待的线程就会造成很大的效率问题，而Condition实例的signalAll()方法 只会唤醒注册在该Condition实例中的所有等待线程。

## 2.volatile关键字

### 2.1.讲一下Java内存模型

在当前的Java模型下，线程把变量保存**本地内存**(比如机器的寄存器)中，而不是直接在主存中进行读写。这就可能造成一个线程A在主存中修改了一个变量的值，而另外一个线程还继续使用它在寄存器中变量值的拷贝，造成**数据的不一致**。

![æ°æ®çä¸ä¸è´](https://camo.githubusercontent.com/5dabf65a6f750c5b767b2e221aaa27eab47d4fa3/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31302f33302f313636633436656465343432336261323f773d32373326683d31363626663d6a70656726733d37323638)

要解决这个问题，就需要把变量声明为volatile，这就指示JVM，这个变量是不稳定的，每次使用它都到主存中进行读写。

说白了，volatile关键字的主要作用就是保证变量的可见性，然后还有一个作用就是防止指令重排序。

### 2.2.说说synchronized关键字和volatile关键字的区别

synchronized关键字和volatile关键字比较

1.**volatile关键字是线程同步的轻量级实现，所以volatile性能肯定比synchronized关键字要好。但是volatile关键字只能用于变量而synchronized关键字可以修饰方法以及代码块**。synchronized关键字在Java1.6之后进行了主要包括为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁以及其它各种优化之后执行效率有个显著提升，**实际开发中使用synchronized关键字的场景还是更多一些**。

2.**多线程访问volatile关键字不会发生阻塞，而synchronized关键字可能会发生阻塞。**

3.**多线程关键字能保证数据的可见性，但不能保证数据的原子性。synchronized关键字两者都能保证。**

4.**volatile关键字主要用于解决变量在多个线程之间的可见性，而synchronized关键字解决的是多个线程之间访问资源的同步性。**

## 3.ThreadLocal

### 3.1.ThreadLocal简介

通常情况下，我们创建的变量是可以被任意一个线程访问并修改的。**如果想实现每个线程都有自己的专属本地变量该如何解决呢？**ThreadLocal类正是为了解决这样的问题。**ThreadLocal类主要解决的就是让每个线程绑定自己的值，可以把ThreadLocal类形象的比喻称存放数据的盒子，盒子中可以存储每个线程的私有数据。**

**如果你创建了一个ThreadLocal变量，那么访问这个变量的每个线程都会有这个变量的本地副本，这也是ThreadLocal变量名的由来。他们可以使用get()和set()方法来获取默认值或将此值更改为当前线程所存的副本的值，从而避免了线程安全问题。**

再举个简单的例子：

比如有两个人去宝屋收集宝物，这两个共用一个袋子的话肯定会产生争执，但是给他们两个人每个人分配一个袋子的话就不会出现这样的问题。如果把这两个人比作线程的话，那么ThreadLocal就是用来这两个线程竞争的。

### 3.2.ThreadLocal示例

相信看了上面的解释，大家已经搞懂 ThreadLocal 类是个什么东西了。

```java
import java.text.SimpleDateFormat;
import java.util.Random;

public class ThreadLocalExample implements Runnable{

     // SimpleDateFormat 不是线程安全的，所以每个线程都要有自己独立的副本
    private static final ThreadLocal<SimpleDateFormat> formatter
        =ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyyMMdd HHmm"));

    public static void main(String[] args) throws InterruptedException {
        ThreadLocalExample obj = new ThreadLocalExample();
        for(int i=0 ; i<10; i++){
            Thread t = new Thread(obj, ""+i);
            Thread.sleep(new Random().nextInt(1000));
            t.start();
        }
    }

    @Override
    public void run() {
        System.out.println("Thread Name= "+Thread.currentThread().getName()
                           +" default Formatter = "+formatter.get().toPattern());
        try {
            Thread.sleep(new Random().nextInt(1000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //formatter pattern is changed here by thread, 
        //but it won't reflect to other threads
        formatter.set(new SimpleDateFormat());

        System.out.println("Thread Name= "+Thread.currentThread().getName()
                           +" formatter = "+formatter.get().toPattern());
    }

}
```

  

### 3.3.ThreadLocal原理

从Thread类源代码入手。

```java
public class Thread implements Runnable {
 ......
//与此线程有关的ThreadLocal值。由ThreadLocal类维护
ThreadLocal.ThreadLocalMap threadLocals = null;

//与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
 ......
}
```

从上面Thread类源代码可以看出Thread类中有一个threadLocals和一个inheritableThreadLocals变量，它们都是

ThreadLocalMap类型的变量，我们可以把ThreadLocalMap理解为ThreadLocal类实现的定制化的HashMap。默认情况下这两个变量都是null，只有当前线程调用 ThreadLocal类的 set或get方法时才创建它们，实际上调用这两个方法的时候，我们调用的是ThreadLocalMap类对应的 get()、set() 方法。

ThreadLocal类的set()方法

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

通过上面这些内容，我们足以通过猜测得出结论：最终的变量是放在当前线程的ThreadLocalMap中，并不是存放在ThreadLocal上，ThreadLocal可以理解为ThreadLocalMap的封装，传递了变量值。

每个Thread中都具备一个ThreadLocalMap，而ThreadLocalMap可以存储在ThreadLocal为key的键值对。这就解释了为什么每个线程访问同一个ThreadLocal，得到的确是不同的数值。另外，ThreadLocal是map结构是为了让每个线程可以关联多个ThreadLocal变量。

### 3.4.ThreadLocal内存泄漏问题

ThreadLocalMap中使用的key为ThreadLocal的弱引用，而value是强引用，所以，如果ThreadLocal没有被外部强引用的话，在垃圾回收的时候key会被清理掉，而value不会被清理掉。这样一来，ThreadLocalMap中就会出现key为null的Entry。假设我们不做任何措施的话，value永远不会被GC回收，这个时候就可能会产生内存泄漏。ThreadLocalMap实现中已经考虑了这种情况，在调用 set()、get()、remove() 方法的时候，会清理掉 key 为 null 的记录。使用完 ThreadLocal方法后 最好手动调用remove()方法。

## 4.线程池

### 4.1.为什么要用线程池

线程池提供了一种限制和管理资源(包括执行一个任务)。每个线程池还维护了一些基本统计信息，例如已完成任务的数量。

这里借用《Java并发编程的艺术》提到的来说一下使用线程池的好处：

1.**降低资源消耗：**通过重复利用已创建的线程降低线程创建和销毁造成的消耗。

2.**提高响应速度：**当任务到达时，任务可以不需要等到线程创建就立即执行。

3.**提高线程的可管理性：**线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

### 4.2.实现Runnanle接口和Callable接口的区别

如果想让线程池执行任务的话需要实现Runnable接口或Callable接口。Runnable接口或Callable接口实现类都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。两者的区别在于Runnable接口不会返回结果但是Callable接口可以返回结果。

### 4.3.执行execute()和submit()方法的区别

1.**execute()方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否。**

2.**submit()用于提交需要返回值的任务。线程池会返回一个Future类型的对象，通过这个Future对象可以判断任务是否执行成功**，并且可以通过future.get()方法来获取返回值，get()方法会阻塞当前线程直到任务完成。

### 4.4.如何创建线程池

《阿里巴巴Java开发手册》中强制线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

1.FixedThreadPool和SingleThreadExecutor：允许请求的队列长度为Integer.MAX_VALUE，可能堆积大量的请求，从而导致OOM。

2.CacheThreadPool和ScheduledThreadPool：允许创建的线程数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。

## 5.Atomic原子类

### 5.1.介绍一下Atomic原子类

Atomic 翻译成中文是原子的意思。在化学上，我们知道原子是构成一般物质的最小单位，在化学反应中是不可分割的。在我们这里 Atomic 是指一个操作是不可中断的。即使是在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程干扰。

所以，所谓原子类说简单点就是具有原子/原子操作特征的类。

### 5.2.JUC包下的原子类是哪4类

**基本类型**

使用原子的方式更新基本类型

- AtomicInteger：整形原子类
- AtomicLong：长整型原子类
- AtomicBoolean：布尔型原子类

**数组类型**

使用原子的方式更新数组里的某个元素

- AtomicIntegerArray：整形数组原子类
- AtomicLongArray：长整形数组原子类
- AtomicReferenceArray：引用类型数组原子类

**引用类型**

- AtomicReference：引用类型原子类
- AtomicStampedReference：原子更新引用类型里的字段原子类
- AtomicMarkableReference ：原子更新带有标记位的引用类型

**对象的属性修改类型**

- AtomicIntegerFieldUpdater：原子更新整形字段的更新器
- AtomicLongFieldUpdater：原子更新长整形字段的更新器
- AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题

### 5.3.讲讲AtomicInteger的使用

```java
class AtomicIntegerTest {
      private AtomicInteger count = new AtomicInteger();
      //使用AtomicInteger之后，不需要对该方法加锁，也可以实现线程安全。
      public void increment() {
                count.incrementAndGet();
      }
     
      public int getCount() {
               return count.get();
      }
}
```

### 5.4.介绍AtomicInteger的原理

AtomicInteger 线程安全原理简单分析

AtomicInteger 类的部分源码：

```java
// setup to use Unsafe.compareAndSwapInt for updates（更新操作时提供“比较并替换”的作用）
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

private volatile int value;
```

AtomicInteger类主要利用CAS(compare and swap)+volatile+native方法来保证原子操作，从而避免了synchronized的高开销，执行效率大为提升。

CAS的原理是拿期望的值和原来的一个值作比较，如果相同则更新成新的值。Unsafe.objectFiledOffset()方法是一个本地方法，这个方法是用来拿到"原来的值"的内存地址，返回值是valueOffset。另外value是一个volatile变量，在内存中可见，因此JVM可以保证任何时候任何线程总能拿到该变量的最新值。

## 6.AQS

### 6.1.AQS介绍

AQS的全称为(AbstractQueuedSynchronizer)，这个类在java.util.concurent.locks包下面。

AQS是一个用来构建锁和同步器的框架，使用AQS能简单高效的构造出应用广泛的同步器，比如我们提到的ReentrantLock，Semaphore，其他的诸如ReentrantReadWriteLock，SynchoronousQueue，FutureTask等等皆是基于AQS的。当然，我们自己也能利用AQS非常轻松容易的构建出符合我们需求的同步器。

### 6.2.AQS原理分析

在面试中被问到并发知识的时候，大多都会被问到"请你说一下自己对AQS的理解"。下面给大家一个示例供大家参考，面试不是背题，大家一定要有自己的思考。保证自己能够通俗的讲出来。

#### 6.2.1.AQS原理概览

**AQS的核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。**

CLH队列是一个虚拟的双向队列(虚拟的双向队列即不存在队列实例，仅存在节点之间的关联关系)。AQS是将每个请求共享资源的线程封装成一个CLH锁队列的一个结点(Node)来实现锁的分配。

看个AQS(AbstractQueuedSynchronizer)原理图：

![enter image description here](https://camo.githubusercontent.com/0c80521a400e6b110fd860b971fc8d9e6f7bf10d/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31302f33302f313636633462626534613963356165373f773d38353226683d34303126663d706e6726733d3231373937)

AQS使用一个int成员变量来表示同步状态，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。

状态信息通过protected类型的getState，setState，compareAndSetState进行操作

```java
private volatile int state;//共享变量，使用volatile修饰保证线程可见性

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

#### 6.2.2.AQS对资源的共享方式

AQS定义两种资源共享方式

1.Exclusive(独占)：只有一个线程能执行，如ReentrantLock。又可分为公平锁和非公平锁：

​	公平锁：先到先得。

​	非公平锁：无视队列顺序去抢锁。

2.Share(共享)：多个线程可同时执行。

ReentrantReadWriteLock可以看作组合式，因为ReentrantReadWriteLock也就是读写锁允许多个线程同时对同一资源进行读。

不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护，AQS已经在顶层实现好了。

#### 6.2.3.AQS底层使用了模板方法模式

同步器的设计是基于模板方法模式的，如果需要自定义同步器一般的方式是这样：

1.使用者继承AQS指定的方法并重写指定的方法名(这些重写方法很简单，无非是对于共享资源state的获取和释放)。

2.将AQS组合在自定义同步组件的实现中，并调用其模板方法，而这些模板会调用使用者重写的方法。

这和我们以往通过实现接口的方式有很大区别，这是模板方法模式很经典的一个运用。

**AQS使用了模板方法模式，自定义同步器时需要重写下面几个AQS提供的模板方法：**

```java
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
```

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占锁并将state+1。此时，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0(即释放锁)为止，其他线程才有机会获取该锁。当然，释放锁之前，A线程是可以重复获取此锁的(state会累加)，这就是可重入的概念。但要注意，获取多少次锁就要释放多少次，这样才能保证state是能回到零态的。

再以CountDownLatch为例，任务分N个子线程去执行，state也初始化为N(注意N要与线程个数一致)。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS(compare and swap)减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主线程就会从await()函数返回，继续后续动作。

一般来说，自定义同步器要么是独占方法，要么是共享模式，它们只需要实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。

### 6.3. AQS 组件总结

- **Semaphore(信号量)-允许多个线程同时访问：** synchronized 和 ReentrantLock 都是一次只允许一个线程访问某个资源，Semaphore(信号量)可以指定多个线程同时访问某个资源。
- **CountDownLatch （倒计时器）：** CountDownLatch是一个同步工具类，用来协调多个线程之间的同步。这个工具通常用来控制线程等待，它可以让某一个线程等待直到倒计时结束，再开始执行。
- **CyclicBarrier(循环栅栏)：** CyclicBarrier 和 CountDownLatch 非常类似，它也可以实现线程间的技术等待，但是它的功能比 CountDownLatch 更加复杂和强大。主要应用场景和 CountDownLatch 类似。CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。CyclicBarrier默认的构造方法是 CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用await()方法告诉 CyclicBarrier 我已经到达了屏障，然后当前线程被阻塞。

## 7.参考

- 《深入理解 Java 虚拟机》
- 《实战 Java 高并发程序设计》
- 《Java并发编程的艺术》
- <http://www.cnblogs.com/waterystone/p/4920797.html>
- <https://www.cnblogs.com/chengxiao/archive/2017/07/24/7141160.html>
- <https://www.journaldev.com/1076/java-threadlocal-example>

### 