---
title: 深入理解RxJava
date: 2017-01-01 23:27:27
tags:
---


``RxJava``使用观察者模式，基于事件驱动。主要包含两部分``Observable``和``Observaber``。而``Rxjava``最大的特色在于其灵活强大的操作符(``Operators``)和调度器(``Scheduler``)

### Operators

#### Creating Observables

##### create(OnSubscribe)
>示例

```java
Observable.create(subscriber -> {
            System.out.println("subscriber");
        }).subscribe(new Subscriber<Object>() {
            @Override public void onCompleted() {
                System.out.println("onCompleted");
            }

            @Override public void onError(Throwable e) {
                System.out.println("onError");
            }

            @Override public void onNext(Object o) {
                System.out.println("onNext");
            }
        });
```

* 这种情况下会打印``subscriber``。
* 这个时候加上

```java
if (!subscriber.isUnsubscribed()) {
                subscriber.onNext(subscriber);
                subscriber.onCompleted();
}
```

* 那么会打印

```java
onNext
onCompleted
```

* 源码分析: 

```java
public static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(RxJavaHooks.onCreate(f));
    }
```

* 通过``RxJavaHooks``创建``OnSubscribe``

```java
public static <T> Observable.OnSubscribe<T> onCreate(Observable.OnSubscribe<T> onSubscribe) {
        Func1<Observable.OnSubscribe, Observable.OnSubscribe> f = onObservableCreate;
        if (f != null) {
            return f.call(onSubscribe);
        }
        return onSubscribe;
    }
```

* ``RxJavaHooks``会在首次使用的时候初始化。

之后还需要订阅

```java
Observable.java

static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
	//省略若干代码
	RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);     
	//省略异常处理代码
}

RxJavaHooks.java

onObservableStart = new Func2<Observable, Observable.OnSubscribe, Observable.OnSubscribe>() {
            @Override
            public Observable.OnSubscribe call(Observable t1, Observable.OnSubscribe t2) {
                return RxJavaPlugins.getInstance().getObservableExecutionHook().onSubscribeStart(t1, t2);
            }
};

RxJavaPlugins.java

public <T> OnSubscribe<T> onSubscribeStart(Observable<? extends T> observableInstance, final OnSubscribe<T> onSubscribe) {
        // pass through by default
        return onSubscribe;
}
```

* 判断订阅者是否为空,之后直接交给``RxJavaHooks``执行``onObservableStart``，这里的参数``observable.onSubscribe``正是``Create``传入的，而在``RxJavaHooks``中``onObservableStart``调用了``RxJavaPlugin``中的``onSubscribeStart``。这个方法直接返回原来的``OnSubscribe``也就是``Create``传入的。这个时候会回调``call``。

* ``create()``有另外两种参数专门用于处理``backPressure(反向压力，生产者生产速度远大于消费者消费速度)``。

``SyncOnSubscribe``里面需要关注的是``SubscriptionProducer``，而``AsyncOnSubscribe``需要关注``AsyncOuterManager``。这两个东西之后再讨论(TODO)

##### defer

> 直到观察者订阅之后才创建可观察对象，示例

```java
Observable.defer(Observable::empty).subscribe(new Subscriber<Object>() {
            @Override public void onCompleted() {
                System.out.println("onCompleted");
            }
        
            @Override public void onError(Throwable e) {
                System.out.println("onError");
            }
        
            @Override public void onNext(Object o) {
                System.out.println("onNext");
            }
});
```

* 源码分析

```java
Observable.java

2.0.3

public static <T> Observable<T> defer(Callable<? extends ObservableSource<? extends T>> supplier) {
        ObjectHelper.requireNonNull(supplier, "supplier is null");
        return RxJavaPlugins.onAssembly(new ObservableDefer<T>(supplier));
    }
    
1.2.4

public static <T> Observable<T> defer(Func0<Observable<T>> observableFactory) {
        return create(new OnSubscribeDefer<T>(observableFactory));
    }    
```

* 1.2.4采用``OnSubscribeDefer``,2.0.3采用``ObservableDefer``。

```java
OnSubscribeDefer.java

@Override
    public void call(final Subscriber<? super T> s) {
        Observable<? extends T> o;
        try {
            o = observableFactory.call();
        } catch (Throwable t) {
            Exceptions.throwOrReport(t, s);
            return;
        }
        o.unsafeSubscribe(Subscribers.wrap(s));
}

ObservableDefer.java

@Override
    public void subscribeActual(Observer<? super T> s) {
        ObservableSource<? extends T> pub;
        try {
            pub = ObjectHelper.requireNonNull(supplier.call(), "null publisher supplied");
        } catch (Throwable t) {
            Exceptions.throwIfFatal(t);
            EmptyDisposable.error(t, s);
            return;
        }

        pub.subscribe(s);
}
```

##### Empty Never Throw

> Empty

* 源码分析

```java
EmptyObservableHolder.java

@Override
    public void call(Subscriber<? super Object> child) {
        child.onCompleted();
}
```

* 直接调用``onCompleted``，这是一个枚举单例

> Never

* 源码分析

```java
NeverObservableHolder.java

@Override
public void call(Subscriber<? super Object> child) {
        // deliberately no op
}
```

* 什么都不做，主要用于做测试,这是一个枚举单例


> Throw

* 源码分析:

```java
@Override 
public void call(Subscriber<? super T> observer) {
        observer.onError(exception);
}
```

##### From

> 将``Future``转化成``Observable``，示例

```java
Observable.from(Executors.newSingleThreadExecutor().submit(() -> {
            System.out.println("before sleep");
            Thread.sleep(5000);
            return true;
        })).subscribe(mSubscriber);
```

* 打印结果

```
before sleep
onNext
onCompleted
```

* 源码分析

```java
Observable.java

public static <T> Observable<T> from(Future<? extends T> future) {
        return (Observable<T>)create(OnSubscribeToObservableFuture.toObservableFuture(future));
}
```

```java
OnSubscribeToObservableFuture.java

public static <T> OnSubscribe<T> toObservableFuture(final Future<? extends T> that) {
        return new ToObservableFuture<T>(that);
}

@Override
        public void call(Subscriber<? super T> subscriber) {
        //核心代码
        if (subscriber.isUnsubscribed()) {
                    return;
                }
                T value = (unit == null) ? (T) that.get() : (T) that.get(time, unit);
                subscriber.setProducer(new SingleProducer<T>(subscriber, value));
}
```

* 等待``future``结果。将结果作为生产者发出消息.``push data``

> 将数组转换成Observable

* 源码

```java
OnSubscribeFromArray.java

public void call(Subscriber<? super T> child) {
        child.setProducer(new FromArrayProducer<T>(child, array));
}
```

* 这里面依然涉及到反向压力的处理，当请求id等于阈值时采用``fastPath``处理。反之采用``slowPath``。正常情况下请求id都是``Long.MaxValue``，当``producer``为空时，id会发生改变

##### Interval
>定时发出事件,示例

