# 理解Java的强引用、软引用、弱引用和虚引用

## 1. Reference 源码分析

### 1.1. Reference 的简介和分类

![](./img/16576be9ee015804.png)

`Reference` 是所有引用对象的基类, 这个类定义了所有引用对象的通用操作. 

由于 `Reference` 实例是由 JVM 创建, 所以自行继承 `Reference` 实现自定义的引用类型是无意义的, 但是可以继承已经存在的引用类型, 如 `SoftReference` 等.


在 JDK1.2 之前, Java 中引用的定义是十分传统: 如果 reference 类型存储的数值代表的是另一块内存的起始地址, 就称这块内存代表着一个引用. 在这种定义之下, 一个对象只有被引用和没有被引用两种状态.

实际上, 我们更希望存在这样的一类对象: 当内存空间还足够的时候, 这些对象能够保留在内存空间中; 如果当内存空间在进行了垃圾收集之后还是非常紧张, 则可以抛弃这些对象. 基于这种特性, 可以满足很多系统缓存功能的使用场景.

`java.lang.ref` 包是 JDK1.2 引入的, 包结构和类分布如下:

```java
- java.lang.ref
  - Cleaner.class
  - Finalizer.class
  - FinalizerHistogram.class
  - FinalReference.class
  - PhantomReference.class
  - Reference.class
  - ReferenceQueue.class
  - SoftReference.classs
  - WeakReference.class
```

引入此包的作用是对引用的概念进行了扩充, 将引用分为强引用(Strong Reference)、软引用(Soft Reference)、弱引用(Weak Reference) 和虚引用(Phantom Reference) 四种类型的引用, 还有一种比较特殊的引用是析构引用(Final Reference), 它是一种特化的虚引用.

四种引用的强度按照下面的次序依次减弱:

```java
StrongReference > SoftReference > WeakReference > PhantomReference
```

值得注意的是:

- 强引用没有对应的类型表示, 也就是说强引用是普遍存在的, 如 `Object object = new Object();`.
- 软引用、弱引用和虚引用都是 `java.lang.ref.Reference` 的直接子类.

### 1.2. Reference 源码分析

先看 `Reference` 的构造方法和成员变量:

```java
public abstract class Reference<T> {
    private T referent;
    volatile ReferenceQueue<? super T> queue;
    volatile Reference next;
    transient private Reference<T> discovered;
    
    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
}
```

**构造描述**：

构造方法依赖于一个泛型的 referent 成员以及一个 `ReferenceQueue<? super T>` 的队列, 如果 `ReferenceQueue` 实例为 null, 则使用 `ReferenceQueue.NULL`.

**成员变量描述**：

- referent: 保存了 `Reference` 引用所指向的对象, 下面直接称为 referent.

```java
// GC特殊处理的对象
private T referent;         /* Treated specially by GC */
```

- queue: 当 `Reference` 实例持有的对象 referent 要被回收的时候, `Reference` 实例会被放入引用队列, 那么程序执行的时候可以从引用队列得到或者监控相应的 `Reference` 实例.

```java
    /* The queue this reference gets enqueued to by GC notification or by
     * calling enqueue().
     *
     * When registered: the queue with which this reference is registered.
     *        enqueued: ReferenceQueue.ENQUEUE
     *        dequeued: ReferenceQueue.NULL
     *    unregistered: ReferenceQueue.NULL
     */
    volatile ReferenceQueue<? super T> queue;
```

- next：下一个`Reference`实例的引用, `Reference`实例通过此构造单向的链表.

```java
    /* The link in a ReferenceQueue's list of Reference objects.
     *
     * When registered: null
     *        enqueued: next element in queue (or this if last)
     *        dequeued: this (marking FinalReferences as inactive)
     *    unregistered: null
     */
    @SuppressWarnings("rawtypes")
    volatile Reference next;
```

- discovered：注意这个属性由 transient 修饰, 基于状态表示不同链表中的下一个待处理的对象, 主要是pending-reference 列表的下一个元素, 通过 JVM 直接调用赋值.

```java
/* When active:  next element in a discovered reference list maintained by GC (or this if last)
*     pending:   next element in the pending list (or null if last)
*     otherwise:   NULL
*/
transient private Reference<T> discovered;  /* used by VM */
```

**实例方法(和ReferenceHandler线程不相关的方法)**:

