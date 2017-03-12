---
title: Java并发实战
tags: Concurrency
---

### 并发基础

竞态状态，原子性
内置锁``synchronized``
重入(复写父类的锁方法)

volatile保证可见性，无法保证原子性

使用条件：

* 对变量的写入操作不依赖变量的当前值，或者能确保只有单线程更新变量的值
* 该变量不会与其他状态变量一起纳入不变性条件
* 在访问变量时不需要加锁



安全发布对象:

* 在静态初始化函数中初始化对象引用
* 将对象的引用保存到``volatile``类型的域或者``AtomicReference``对象中
* 将对象的引用保存到某个正确构造对象的``final``类型域中
* 将对象的引用保存到一个由锁保护的域中

线程安全库: Concurrent包下

并发程序策略:

* 线程封闭
* 只读共享
* 线程安全共享
* 保护对象

### 设计线程安全的类

* 找出构成对象状态的所有变量
* 找出约束状态变量的不变性条件
* 建立对象状态的并发访问管理策略

### CountDownLatch
> 让线程等待直到别的线程执行完，示例

```java
BitSet bitSet = new BitSet();
CountDownLatch countDownLatch = new CountDownLatch(2);

Thread t1 = new Thread(() -> {
    bitSet.set(1);
    countDownLatch.countDown();
});

Thread t2 = new Thread(() -> {
    try {
        Thread.sleep(1000);
    }catch (Exception e) {

    }
    bitSet.set(2);
    countDownLatch.countDown();
});

t1.start();
t2.start();
countDownLatch.await();

System.out.println(bitSet.get(1));
System.out.println(bitSet.get(2));
```

* 首先设置``CountDownLatch``初始2，两个线程内部各自操作，并且对``CountDownLatch``减一操作。只有当``CountDownLatch``减到0时，``await``才能够通过

* 源码分析

```java

private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c - 1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}

public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public void countDown() {
    sync.releaseShared(1);
}
```

* 内部``Sync``继承了``AbstractQueuedSynchronizer``，重写了``tryAcquireShared()``，状态个数达到0返回1，其他情况返回-1。重写了``tryReleaseShared ``，这里采用的是``CAS``(先比较后更新)来更新状态。原理中可以看出最重要的是``AbstractQueuedSynchronizer``框架

### CyclicBarrier
> 让所有的线程都等待到某个点再一起开始，示例

```java
CyclicBarrier cyclicBarrier = new CyclicBarrier(1, new MyAction());
assertEquals(0, cyclicBarrier.getNumberWaiting());
try {
    cyclicBarrier.await();
} catch (InterruptedException | BrokenBarrierException e) {
    e.printStackTrace();
}
System.out.println(countAction);
```
* 设置触发次数为1，当调用了	``CyclicBarrier``的``await``一次之后，主线程开始执行，并且``MyAction``会执行

* 源码分析

```java
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();//获取锁
    try {
        final Generation g = generation;
		  //等待被打断	
        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;
        //判断所有的线程是否都已经达到
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                //执行默认runnable
                    command.run();
                ranAction = true;
                nextGeneration();//更新状态，并且通知所有Condition
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                //等待通知
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
    //释放锁
        lock.unlock();
    }
}
```

* 当前线程获取锁。如果关卡被打破抛出异常, 否则对于关卡数量减一，如果关卡数量为0,执行默认``runnable``并更新状态并发出通知。否则``condition``等待。最后释放锁

### AbstractExecutorService
> 提供``ExecutorService``的默认实现，示例

```java
ExecutorService executorService = new DirectExecutorService();
final AtomicBoolean atomicBoolean = new AtomicBoolean(false);
final Future<?> future = executorService.submit(() -> {
    atomicBoolean.set(true);
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});
assertNull(future.get());
assertNull(future.get(0, TimeUnit.SECONDS));
assertTrue(atomicBoolean.get());
assertTrue(future.isDone());
assertFalse(future.isCancelled());
```

* 源码分析

```java
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}
```
* 创建``FutureTask``，执行``task``

### AbstractQueuedSynchronizer
> 线程锁等待队列，如果锁已经被线程获取，那么后面的线程进入队列等待。其支持``shared``和``exclusive``模式。内部主要依赖UnSafe。示例