```java
final CountDownLatch latch = new CountDownLatch(5);
        long startTime = System.currentTimeMillis();
        Observable.interval(1, TimeUnit.SECONDS).subscribe(counter -> {
            latch.countDown();
            System.out.println("Got: " + counter + " time: " + (System.currentTimeMillis() - startTime));
        });
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

结果
Got: 0 time: 1252
Got: 1 time: 2245
Got: 2 time: 3249
Got: 3 time: 4247
Got: 4 time: 5244
       
```

* 源码分析

```java
OnSubscribeTimerPeriodically.java

默认使用computation调度器
public void call(final Subscriber<? super Long> child) {
        final Worker worker = scheduler.createWorker();
        child.add(worker);
        worker.schedulePeriodically(new Action0() {
            long counter;
            @Override
            public void call() {
                try {
                    child.onNext(counter++);
                } catch (Throwable e) {
                    try {
                        worker.unsubscribe();
                    } finally {
                        Exceptions.throwOrReport(e, child);
                    }
                }
            }

        }, initialDelay, period, unit);
}

```

* 创建工作器。工作器串行执行。

##### Just
> 发送特定的消息，有多个重载方法，区分在于单个参数和多个参数，单个参数使用ScalarSynchronousObservable,而多个参数使用from

* 源码分析

```java
ScalarSynchronousObservable.java

protected ScalarSynchronousObservable(final T t) {
        super(RxJavaHooks.onCreate(new JustOnSubscribe<T>(t)));
        this.t = t;
}

public void call(Subscriber<? super T> s) {
            s.setProducer(createProducer(s, value));
}
```

* * 创建生产者时区分单生产者和弱单生产者

```java
WeakSingleProducer.java

public void request(long n) {
            if (once) {
                return;
            }
            if (n < 0L) {
                throw new IllegalStateException("n >= required but it was " + n);
            }
            if (n == 0L) {
                return;
            }
            once = true;
            Subscriber<? super T> a = actual;
            if (a.isUnsubscribed()) {
                return;
            }
            T v = value;
            try {
                a.onNext(v);
            } catch (Throwable e) {
                Exceptions.throwOrReport(e, a, v);
                return;
            }

            if (a.isUnsubscribed()) {
                return;
            }
            a.onCompleted();
}
```
* 弱单生产者不考虑并发问题，仅仅只使用``once``来记录此次请求已经发出

```java
public void request(long n) {
        // negative requests are bugs
        if (n < 0) {
            throw new IllegalArgumentException("n >= 0 required");
        }
        // we ignore zero requests
        if (n == 0) {
            return;
        }
        // atomically change the state into emitting mode
        if (compareAndSet(false, true)) {
            // avoid re-reading the instance fields
            final Subscriber<? super T> c = child;
            // eagerly check for unsubscription
            if (c.isUnsubscribed()) {
                return;
            }
            T v = value;
            // emit the value
            try {
                c.onNext(v);
            } catch (Throwable e) {
                Exceptions.throwOrReport(e, c, v);
                return;
            }
            // eagerly check for unsubscription
            if (c.isUnsubscribed()) {
                return;
            }
            // complete the child
            c.onCompleted();
        }
}
```

* 单生产者考虑并发问题，继承``AtomicBoolean``，通过``compareAndSet``来确保线程安全

##### Range
>串行发出某一范围的消息

* 源码分析

```java
public void request(long requestedAmount) {
            if (get() == Long.MAX_VALUE) {
                // already started with fast-path
                return;
            }
            if (requestedAmount == Long.MAX_VALUE && compareAndSet(0L, Long.MAX_VALUE)) {
                // fast-path without backpressure
                fastPath();
            } else if (requestedAmount > 0L) {
                long c = BackpressureUtils.getAndAddRequest(this, requestedAmount);
                if (c == 0L) {
                    // backpressure is requested
                    slowPath(requestedAmount);
                }
            }
}
```

* 策略依然跟大部分``producer``一样,正常情况下使用``fastPath``，``slowPath``则会考虑线程安全

##### Repeat
>重复发出消息，示例

```java
Observable.range(0, 3).repeat(3).subscribe(mSubscriber);

打印结果:
onNext value: 0
onNext value: 1
onNext value: 2
onNext value: 0
onNext value: 1
onNext value: 2
onNext value: 0
onNext value: 1
onNext value: 2
onCompleted

打印三组range
```

* 源码分析

```java
repeat默认使用trampoline作为线程调度器，在当前线程串行执行

OnSubscribeRedo.java

public void call(final Subscriber<? super T> child) {

	//订阅源消息
	final Action0 subscribeToSource = new Action0() {
            @Override
            public void call() {
                if (child.isUnsubscribed()) {
                    return;
                }

                Subscriber<T> terminalDelegatingSubscriber = new Subscriber<T>() {
                    boolean done;

                    @Override
                    public void onCompleted() {
                        if (!done) {
                            done = true;
                            unsubscribe();
                            terminals.onNext(Notification.createOnCompleted());
                        }
                    }

                    @Override
                    public void onError(Throwable e) {
                        if (!done) {
                            done = true;
                            unsubscribe();
                            terminals.onNext(Notification.createOnError(e));
                        }
                    }

                    @Override
                    public void onNext(T v) {
                        if (!done) {
                            child.onNext(v);
                            decrementConsumerCapacity();
                            arbiter.produced(1);
                        }
                    }

                    private void decrementConsumerCapacity() {
                        // use a CAS loop because we don't want to decrement the
                        // value if it is Long.MAX_VALUE
                        while (true) {
                            long cc = consumerCapacity.get();
                            if (cc != Long.MAX_VALUE) {
                                if (consumerCapacity.compareAndSet(cc, cc - 1)) {
                                    break;
                                }
                            } else {
                                break;
                            }
                        }
                    }

                    @Override
                    public void setProducer(Producer producer) {
                        arbiter.setProducer(producer);
                    }
                };
                // new subscription each time so if it unsubscribes itself it does not prevent retries
                // by unsubscribing the child subscription
                sourceSubscriptions.set(terminalDelegatingSubscriber);
                source.unsafeSubscribe(terminalDelegatingSubscriber);
            }
        };
        
        //重复发送源消息
        final Observable<?> restarts = controlHandlerFunction.call(
                terminals.lift(new Operator<Notification<?>, Notification<?>>() {
                    @Override
                    public Subscriber<? super Notification<?>> call(final Subscriber<? super Notification<?>> filteredTerminals) {
                        return new Subscriber<Notification<?>>(filteredTerminals) {
                            @Override
                            public void onCompleted() {
                                filteredTerminals.onCompleted();
                            }

                            @Override
                            public void onError(Throwable e) {
                                filteredTerminals.onError(e);
                            }

                            @Override
                            public void onNext(Notification<?> t) {
                                if (t.isOnCompleted() && stopOnComplete) {
                                    filteredTerminals.onCompleted();
                                } else if (t.isOnError() && stopOnError) {
                                    filteredTerminals.onError(t.getThrowable());
                                } else {
                                    filteredTerminals.onNext(t);
                                }
                            }

                            @Override
                            public void setProducer(Producer producer) {
                                producer.request(Long.MAX_VALUE);
                            }
                        };
                    }
                }));
     //订阅重复发送的源消息           
     worker.schedule(new Action0() {
            @Override
            public void call() {
                restarts.unsafeSubscribe(new Subscriber<Object>(child) {
                    @Override
                    public void onCompleted() {
                        child.onCompleted();
                    }

                    @Override
                    public void onError(Throwable e) {
                        child.onError(e);
                    }

                    @Override
                    public void onNext(Object t) {
                        if (!child.isUnsubscribed()) {
                            // perform a best endeavours check on consumerCapacity
                            // with the intent of only resubscribing immediately
                            // if there is outstanding capacity
                            if (consumerCapacity.get() > 0) {
                                worker.schedule(subscribeToSource);
                            } else {
                                // set this to true so that on next request
                                // subscribeToSource will be scheduled
                                resumeBoundary.compareAndSet(false, true);
                            }
                        }
                    }

                    @Override
                    public void setProducer(Producer producer) {
                        producer.request(Long.MAX_VALUE);
                    }
                });
            }
        });                
}
```
* 流程
  1. 订阅源消息
  2. 重复发送源消息
  3. 订阅重复发送的源消息