```java
// 获取持有的referent实例
@HotSpotIntrinsicCandidate
public T get() {
     return this.referent;
}

// 把持有的referent实例置为null
public void clear() {
     this.referent = null;
}

// 判断是否处于enqeued状态
public boolean isEnqueued() {
     return (this.queue == ReferenceQueue.ENQUEUED);
}

// 入队参数，同时会把referent置为null
public boolean enqueue() {
     this.referent = null;
     return this.queue.enqueue(this);
}

// 覆盖clone方法并且抛出异常，也就是禁止clone
@Override
protected Object clone() throws CloneNotSupportedException {
     throw new CloneNotSupportedException();
}

// 确保给定的引用实例是强可达的
@ForceInline
public static void reachabilityFence(Object ref) {
}
```

### 1.3. ReferenceHandler线程

ReferenceHandler 线程是由 `Reference` 静态代码块中建立并且运行的线程, 它的运行方法中依赖了比较多的本地(native) 方法, ReferenceHandler 线程的主要功能是处理 pending 链表中的引用对象:

```java
    // ReferenceHandler直接继承于Thread覆盖了run方法
    private static class ReferenceHandler extends Thread {
        
        // 静态工具方法用于确保对应的类型已经初始化
        private static void ensureClassInitialized(Class<?> clazz) {
            try {
                Class.forName(clazz.getName(), true, clazz.getClassLoader());
            } catch (ClassNotFoundException e) {
                throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
            }
        }

        static {
            // 确保Cleaner这个类已经初始化
            // pre-load and initialize Cleaner class so that we don't
            // get into trouble later in the run loop if there's
            // memory shortage while loading/initializing it lazily.
            ensureClassInitialized(Cleaner.class);
        }

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, null, name, 0, false);
        }
        
        // 注意run方法是一个死循环执行processPendingReferences
        public void run() {
            while (true) {
                processPendingReferences();
            }
        }
    }

    /* 原子获取(后)并且清理VM中的pending引用链表
     * Atomically get and clear (set to null) the VM's pending-Reference list.
     */
    private static native Reference<Object> getAndClearReferencePendingList();

    /* 检验VM中的pending引用对象链表是否有剩余元素
     * Test whether the VM's pending-Reference list contains any entries.
     */
    private static native boolean hasReferencePendingList();

    /* 等待直到pending引用对象链表不为null，此方法阻塞的具体实现又VM实现
     * Wait until the VM's pending-Reference list may be non-null.
     */
    private static native void waitForReferencePendingList();

    // 锁对象，用于控制等待pending对象时候的加锁和开始处理这些对象时候的解锁
    private static final Object processPendingLock = new Object();
    // 正在处理pending对象的时候，这个变量会更新为true，处理完毕或者初始化状态为false，用于避免重复处理或者重复等待
    private static boolean processPendingActive = false;

    // 这个是死循环中的核心方法，功能是处理pending链表中的引用元素
    private static void processPendingReferences() {
        // Only the singleton reference processing thread calls
        // waitForReferencePendingList() and getAndClearReferencePendingList().
        // These are separate operations to avoid a race with other threads
        // that are calling waitForReferenceProcessing().
        // （1）等待
        waitForReferencePendingList();
        Reference<Object> pendingList;
        synchronized (processPendingLock) {
            // （2）获取并清理，标记处理中状态
            pendingList = getAndClearReferencePendingList();
            processPendingActive = true;
        }
        // （3）通过discovered(下一个元素)遍历pending链表进行处理
        while (pendingList != null) {
            Reference<Object> ref = pendingList;
            pendingList = ref.discovered;
            ref.discovered = null;
            // 如果是Cleaner类型执行执行clean方法并且对锁对象processPendingLock进行唤醒所有阻塞的线程
            if (ref instanceof Cleaner) {
                ((Cleaner)ref).clean();
                // Notify any waiters that progress has been made.
                // This improves latency for nio.Bits waiters, which
                // are the only important ones.
                synchronized (processPendingLock) {
                    processPendingLock.notifyAll();
                }
            } else {
                // 非Cleaner类型并且引用队列不为ReferenceQueue.NULL则进行入队操作
                ReferenceQueue<? super Object> q = ref.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(ref);
            }
        }
        // （4）当次循环结束之前再次唤醒锁对象processPendingLock上阻塞的所有线程
        // Notify any waiters of completion of current round.
        synchronized (processPendingLock) {
            processPendingActive = false;
            processPendingLock.notifyAll();
        }
    }
```