```java
static class Mutex extends AbstractQueuedSynchronizer {
    /** An eccentric value for locked synchronizer state. */
    static final int LOCKED = (1 << 31) | (1 << 15);

    static final int UNLOCKED = 0;

    @Override protected boolean isHeldExclusively() {
        int state = getState();
        assertTrue(state == UNLOCKED || state == LOCKED);
        return state == LOCKED;
    }

    @Override protected boolean tryAcquire(int acquires) {
        assertEquals(LOCKED, acquires);
        boolean result = compareAndSetState(UNLOCKED, LOCKED);
        System.out.println("tryAcquire result " + result);
        return result;
    }

    @Override protected boolean tryRelease(int releases) {
        if (getState() != LOCKED) {
            throw new IllegalMonitorStateException();
        }
        assertEquals(LOCKED, releases);
        setState(UNLOCKED);
        return true;
    }

    public boolean tryAcquireNanos(long nanos) throws InterruptedException {
        return tryAcquireNanos(LOCKED, nanos);
    }

    public boolean tryAcquire() {
        return tryAcquire(LOCKED);
    }

    public boolean tryRelease() {
        return tryRelease(LOCKED);
    }

    public void acquire() {
        acquire(LOCKED);
    }

    public void acquireInterruptibly() throws InterruptedException {
        acquireInterruptibly(LOCKED);
    }

    public void release() {
        release(LOCKED);
    }

    public ConditionObject newCondition() {
        return new ConditionObject();
    }
}
```
* 自定义一个锁队列
* 源码分析

```java
//获取独占模式的锁
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

//需要由子类去重写
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

//创建Node
private Node addWaiter(Node mode) {
    Node node = new Node(mode);

    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            U.putObject(node, Node.PREV, oldTail);
            //CAS更新tail
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}

private final void initializeSyncQueue() {
        Node h;
    //初始化tail    
    if (U.compareAndSwapObject(this, HEAD, null, (h = new Node())))
        tail = h;
}

//需要获取锁的线程进入队列
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
        	  //获取之前节点	
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        //更新之前节点的状态 
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}

```
* 分两种模式，独占式和共享式。独占式中，如果有线程已经获取了锁，那么其他线程是无法获取的，共享式中，其他线程也有可能能够获取锁
* 尝试获取锁时，创建独占式节点(``addWaiter``)。之后调用``acquireQueued``。其循环遍历节点，并更新节点状态。如果这些操作调用失败。则线程中断。共享式中，会尝试先释放锁。

### BlockingQueue
>支持阻塞的队列

#### 结构图

![BlockingQueue](./BlockingQueue.png)

#### ArrayBlockingQueue
> 先进先出队列，示例


```

//添加元素
//非阻塞式
offer()
add()

//阻塞式
put()

//获取元素
//非阻塞式
poll() //移除元素
peek() //不移除元素

//阻塞式
//take

BlockingQueue queue = emptyCollection();
for(int i = 0; i < SIZE; i++) {
    queue.add(i);
}
try {
    Thread t = new Thread(() -> {
        try {
            System.out.println("take: " + queue.take());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    });
    t.start();
    queue.put(-1);
} catch (InterruptedException e) {
    e.printStackTrace();
}

printQueueRemainCapacity(queue);
System.out.println("peek: " + queue.peek());
printQueueRemainCapacity(queue);
System.out.println("poll: " + queue.poll());
printQueueRemainCapacity(queue);
```

* ``put``方法会一直阻塞指导队列的有空间容纳新的元素。``take``会阻塞直到队列里有元素

* 源码分析
>底层使用数组作为存储数据结构，非空条件，非满条件来通知数据能够更新

```java

public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}
    
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}

public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
    this(capacity, fair);

    final ReentrantLock lock = this.lock;
    lock.lock(); // Lock only for visibility, not mutual exclusion
    try {
        int i = 0;
        try {
            for (E e : c)
                items[i++] = Objects.requireNonNull(e);
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```
* 提供三种构造方法，并且可以指定数据。将数据复制到数组中，复制数据过程中使用``Lock``锁
* 创建数组大小，创建公平锁或者非公平锁，创建锁条件

```java
public boolean offer(E e) {
    Objects.requireNonNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

public boolean add(E e) {
    return super.add(e);
}

public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}

private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length) putIndex = 0;
    count++;
    notEmpty.signal();
}
```