##### Timer  
> 延时特定时间后发出一个消息，默认使用computation作为线程调度器

* 源码分析

```java
public void call(final Subscriber<? super Long> child) {
        Worker worker = scheduler.createWorker();
        child.add(worker);
        worker.schedule(new Action0() {
            @Override
            public void call() {
                try {
                    child.onNext(0L);
                } catch (Throwable t) {
                    Exceptions.throwOrReport(t, child);
                    return;
                }
                child.onCompleted();
            }
        }, time, unit);
}
```

#### Transforming Observables

##### Buffer
> 缓存源数据发出的消息，再一起发出。示例

```java
Integer[] integers = new Integer[9];
for(int i = 0; i < integers.length; i++) {
    integers[i] = i;
}
Observable.from(integers).buffer(2).subscribe(mSubscriber);

结果:2个值一组

onNext value: [0, 1]
onNext value: [2, 3]
onNext value: [4, 5]
onNext value: [6, 7]
onNext value: [8]
onCompleted
```

* 源码分析

###### ``buffer(int count, int skip)``
   
```java
OperatorBufferWithSize.java

public Subscriber<? super T> call(final Subscriber<? super List<T>> child) {
        if (skip == count) {
            BufferExact<T> parent = new BufferExact<T>(child, count);

            child.add(parent);
            child.setProducer(parent.createProducer());

            return parent;
        }
        if (skip > count) {
            BufferSkip<T> parent = new BufferSkip<T>(child, count, skip);

            child.add(parent);
            child.setProducer(parent.createProducer());

            return parent;
        }
        BufferOverlap<T> parent = new BufferOverlap<T>(child, count, skip);

        child.add(parent);
        child.setProducer(parent.createProducer());

        return parent;
}
```

* 当``skip``等于``count``(默认情况)

```java
BufferExact

public void onNext(T t) {
            List<T> b = buffer;
            if (b == null) {
                b = new ArrayList<T>(count);
                buffer = b;
            }

            b.add(t);

            if (b.size() == count) {
                buffer = null;
                actual.onNext(b);
            }
}
```
* 创建大小为``count``的数组，将接收到的数组添加到数组中，数组满，将数组作为数据发出

* ``skip``大于``count``

```java
BufferSkip.java

public void onNext(T t) {
            long i = index;
            List<T> b = buffer;
            if (i == 0) {
                b = new ArrayList<T>(count);
                buffer = b;
            }
            i++;
            if (i == skip) {
                index = 0;
            } else {
                index = i;
            }

            if (b != null) {
                b.add(t);

                if (b.size() == count) {
                    buffer = null;
                    actual.onNext(b);
                }
            }
}


```

* 每组``count``个，接收``skip``个数据，填满``count``个，剩余丢弃

* ``count``大于``skip``

```java
BufferOverlap.java

public void onNext(T t) {
            long i = index;
            if (i == 0) {
                List<T> b = new ArrayList<T>(count);
                queue.offer(b);
            }
            i++;
            if (i == skip) {
                index = 0;
            } else {
                index = i;
            }

            for (List<T> list : queue) {
                list.add(t);
            }

            List<T> b = queue.peek();
            if (b != null && b.size() == count) {
                queue.poll();
                produced++;
                actual.onNext(b);
            }
}
```
* 使用``ArrayDeque``存储每组数据。``ArrayDeque``会储存之前的数据。

###### buffer(Func0)

* 源码分析

```java
OperatorBufferWithSingleObservable.java

public Subscriber<? super T> call(final Subscriber<? super List<T>> child) {
        Observable<? extends TClosing> closing;
        try {
            closing = bufferClosingSelector.call();
        } catch (Throwable t) {
            Exceptions.throwOrReport(t, child);
            return Subscribers.empty();
        }
        final BufferingSubscriber s = new BufferingSubscriber(new SerializedSubscriber<List<T>>(child));

        Subscriber<TClosing> closingSubscriber = new Subscriber<TClosing>() {

            @Override
            public void onNext(TClosing t) {
                s.emit();
            }

            @Override
            public void onError(Throwable e) {
                s.onError(e);
            }

            @Override
            public void onCompleted() {
                s.onCompleted();
            }
        };

        child.add(closingSubscriber);
        child.add(s);

        closing.unsafeSubscribe(closingSubscriber);

        return s;
}


```

* ``bufferClosingSelector``产生的``Observable``订阅``BufferingSubscriber``其内部使用``chunk``收集数据，当``closingSubscriber``被订阅，将``chunk``数据发出

###### buffer(long timespan, long timeshift, TimeUnit unit)

* 源码分析
>默认使用computation线程调度

```java
OperatorBufferWithTime.java

if (timespan == timeshift) {
            ExactSubscriber parent = new ExactSubscriber(serialized, inner);
            parent.add(inner);
            child.add(parent);
            parent.scheduleExact();
            return parent;
        }

        InexactSubscriber parent = new InexactSubscriber(serialized, inner);
        parent.add(inner);
        child.add(parent);
        parent.startNewChunk();
        parent.scheduleChunk();
return parent;
```

* 数据刷新时间和数据块创建时间一样时使用``ExactSubscriber``

```java
public void onNext(T t) {
            List<T> toEmit = null;
            synchronized (this) {
                if (done) {
                    return;
                }
                chunk.add(t);
                if (chunk.size() == count) {
                    toEmit = chunk;
                    chunk = new ArrayList<T>();
                }
            }
            if (toEmit != null) {
                child.onNext(toEmit);
            }
}
```

* 反之使用``InexactSubscriber``，其使用``LinkedList``作为``chunk``数据结构。用于存储多余的数据。数据生成的速度大于处理速度

##### FlatMap
> 转换消息成被观察对象，对map(func)的结果做merge

* 源码分析