ReferenceHandler 线程启动的静态代码块如下:

```java
    static {
        // ThreadGroup继承当前执行线程(一般是主线程)的线程组
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        // 创建线程实例，命名为Reference Handler，配置最高优先级和后台运行(守护线程)，然后启动
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
        // 注意这里覆盖了全局的jdk.internal.misc.JavaLangRefAccess实现
        // provide access in SharedSecrets
        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
            @Override
            public boolean waitForReferenceProcessing()
                throws InterruptedException{
                return Reference.waitForReferenceProcessing();
            }

            @Override
            public void runFinalization() {
                Finalizer.runFinalization();
            }
        });
    }

    // 如果正在处理pending链表中的引用对象或者监测到VM中的pending链表中还有剩余元素则基于锁对象processPendingLock进行等待
    private static boolean waitForReferenceProcessing()
        throws InterruptedException{
        synchronized (processPendingLock) {
            if (processPendingActive || hasReferencePendingList()) {
                // Wait for progress, not necessarily completion.
                processPendingLock.wait();
                return true;
            } else {
                return false;
            }
        }
    }
```

由于 ReferenceHandler 线程是 `Reference` 的静态代码创建的, 所以只要 `Reference` 这个父类被初始化, 该线程就会创建和运行, 由于它是守护线程, 除非 JVM 进程终结, 否则它会一直在后台运行(注意它的 `run()` 方法里面使用了死循环).




## 2. 强引用(StrongReference)

强引用是使用最普遍的引用. 如果一个对象具有强引用, 那垃圾回收器绝不会回收它. 如下:

```java
Object strongReference = new Object();
```

当内存空间不足时, Java 虚拟机宁愿抛出 `OutOfMemoryError` 错误, 使程序异常终止, 也不会靠随意回收具有强引用的对象来解决内存不足的问题.

如果强引用对象不使用时, 需要弱化从而使 GC 能够回收, 如下:

```java
strongReference = null;
```

显式地设置 `strongReference` 对象为 `null`, 或让其超出对象的生命周期范围, 则 gc 认为该对象不存在引用, 这时就可以回收这个对象. 具体什么时候收集这要取决于 GC 算法.

```java
public void test() {
    Object strongReference = new Object();
    // 省略其他操作
}
```

在一个方法的内部有一个强引用, 这个引用保存在 Java 栈中, 而真正的引用内容(`Object`) 保存在 Java 堆中.

当这个方法运行完成后, 就会退出方法栈, 则引用对象的引用数为0, 这个对象会被回收.

但是如果这个 `strongReference` 是全局变量时, 就需要在不用这个对象时赋值为 `null`, 因为强引用不会被垃圾回收.

`ArrayList` 的 `clear` 方法:

![](./img/16576be9edfcad06.png)

在 `ArrayList` 类中定义了一个 `elementData` 数组, 在调用 `clear` 方法清空数组时, 每个数组元素被赋值为 `null`.

不同于 `elementData=null`, 强引用仍然存在, 避免在后续调用 `add()` 等方法添加元素时进行内存的重新分配.

## 3. 软引用(SoftReference)

如果一个对象只具有软引用, 则内存空间充足时, 垃圾回收器就不会回收它; 如果内存空间不足了, 就会回收这些对象的内存. 只要垃圾回收器没有回收它, 该对象就可以被程序使用.

> 软引用可用来实现内存敏感的高速缓存。

```java
String str = new String("abc");
SoftReference<String> softReference = new SoftReference<String>(str);
```

软引用可以和一个引用队列(`ReferenceQueue`) 联合使用. 如果软引用所引用对象被垃圾回收, JAVA 虚拟机就会把这个软引用加入到与之关联的引用队列中.


## 讨论

**1.对于 不同于 `elementData=null`, 强引用仍然存在 这句话的理解.**


## 参考资料

[深入理解JDK中的Reference原理和源码实现](https://www.throwable.club/2019/02/16/java-reference/)   [浏览本地页面](./html/深入理解JDK中的Reference原理和源码实现 - Throwable.html)


[理解Java的强引用、软引用、弱引用和虚引用](https://juejin.im/post/6844903665241686029)

[Java引用总结--StrongReference、SoftReference、WeakReference、PhantomReference](https://www.cnblogs.com/skywang12345/p/3154474.html)