* ``add``方法调用的是``offer``方法，如果队列已满会抛出异常
* ``offer``方法不支持``null``元素，如果有空间容纳新的元素，那么调用``enqueue``
* ``enqueue``将元素放入数组，并发出队列非空通知

```java
public void put(E e) throws InterruptedException {
    Objects.requireNonNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();//一直阻塞，直到收到队列非满通知
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```

* ``put``操作，如果队列已经满了，那么会一直阻塞到收到队列非满的通知

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0L)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}

public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length) takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return x;
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

* ``pool``方法会调用``dequeue``方法，会将该位置的元素置为``null``,``dequeue``会发出队列未满信号
* ``peek``直接返回队列前面的元素
* ``take``如果队列元素为空，也是调用``dequeue``则会一直阻塞直到接收到队列元素不为空信号
* ``ArrayBlockingQueue``包含一个``Itrs``用于分享当前活动迭代器的状态，并且允许队列更新迭代器的状态。``Itr``实现一个迭代器接口



#### ArrayDeque
> 双端队列,示例

```java
ArrayDeque q = new ArrayDeque();
assertTrue(q.offer(zero));
assertTrue(q.offer(one));
assertSame(zero, q.peekFirst());
assertSame(one, q.peekLast());
```

* 源码分析

```java
public ArrayDeque() {
    elements = new Object[16];
}

private void allocateElements(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    // Find the best power of two to hold elements.
    // Tests "<=" because arrays aren't kept full.
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;

        if (initialCapacity < 0)    // Too many elements, must back off
            initialCapacity >>>= 1; // Good luck allocating 2^30 elements
    }
    elements = new Object[initialCapacity];
}

private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;//容量翻倍
    if (newCapacity < 0)
    	//溢出处理
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    //复制到新的数组
    System.arraycopy(elements, p, a, 0, r);
    //
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```
* 默认大小16，最小大小8。数量上限2^32
* 添加元素

```java
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    if (head == tail)
        doubleCapacity();
}

public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}

```
* 初始，``head``和``tail``都是0，当加入元素时
* ``addFirst``, if``head = 0``，那么``head = (head-1) & (elements.length - 1)``，``head``指向数组尾部,对于``[2][1][0]``，其加入的顺序是``2->1->0``
* ``addLast``,加入的顺序是``0->1->2``
* ``offerFirst``和``offerLast``分别调用``addFirst``和``addLast``
* 获取元素

```java
public E pollFirst() {
    final Object[] elements = this.elements;
    final int h = head;
    @SuppressWarnings("unchecked")
    E result = (E) elements[h];
    // Element is null if deque empty
    if (result != null) {
        elements[h] = null; // Must null out slot
        head = (h + 1) & (elements.length - 1);
    }
    return result;
}

public E pollLast() {
    final Object[] elements = this.elements;
    final int t = (tail - 1) & (elements.length - 1);
    @SuppressWarnings("unchecked")
    E result = (E) elements[t];
    if (result != null) {
        elements[t] = null;
        tail = t;
    }
    return result;
}
```

* ``pollFirst()``，``head``指向的是数组高位索引，将该位置的对象置空并将``head``移动到下一个高位，最终``head = 0``
* ``pollLast()``, ``tail``向数组低位移动
* ``removeFirst``和``removeLast``分别调用``pollFirst``和``pollLast``

#### Atomic(原子系列)
>包括``AtomicXXX``，主要用于原子更新

* 源码分析

```java
public final boolean compareAndSet(boolean expect, boolean update) {
	return U.compareAndSwapInt(this, VALUE,
                           (expect ? 1 : 0),
                           (update ? 1 : 0));
}
```

* 主要依靠``UnSafe``的``compareAndSwapInt``来进行线程安全更新

#### CompletableFuture
>jdk1.8之后，能够被显式完成的任务, 示例

```java
CompletableFuture<Integer> f = new CompletableFuture<>();
f.complete(0);
assertTrue(f.isDone());
assertEquals(Integer.valueOf(0), f.get());
```
* 显式设置结果为0

* 源码分析

```java
volatile Object result;       // Either the result or boxed AltResult
volatile Completion stack;    // Top of Treiber stack of dependent actions
```

* 其内部维护了一个结果，和一个完成栈(链表)