```java
Observable.java

public static <T> Observable<T> merge(Observable<? extends Observable<? extends T>> source) {
        if (source.getClass() == ScalarSynchronousObservable.class) {
            return ((ScalarSynchronousObservable<T>)source).scalarFlatMap((Func1)UtilityFunctions.identity());
        }
        return source.lift(OperatorMerge.<T>instance(false));
}

OperatorMerge.java，合并多个Observable成一个

public Subscriber<Observable<? extends T>> call(final Subscriber<? super T> child) {
        MergeSubscriber<T> subscriber = new MergeSubscriber<T>(child, delayErrors, maxConcurrent);
        MergeProducer<T> producer = new MergeProducer<T>(subscriber);
        subscriber.producer = producer;

        child.add(subscriber);
        child.setProducer(producer);

        return subscriber;
 }
 
 MergeSubscriber.java
 
 public void onNext(Observable<? extends T> t) {
            if (t == null) {
                return;
            }
            if (t == Observable.empty()) {
                emitEmpty();
            } else
            if (t instanceof ScalarSynchronousObservable) {
                tryEmit(((ScalarSynchronousObservable<? extends T>)t).get());
            } else {
                InnerSubscriber<T> inner = new InnerSubscriber<T>(this, uniqueId++);
                addInner(inner);
                t.unsafeSubscribe(inner);
                emit();
            }
}

```

* 创建``InnerSubscriber``，其内部会调用``tryEmit``。之后调用``emitLoop``，循环从唤醒队列中取出消息发出

##### GroupBy
> 将源数据按特征分组，示例

```java
Observable.just(1, 2, 3, 4, 5)
                  .groupBy(integer -> integer % 2 == 0)
                  .subscribe(grouped -> grouped.toList().subscribe(integers -> System.out.println(integers + " (Even: " + grouped.getKey() + ")")));
```

* 源码分析

```java
OperatorGroupBy.java

public Subscriber<? super T> call(Subscriber<? super GroupedObservable<K, V>> child) {
        final GroupBySubscriber<T, K, V> parent; // NOPMD
        try {
            parent = new GroupBySubscriber<T, K, V>(child, keySelector, valueSelector, bufferSize, delayError, mapFactory);
        } catch (Throwable ex) {
            //Can reach here because mapFactory.call() may throw in constructor of GroupBySubscriber
            Exceptions.throwOrReport(ex, child);
            Subscriber<? super T> parent2 = Subscribers.empty();
            parent2.unsubscribe();
            return parent2;
        }

        child.add(Subscriptions.create(new Action0() {
            @Override
            public void call() {
                parent.cancel();
            }
        }));

        child.setProducer(parent.producer);

        return parent;
}

GroupBySubscriber.java

public void onNext(T t) {
	K key;
	try {
	    key = keySelector.call(t);
	} catch (Throwable ex) {
	    unsubscribe();
	    errorAll(a, q, ex);
	    return;
	}
	
	...
	
	if (group == null) {
        // if the main has been cancelled, stop creating groups
        // and skip this value
        if (!cancelled.get()) {
            group = GroupedUnicast.createWith(key, bufferSize, this, delayError);
            groups.put(mapKey, group);

            groupCount.getAndIncrement();

            notNew = false;
            q.offer(group);
            drain();
        } else {
            return;
        }
    }
    
    ...
    
    V v;
    try {
        v = valueSelector.call(t);
    } catch (Throwable ex) {
        unsubscribe();
        errorAll(a, q, ex);
        return;
    }

    group.onNext(v);
}
```

* 使用键选择器生成``key``，使用``key``创建``GroupedUnicast``存储起来。通过值选择器生成值并发出。


##### Map
>与flatmap类似，flatmap多了merge步骤

##### Scan
>对新发出的信息与之前发出的消息一起做某种处理。示例

```java
Observable.just(1, 2, 3, 4, 5).scan((sum, value) -> sum + value).subscribe(integer -> System.out.println("Sum: " + integer));

结果:
Sum: 1
Sum: 3
Sum: 6
Sum: 10
Sum: 15
```

* 源码分析

```java
OperatorScan.java

//没有初始值
public void onNext(T t) {
    R v;
    if (!once) {
        once = true;
        v = (R)t;
    } else {
        v = value;
        try {
        		//使用之前的结果
            v = accumulator.call(v, t);
        } catch (Throwable e) {
            Exceptions.throwOrReport(e, child, t);
            return;
        }
    }
    value = v;
    child.onNext(v);
}

//有初始值
@Override
public void onNext(T currentValue) {
    R v = value;
    try {
        v = accumulator.call(v, currentValue);
    } catch (Throwable e) {
        Exceptions.throwOrReport(e, this, currentValue);
        return;
    }
    value = v;
    ip.onNext(v);
}
```

* 接收到消息，使用之前计算的结果和当前值做计算并存储


##### Window
> 与buffer类似，但其缓存后是以新的对象作为消息发出(Subject)

#### Filtering Observables

##### Debounce
> 只发出经过特定时间从源消息接收到的消息，示例

```java
Observable.from(integers).debounce(Observable::just).subscribe(mSubscriber);

结果:
onNext value: 0
onNext value: 1
onNext value: 2
onNext value: 3
onNext value: 4
onNext value: 5
onNext value: 6
onNext value: 7
onNext value: 8
onCompleted

Observable.from(integers).debounce(1, TimeUnit.SECONDS).subscribe(mSubscriber);

结果:
onNext value: 8
onCompleted
```

* 源码分析

```java
OperatorDebounceWithTime.java

public void onNext(final T t) {
    final int index = state.next(t);
    serial.set(worker.schedule(new Action0() {
        @Override
        public void call() {
            state.emit(index, s, self);
        }
    }, timeout, unit));
}


```
* 接收源消息自增。定时调用``emit``发送消息，如果源消息的发送速度快于当前调度时间，则这些事件会丢弃。事件发送结束时会发送最后一个消息

##### Distinct
>过滤重复的消息，示例

```java
Observable.just(1, 2, 3, 4, 1, 2, 3, 4).distinct().subscribe(integer -> System.out.println("integer: " + integer));

结果:
integer: 1
integer: 2
integer: 3
integer: 4
```

* 源码分析

```java
OperatorDistinct.java

public void onNext(T t) {
    U key = keySelector.call(t);
    if (keyMemory.add(key)) {
        child.onNext(t);
    } else {
        request(1);
    }
}
```

* 使用一个``Set``来存储消息

##### ElementAt
> 只发送第n个消息，示例

```java
Observable.from(integers).elementAt(5).subscribe(mSubscriber);

结果:
onNext value: 5
onCompleted
```

* 源码分析

```java
OperatorElementAt.java

public void onNext(T value) {
    if (currentIndex++ == index) {
        child.onNext(value);
        child.onCompleted();
        unsubscribe();
    }
}
```

* 只有当第n个消息才发送

##### Filter
> 按特定条件过滤消息，示例

```java
Observable.just(1, 2, 3, 4).filter(integer -> integer > 2).subscribe(integer -> System.out.println("integer: " + integer));

结果:
integer: 3
integer: 4

```

* 源码分析

```java
OnSubscribeFilter.java

public void onNext(T t) {
    boolean result;

    try {
        result = predicate.call(t);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        unsubscribe();
        onError(OnErrorThrowable.addValueAsLastCause(ex, t));
        return;
    }

    if (result) {
        actual.onNext(t);
    } else {
        request(1);
    }
}
```

* 使用选择器过滤消息

##### first
> 发出第一个消息，示例

```java
Observable.from(integers).first(integer -> false).subscribe(mSubscriber);

可以决定第一个消息是否发出

结果:
onError
```

* 源码分析

```java
OperatorTake.java

public void onNext(T i) {
    if (!isUnsubscribed() && count++ < limit) {
        boolean stop = count == limit;
        child.onNext(i);
        if (stop && !completed) {
            completed = true;
            try {
                child.onCompleted();
            } finally {
                unsubscribe();
            }
        }
    }
}
```

* 只发出指定数量的消息

##### IgnoreElements
> 不发出任何消息，但发出结束的通知，示例

```java
Observable.from(integers).ignoreElements().subscribe(mSubscriber);

结果:
onCompleted
```

##### last
> 只发出最后一个消息，示例

```java
Observable.from(integers).last().subscribe(mSubscriber);

结果：
onNext value: 8
onCompleted
```

* 源码分析

```java
public void onNext(T t) {
	//存储最新的值
    value = t;
}

public void onCompleted() {
    Object o = value;
    if (o == EMPTY) {
        complete();
    } else {
    	//结束时，发出最后一个消息
        complete((T)o);
    }
}
```

* 结束时，发出最后的消息

##### Sample
> 发出一段时间内最近的消息

* 源码分析

```java
OperatorSampleWithTime.java

public void onNext(T t) {
    value.set(t);
}

public void onCompleted() {
    emitIfNonEmpty();
    subscriber.onCompleted();
    unsubscribe();
}

private void emitIfNonEmpty() {
    Object localValue = value.getAndSet(EMPTY_TOKEN);
    if (localValue != EMPTY_TOKEN) {
        try {
            @SuppressWarnings("unchecked")
            T v = (T)localValue;
            subscriber.onNext(v);
        } catch (Throwable e) {
            Exceptions.throwOrReport(e, this);
        }
    }
}
```

* 存储最新的值，结束时发出最新的值


##### skip
>跳过n个消息不发送，示例

```java
Observable.from(integers).skip(5).subscribe(mSubscriber);

结果:
onNext value: 5
onNext value: 6
onNext value: 7
onNext value: 8
onCompleted
```

* 源码分析

```java
OperatorSkip.java

public void onNext(T t) {
    if (skipped >= toSkip) {
        child.onNext(t);
    } else {
        skipped += 1;
    }
}
```

* 跳过n个消息

##### skipLast
>跳过后n个消息不发送，示例

```java
Observable.from(integers).skipLast(5).subscribe(mSubscriber);

结果:
onNext value: 0
onNext value: 1
onNext value: 2
onNext value: 3
onCompleted
```

* 源码分析

```java
OperatorSkipLast.java

public void onNext(T value) {
    if (count == 0) {
        // If count == 0, we do not need to put value into deque
        // and remove it at once. We can emit the value
        // directly.
        subscriber.onNext(value);
        return;
    }
    if (deque.size() == count) {
        subscriber.onNext(NotificationLite.<T>getValue(deque.removeFirst()));
    } else {
        request(1);
    }
    deque.offerLast(NotificationLite.next(value));
}
```

* 使用双端队列存储消息，将前面的消息存入，当数量达到要求，从双端队列中取数据发送

##### Take, TakeLast
>发送前n个消息

* 源码分析，前面已有

#### Combining Observable

##### And/Then/When
>通过patter和plan合并多个消息源发出的数据集,不是rxjava核心库的一部分

##### CombineLast
>使用特定方法合并两个消息源最近的数据，示例

```java
CountDownLatch countDownLatch = new CountDownLatch(1);
        MySubscriber subscriber = new MySubscriber();
        subscriber.setCountDownLatch(countDownLatch);
        Observable.combineLatest(Observable.interval(1, TimeUnit.SECONDS).take(5), Observable.from(integers), (first, second) -> first + second).subscribe(subscriber);
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
结果:
onNext value: 8
onNext value: 9
onNext value: 10
onNext value: 11
onNext value: 12
onCompleted
        
```
* 第一个消息源一秒钟发出一个消息，第二个消息源一次性发完，因此合并消息会合并最后一个消息

* 源码分析

```java
LatestCoordinator.java

public void subscribe(Observable<? extends T>[] sources) {
    Subscriber<T>[] as = subscribers;
    int len = as.length;
    for (int i = 0; i < len; i++) {
        as[i] = new CombinerSubscriber<T, R>(this, i);
    }
    lazySet(0); // release array contents
    actual.add(this);
    actual.setProducer(this);
    for (int i = 0; i < len; i++) {
        if (cancelled) {
            return;
        }
        ((Observable<T>)sources[i]).subscribe(as[i]);
    }
}
```

* 创建多个``CombinerSubscriber``对应多个消息源。

```java
CombinerSubscriber.java

void combine(Object value, int index) {
	if (!empty) {
        if (value != null && allSourcesFinished) {
        //插入最新的数据
            queue.offer(combinerSubscriber, latest.clone());
        } else
        if (value == null && error.get() != null && (o == MISSING || !delayError)) {
            done = true; // if this source completed without a value
        }
    } else {
        done = true;
    }
	 drain()
}

drain() {
	R v;
    try {
        v = combiner.call(array);
    } catch (Throwable ex) {
        cancelled = true;
        cancel(q);
        a.onError(ex);
        return;
    }

    a.onNext(v);
}
```

* 收集最近的数据到``SpscLinkedArrayQueue``中

##### Join
>只要一个数据在另一个数据定义的时间窗口内发出，合并两个数据源发出的数据

##### Merge
>根据时间轴合并多个数据源成一个，示例

```java
CountDownLatch countDownLatch = new CountDownLatch(1);
    MySubscriber subscriber = new MySubscriber();
    subscriber.setCountDownLatch(countDownLatch);
    Observable.merge(Observable.interval(1, TimeUnit.SECONDS).take(5), Observable.from(integers)).subscribe(subscriber);
    try {
        countDownLatch.await();
    } catch (InterruptedException e) {
        e.printStackTrace();
}

结果:
onNext value: 0
onNext value: 1
onNext value: 2
onNext value: 3
onNext value: 4
onNext value: 5
onNext value: 6
onNext value: 7
onNext value: 8
onNext value: 0
onNext value: 1
onNext value: 2
onNext value: 3
onNext value: 4
onCompleted
```

##### StartWith
>开始发送数据前发送一系列指定的数据，示例

```java
Observable.from(integers).startWith(100).subscribe(mSubscriber);

结果:
onNext value: 100
onNext value: 0
onNext value: 1
onNext value: 2
onNext value: 3
onNext value: 4
onNext value: 5
onNext value: 6
onNext value: 7
onNext value: 8
onCompleted
```

* 源码分析，其使用的是``concat``

```java
OnSubscribeConcatMap.java

public void onNext(T t) {
    if (!queue.offer(NotificationLite.next(t))) {
        unsubscribe();
        onError(new MissingBackpressureException());
    } else {
        drain();
    }
}
```

* 队列未满时，往队列里插入数据，之后调用``drain``消费数据

##### Switch
> 转换多个数据源为单个数据源，使用最近数据转换，示例

```java
Observable.from(integers1).switchIfEmpty(Observable.from(integers)).subscribe(mSubscriber);

结果:
onNext value: 0
onNext value: 1
onNext value: 2
onNext value: 3
onNext value: 4
onNext value: 5
onNext value: 6
onNext value: 7
onNext value: 8
onCompleted
```