```java
/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the {@link ForkJoinPool#commonPool()} with
 * the value obtained by calling the given Supplier.
 *
 * @param supplier a function returning the value to be used
 * to complete the returned CompletableFuture
 * @param <U> the function's return type
 * @return the new CompletableFuture
 */
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
    return asyncSupplyStage(ASYNC_POOL, supplier);
}

/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the given executor with the value obtained
 * by calling the given Supplier.
 *
 * @param supplier a function returning the value to be used
 * to complete the returned CompletableFuture
 * @param executor the executor to use for asynchronous execution
 * @param <U> the function's return type
 * @return the new CompletableFuture
 */
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                   Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}

/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the {@link ForkJoinPool#commonPool()} after
 * it runs the given action.
 *
 * @param runnable the action to run before completing the
 * returned CompletableFuture
 * @return the new CompletableFuture
 */
public static CompletableFuture<Void> runAsync(Runnable runnable) {
    return asyncRunStage(ASYNC_POOL, runnable);
}

/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the given executor after it runs the given
 * action.
 *
 * @param runnable the action to run before completing the
 * returned CompletableFuture
 * @param executor the executor to use for asynchronous execution
 * @return the new CompletableFuture
 */
public static CompletableFuture<Void> runAsync(Runnable runnable,
                                               Executor executor) {
    return asyncRunStage(screenExecutor(executor), runnable);
}

/**
 * Returns a new CompletableFuture that is already completed with
 * the given value.
 *
 * @param value the value
 * @param <U> the type of the value
 * @return the completed CompletableFuture
 */
public static <U> CompletableFuture<U> completedFuture(U value) {
    return new CompletableFuture<U>((value == null) ? NIL : value);
}

static CompletableFuture<Void> asyncRunStage(Executor e, Runnable f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<Void> d = new CompletableFuture<Void>();
    e.execute(new AsyncRun(d, f));
    return d;
}
```
* 支持多种方式构造``CompletableFuture``，支持配置线程执行器。默认如果支持并行，使用``ForkJoinPool``，否则直接使用线程。最终在``AsyncRun``执行。``AsycRun``继承自``ForkJoinTask``

```java
public void run() {
    CompletableFuture<Void> d; Runnable f;
    if ((d = dep) != null && (f = fn) != null) {
        dep = null; fn = null;
        if (d.result == null) {
            try {
                f.run();
                d.completeNull();
            } catch (Throwable ex) {
                d.completeThrowable(ex);
            }
        }
        d.postComplete();
    }
}

final void postComplete() {
    /*
     * On each step, variable f holds current dependents to pop
     * and run.  It is extended along only one path at a time,
     * pushing others to avoid unbounded recursion.
     */
    CompletableFuture<?> f = this; Completion h;
    while ((h = f.stack) != null ||
           (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
        if (f.casStack(h, t = h.next)) {
            if (t != null) {
                if (f != this) {
                    pushStack(h);
                    continue;
                }
                h.next = null;    // detach
            }
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}
```

* 还未执行完，尝试执行``runnable``。通知``CompletableFuture``执行完成
* 尝试更新完成列表

```java
public T get() throws InterruptedException, ExecutionException {
    Object r;
    return reportGet((r = result) == null ? waitingGet(true) : r);
}

private Object waitingGet(boolean interruptible) {
    Signaller q = null;
    boolean queued = false;
    int spins = SPINS;
    Object r;
    while ((r = result) == null) {
        if (spins > 0) {
            if (ThreadLocalRandom.nextSecondarySeed() >= 0)
                --spins;
        }
        else if (q == null)
            q = new Signaller(interruptible, 0L, 0L);
        else if (!queued)
            queued = tryPushStack(q);
        else {
            try {
                ForkJoinPool.managedBlock(q);
            } catch (InterruptedException ie) { // currently cannot happen
                q.interrupted = true;
            }
            if (q.interrupted && interruptible)
                break;
        }
    }
    if (q != null) {
        q.thread = null;
        if (q.interrupted) {
            if (interruptible)
                cleanStack();
            else
                Thread.currentThread().interrupt();
        }
    }
    if (r != null)
        postComplete();
    return r;
}
```

* 如果结果还未执行完，等待
* 创建``Signaller``，其为记录和释放等待的线程。将``Signaller``交给``ForkJoinPool``管理



#### ForkJoinPool