##### zip
> 按数据合并两个数据源，示例

```java
Observable<Integer> evens = Observable.just(2, 4, 6, 8, 10);
        Observable<Integer> odds = Observable.just(1, 3, 5, 7, 9);
        Observable.zip(evens, odds, (v1, v2) -> v1 + " + " + v2 + " is: " + (v1 + v2)).subscribe(System.out::println);
        
结果:

2 + 1 is: 3
4 + 3 is: 7
6 + 5 is: 11
8 + 7 is: 15
10 + 9 is: 19        
```

#### Error Handling Operators
##### Catch
> 从错误中恢复，继续执行，示例

###### onErrorReturn
>出现错误时，返回一个特定的值，示例

```java
Observable.from(integers).map(integer -> integer / 0).onErrorReturn(throwable -> 0).subscribe(mSubscriber);

结果:
onNext value: 0
onCompleted
```

###### onErrorResumeNext, onExceptionResumeNext
>出现错误，构建另一个消息源，示例

```java
Observable.from(integers).map(integer -> integer / 0).onErrorResumeNext(throwable -> Observable.from(integers1)).subscribe(mSubscriber);

结果:
onNext value: 0
onNext value: -1
onNext value: -2
onNext value: -3
onNext value: -4
onNext value: -5
onNext value: -6
onNext value: -7
onNext value: -8
onCompleted
```

* 源码分析

```java
public void onError(Throwable e) {
    if (done) {
        Exceptions.throwIfFatal(e);
        RxJavaHooks.onError(e);
        return;
    }
    done = true;
    try {
        unsubscribe();

        Subscriber<T> next = new Subscriber<T>() {
            @Override
            public void onNext(T t) {
                child.onNext(t);
            }
            @Override
            public void onError(Throwable e) {
                child.onError(e);
            }
            @Override
            public void onCompleted() {
                child.onCompleted();
            }
            @Override
            public void setProducer(Producer producer) {
                pa.setProducer(producer);
            }
        };
        serial.set(next);

        long p = produced;
        if (p != 0L) {
            pa.produced(p);
        }

        Observable<? extends T> resume = resumeFunction.call(e);

        resume.unsafeSubscribe(next);
    } catch (Throwable e2) {
        Exceptions.throwOrReport(e2, child);
    }
}
```

* 出现错误，使用外部提供的方式产生新的消息源并发出

##### Retry
###### retry
> 出现错误，重新订阅并重试，示例

```java
Observable.from(integers).map(integer -> {
    int a = 1;
    if (integer == a) {
        throw new RuntimeException("aa");
    }
    return integer;
}).retry(1).subscribe(mSubscriber);

```
* ``retry``也可以提供选择器来决定是否重试

###### retryWhen
> 捕获错误，生成第二个消息源,并监听此消息源，如果此消息源发出消息，则重新订阅源消息，示例

```java
Observable.from(integers).map(integer -> {
            int a = 5;
            if (integer == a) {
                throw new RuntimeException("aa");
            }
            return integer;
        }).retryWhen(observable -> observable.zipWith(Observable.range(1, 3), (n, i) -> i).flatMap(i -> {
            System.out.println("delay retry by " + i + " second(s)");
            return Observable.timer(i, TimeUnit.SECONDS);
        })).toBlocking().forEach(System.out::println);
```

#### Observable Utility Operators
##### Delay
> 延迟发出消息，示例

```java
Observable.from(integers).delay(5, TimeUnit.SECONDS).toBlocking().forEach(System.out::println);
```

* 源码分析

```java
public void onNext(final T t) {
    worker.schedule(new Action0() {

        @Override
        public void call() {
            if (!done) {
                child.onNext(t);
            }
        }

    }, delay, unit);
}
```

* 线程调度延时

##### Do
>在可观察对象生命周期发生之前调用，示例

```java
Observable.from(integers).doAfterTerminate(() -> {
    System.out.println("terminate");
}).subscribe(mSubscriber);
```

##### Materialize/Dematerialize
> 对于发出的每个消息进行包装与拆包装

```java
Observable.from(integers).materialize().dematerialize().subscribe(mSubscriber);
```

##### ObserveOn
> 指定观察者在哪个线程观察结果

* 源码

```java
ScalarSynchronousObservable.java

public Observable<T> scalarScheduleOn(final Scheduler scheduler) {
    Func1<Action0, Subscription> onSchedule;
    if (scheduler instanceof EventLoopsScheduler) {
        final EventLoopsScheduler els = (EventLoopsScheduler) scheduler;
        onSchedule = new Func1<Action0, Subscription>() {
            @Override
            public Subscription call(Action0 a) {
                return els.scheduleDirect(a);
            }
        };
    } else {
        onSchedule = new Func1<Action0, Subscription>() {
            @Override
            public Subscription call(final Action0 a) {
                final Scheduler.Worker w = scheduler.createWorker();
                w.schedule(new Action0() {
                    @Override
                    public void call() {
                        try {
                            a.call();
                        } finally {
                            w.unsubscribe();
                        }
                    }
                });
                return w;
            }
        };
    }

    return create(new ScalarAsyncOnSubscribe<T>(t, onSchedule));
}
```

* 区分``EventLoopsScheduler``和其他调度器。如果是``EventLoopsScheduler``直接调度，否则创建调度器,由调度器运行接收到的消息。解除调度器订阅

```java
OperatorObserveOn.java

public Subscriber<? super T> call(Subscriber<? super T> child) {
    if (scheduler instanceof ImmediateScheduler) {
        // avoid overhead, execute directly
        return child;
    } else if (scheduler instanceof TrampolineScheduler) {
        // avoid overhead, execute directly
        return child;
    } else {
        ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
        parent.init();
        return parent;
    }
}
```

* 如果调度器是``ImmediateScheduler(即时调度器)``或者``TrampolineScheduler(串行调度器)``则立即执行。否则创建``ObserveOnSubscriber``

```java
ObserveOnSubscriber.java

protected void schedule() {
    if (counter.getAndIncrement() == 0) {
        recursiveScheduler.schedule(this);
    }
}

while (requestAmount != currentEmission) {
    boolean done = finished;
    Object v = q.poll();
    boolean empty = v == null;

    if (checkTerminated(done, empty, localChild, q)) {
        return;
    }

    if (empty) {
        break;
    }

    localChild.onNext(NotificationLite.<T>getValue(v));

    currentEmission++;
    if (currentEmission == limit) {
        requestAmount = BackpressureUtils.produced(requested, currentEmission);
        request(currentEmission);
        currentEmission = 0L;
    }
}
```

* 其中有个队列(``SpscAtomicArrayQueue``)，用于接收消息，开始循环从队列中取出消息，如果发送的消息数量和请求消息的数量一致，表示此次事件发送结束

##### Serialize
> 强制串行发送消息

##### SubscribeOn
>指定可观察对象在哪个线程上执行

* 源码分析

```java
inner.schedule(new Action0() {
    @Override
    public void call() {
        final Thread t = Thread.currentThread();

        Subscriber<T> s = new Subscriber<T>(subscriber) {
            @Override
            public void onNext(T t) {
                subscriber.onNext(t);
            }

            @Override
            public void onError(Throwable e) {
                try {
                    subscriber.onError(e);
                } finally {
                    inner.unsubscribe();
                }
            }

            @Override
            public void onCompleted() {
                try {
                    subscriber.onCompleted();
                } finally {
                    inner.unsubscribe();
                }
            }

            @Override
            public void setProducer(final Producer p) {
                subscriber.setProducer(new Producer() {
                    @Override
                    public void request(final long n) {
                        if (t == Thread.currentThread()) {
                            p.request(n);
                        } else {
                            inner.schedule(new Action0() {
                                @Override
                                public void call() {
                                    p.request(n);
                                }
                            });
                        }
                    }
                });
            }
        };

        source.unsafeSubscribe(s);
    }
});
```

* 开启线程调度，从当前线程中发出消息到下一级订阅者

##### TimeInterval
> 将源消息源发出的消息转换成表示消息与消息直接的时间间隔,示例

```java
Observable.from(integers).timeInterval().subscribe(mSubscriber);

结果
onNext value: TimeInterval [intervalInMilliseconds=2, value=0]
onNext value: TimeInterval [intervalInMilliseconds=1, value=1]
onNext value: TimeInterval [intervalInMilliseconds=0, value=2]
onNext value: TimeInterval [intervalInMilliseconds=0, value=3]
onNext value: TimeInterval [intervalInMilliseconds=0, value=4]
onNext value: TimeInterval [intervalInMilliseconds=0, value=5]
onNext value: TimeInterval [intervalInMilliseconds=0, value=6]
onNext value: TimeInterval [intervalInMilliseconds=0, value=7]
onNext value: TimeInterval [intervalInMilliseconds=0, value=8]
onCompleted
```

* 源码分析

```java
public void onNext(T args) {
    long nowTimestamp = scheduler.now();
    subscriber.onNext(new TimeInterval<T>(nowTimestamp - lastTimestamp, args));
    lastTimestamp = nowTimestamp;
}
```

* 当前时间减去上次接收到消息的时间，封装成``TimeIntegerval``

##### TimeOut
> 如果在一定时间内没有发出消息，发出``error``通知

##### Timestamp
> 给每个发出的消息带上时间戳,示例

```java
Observable.from(integers).timestamp().subscribe(mSubscriber);
```

##### Using
> 创建可分配的资源，其与可观察对象具有一样的生命周期

#### Conditional and Boolean Operators
##### All
>规定是否所有的消息发出，示例

```java
Observable.from(integers).all(integer -> integer < 5).subscribe(mSubscriber);

结果:
onNext value: false
onCompleted
```

##### Amb
>只发送多个消息源中最先产生消息的消息源，示例

```java
Observable.amb(Observable.from(integers).delay(2, TimeUnit.SECONDS), Observable.from(integers1).delay(1, TimeUnit.SECONDS))
                  .toBlocking()
                  .forEach(System.out::println);
                  
结果:
0
-1
-2
-3
-4
-5
-6
-7
-8  
```

##### Contain
> 发出的消息是否包含某消息，示例

```java
Observable.from(integers).contains(1).subscribe(mSubscriber);

结果:
onNext value: true
onCompleted
```

##### DefaultIfEmpty
> 如果没有发出任何消息，发出默认消息,示例

```java
Observable.empty().defaultIfEmpty(11).subscribe(mSubscriber);

结果:
onNext value: 11
onCompleted
```

##### SequenceEqual
> 两个消息源是否发出相同的消息,示例

```java
Observable.sequenceEqual(Observable.from(integers), Observable.from(integers)).subscribe(mSubscriber);

结果:
onNext value: true
onCompleted
```

##### SkipUntil, TakeUntil
> 丢弃/发送第一个消息源在第二个消息源发出消息之前发出的消息，示例

```java
Observable.from(integers).skipUntil(Observable.from(integers1).delay(1, TimeUnit.SECONDS)).subscribe(mSubscriber);

结果:
onCompleted
```

##### SkipWhile, TakeWhile
> 丢弃/发送消息直到某个条件变成false，示例

```java
Observable.from(integers).skipWhile(integer -> integer >= 0).subscribe(mSubscriber);

结果:
onCompleted
```

#### Mathematical and Aggregate Operators
##### Average
> 发出消息的平均值，示例

```java
MathObservable.from(Observable.from(integers)).averageInteger(integer -> integer).forEach(System.out::println);

结果：
4
```

##### Concat
> 拼接多个消息源，按顺序发送消息，示例

```java
Observable.concat(Observable.from(integers), Observable.from(integers1)).forEach(System.out::println);
```

##### Count
> 统计发出消息的数量，示例

```java
assertEquals(Integer.valueOf(integers.length), Observable.from(integers).count().toBlocking().single());
```

##### Max/Min
> 获取消息中最大/最小值，示例

```java
assertEquals(integers[integers.length - 1], MathObservable.from(Observable.from(integers)).max((o1, o2) -> o1 - o2).toBlocking().single());
```

##### Reduce, Sum
> 对每个发出的消息做操作，发送最终的值，示例

```java
Observable.from(integers).reduce((integer, integer2) -> integer + integer2).forEach(System.out::println);

结果:
36
```

#### Backpressure Operators

* Buffer, Sample, Debounce, Window

#### Connectable Observable Operators
##### Connect
> 连接``Observable``和``Subscribers``。可以指定连接几个``Subscriber``。只有当所有的``Subscriber``都订阅了才开始发送消息。示例

```java
	final AtomicInteger run = new AtomicInteger();

    final ConnectableObservable<Integer> co = Observable.defer(() -> Observable.just(run.incrementAndGet())).publish();

	//自动连接，生成可观察对象
    final Observable<Integer> source = co.autoConnect();

    assertEquals(0, run.get());

    TestSubscriber<Integer> ts1 = TestSubscriber.create();
    //订阅了一次
    source.subscribe(ts1);

    ts1.assertCompleted();
    ts1.assertNoErrors();
    ts1.assertValue(1);

    assertEquals(1, run.get());

    TestSubscriber<Integer> ts2 = TestSubscriber.create();
    //第二次订阅，这是无效的，因为默认只指定了一次订阅
    source.subscribe(ts2);

    ts2.assertNotCompleted();
    ts2.assertNoErrors();
    ts2.assertNoValues();

    assertEquals(1, run.get());
```

##### Publish
>将普通``Observable``转换成``ConnectableObservable``，示例

```java
final AtomicInteger counter = new AtomicInteger();
ConnectableObservable<String> o = Observable.create(new Observable.OnSubscribe<String>() {

    @Override public void call(final Subscriber<? super String> observer) {
        new Thread(() -> {
            counter.incrementAndGet();
            observer.onNext("one");
            observer.onCompleted();
        }).start();
    }
}).publish();

final CountDownLatch countDownLatch = new CountDownLatch(2);
o.subscribe(v -> {
    assertEquals("one", v);
    countDownLatch.countDown();
});

// subscribe again
o.subscribe(v -> {
    assertEquals("one", v);
    countDownLatch.countDown();
});

//连接，才能收到消息
final Subscription subscription = o.connect();
try {
    if (!countDownLatch.await(1000, TimeUnit.MILLISECONDS)) {
        fail("subscriptions did not receive values");
    }
    assertEquals(1, counter.get());
} finally {
    subscription.unsubscribe();
}
```

##### RefCount
>让``ConnectTableObservable``表现像普通``Observable``。示例

```java
final AtomicInteger subscribeCount = new AtomicInteger();
final AtomicInteger nextCount = new AtomicInteger();
final Observable<Integer> observable = Observable.from(integers)
                                                 .doOnSubscribe(subscribeCount::incrementAndGet)
                                                 .doOnNext(l -> nextCount.incrementAndGet())
                                                 .publish()
                                                 .refCount();
final AtomicInteger receivedCount = new AtomicInteger();
//订阅一次,doOnNext会执行
final Subscription s1 = observable.subscribe(l -> receivedCount.incrementAndGet());
//订阅第二次, doOnNext会执行
final Subscription s2 = observable.subscribe();
try {
    Thread.sleep(50);
} catch (InterruptedException e) {
}
s2.unsubscribe();
s1.unsubscribe();

System.out.println("onNext Count: " + nextCount.get());

assertEquals(nextCount.get(), receivedCount.get() * 2);

assertEquals(2, subscribeCount.get());

```

* 普通``Observable``和``ConnectTableObservable``区别在于，普通的``Observable``只要订阅，就会发送消息，而``ConnectTableObservable``会连接几个订阅者，如连接1各订阅者，但是订阅了两次，那么后面一次是不会执行的。

##### Replay, Cache
> 确保所有的观察者看到相同的消息时序，虽然它们订阅的时候，消息源可能已经开始发送消息了。示例

```java
final AtomicInteger counter = new AtomicInteger();
Observable<String> o = Observable.create(new Observable.OnSubscribe<String>() {

    @Override public void call(final Subscriber<? super String> observer) {
        new Thread(() -> {
            counter.incrementAndGet();
            System.out.println("published observable being executed");
            observer.onNext("one");
            observer.onCompleted();
        }).start();
    }
}).replay().autoConnect();

// we then expect the following 2 subscriptions to get that same value
final CountDownLatch latch = new CountDownLatch(2);

// subscribe once
o.subscribe(v -> {
    assertEquals("one", v);
    System.out.println("v: " + v);
    latch.countDown();
});

// subscribe again
o.subscribe(v -> {
    assertEquals("one", v);
    System.out.println("v: " + v);
    latch.countDown();
});

if (!latch.await(1000, TimeUnit.MILLISECONDS)) {
    fail("subscriptions did not receive values");
}
assertEquals(1, counter.get());
```

[ref](http://reactivex.io/documentation/operators.html)


#### [Custom Operator](https://github.com/ReactiveX/RxJava/wiki/Implementing-custom-operators-(draft))

#### 流程

##### 基本流程图

![Observable](./Observable.png)

* 流程分析:
  1. 通过类方法``create``构造``Observable``对象。``create``内部使用``RxJavaHook``方法来创建``OnSubscribe``。
  2. 调用``subscribe(Subscriber)``方法，内部会将``Subscriber``转换成``SafeSubscriber``。然后调用``RxJavaHooks.onObservableStart(Observable, OnSubscribe).call(subscriber)``。这其实就是使用第一步设置的``OnSubscribe``调用``call``。上诉流程中``RxJavaHooks``其实就是对整个流程的一些关键步骤做``hook``以便可以由后续操作。这是最基本的``RxJava``流程

##### 通用流程图

![Operator](./Operator.png)

* 流程分析
  1. 当调用``subscribe``时会调用离它最近的``OnSubscribe``，如果是``Operator``的话。那就会调用最近的``OnSubscribeLift``的``call``。这时``RxJavaHooks``会调用``onObserveLift``的``call``产生新的订阅者。父类``OnSubscribe``会调用这个新的订阅者，并通过``call``将这个订阅者传给父类``OnSubscribe``中，在其中给子订阅者设置生产者``setProducer``，这时生产者会调用``request``。然后会在这里将消息发给子订阅者。
  2. 线程调度: ``OperatorSubscribeOn``,将每个生产者产生的所有的数据单独放到一个``Runnable``当中运行。 ``OperatorObserveOn``则是使用队列。来一个消息往队列里面插入，并要求队列开始执行。典型的多生产者但但消费者模型。``事件产生``使用``subscribeOn``来切换线程。而且只有第一个``subscribeOn``会生效。而事件加工和消费使用``observeonOn``来切换线程。影响的是后续的``Subscriber``。

### Subject
> Subject即可做``Observable``，也可以做``Observer``.示例

```java
final TestScheduler scheduler = new TestScheduler();

scheduler.advanceTimeTo(100, TimeUnit.SECONDS);
final TestSubject<Object> subject = TestSubject.create(scheduler);
final Observer observer = mock(Observer.class);
subject.subscribe(observer);
subject.onNext(1);
scheduler.triggerActions();

verify(observer, times(1)).onNext(1);
```

* ``subject``作为``Observable``发出事件
* 默认有四种``Subject``

#### AsyncSubject
> 发出源消息的最后一个消息。必须在源消息发出``complete``之后。示例

```java
final AsyncSubject<Object> subject = AsyncSubject.create();
final Observer observer = mock(Observer.class);
subject.subscribe(observer);

subject.onNext("first");
subject.onNext("second");
subject.onNext("third");
//如果没有发出结束信号，那么AsyncSubject不会发出任何事件。反之，会发出最后一个事件
subject.onCompleted();

//observer收到最后一个事件
verify(observer, times(1)).onNext(anyString());
verify(observer, never()).onError(new Throwable());
//收到结束信号
verify(observer, times(1)).onCompleted();
```

#### BehaviorSubject
> 发送默认值之后发送剩余的事件 示例


```java
final BehaviorSubject<Object> subject = BehaviorSubject.create("default");
        final Observer observerA = mock(Observer.class);
        final Observer observerB = mock(Observer.class);
        final Observer observerC = mock(Observer.class);

final InOrder inOrder = inOrder(observerA, observerB, observerC);

subject.subscribe(observerA);
subject.subscribe(observerB);    

inOrder.verify(observerA).onNext("default");
inOrder.verify(observerB).onNext("default");
```

* ``default``事件会先于其他事件被发出

#### PublishSubject
>发送从订阅时刻起的数据。示例

```java
final PublishSubject<Object> subject = PublishSubject.create();
final Observer observer = mock(Observer.class);

subject.onNext("one");
subject.onNext("two");

//之后发送的消息，observer才能收到
subject.subscribe(observer);

subject.onNext("three");

verify(observer, never()).onNext("one");
verify(observer, never()).onNext("two");
verify(observer, times(1)).onNext("three");
```

#### ReplaySubject
> 发送所有的事件，不管什么时候订阅，与PublishSubject相反。示例

```java
final ReplaySubject<Object> subject = ReplaySubject.create();
final Observer observer = mock(Observer.class);

subject.onNext("one");
subject.onNext("two");

subject.subscribe(observer);

subject.onNext("three");

verify(observer, times(1)).onNext("one");
verify(observer, times(1)).onNext("two");
verify(observer, times(1)).onNext("three");
```

### Rxjava2.0
> 待续

