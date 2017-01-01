---
title: Google Agera vs. ReactiveX
date: 2016-04-27 14:48:15
tags:
---

[原文](http://akarnokd.blogspot.com/2016/04/google-agera-vs-reactivex.html)

### 介绍
如果你经常关注``Android``开发动态的话，或者关注``Reactive``相关的动态,最近``Google``有个重大发布。它们发布了针对于``Android``的反应式编程库:``Agera``。

``By Google``是``Google``一个从事``Google Play Movies``的小组。当然，这听起来更像``Google``.

不管是谁发布的，我们要关注的是，它与现有的反应式编程库``RxJava``,``Reactor``以及``Akka-Streams``有什么区别

### 核心API
``Agera``基于少值观察者模式:被观察者通过``update()``拿到更新和变化的信号.然后响应这些变化来算出什么改变了。以下是一个无参反应式数据流，它依赖一端``update``

```
interface Updatable {
    void update();
}
 
interface Observable {
   void addUpdatable(Updatable u);
   void removeUpdatable(Updatable u);
}
```

它们看起来挺合理的？不幸的事，它们也有``java.util.Observable``和其他基于``addListener/removeListener``的反应式API

### Agera Observable
这对方法的问题是每个添加了``Updatable``行为的``Observable``都不得不去记住原始的``Updatable``以便能移除同样的``Updatable``:

```
public final class DoOnUpdate implements Observable {
    final Observable source;
 
    final Runnable action;
 
    final ConcurrentHashMap<Updatable, DoOnUpdatable> map;
 
    public DoOnUpdate(Observable source, Runnable action) {
         this.source = source;
         this.action = action;
         this.map = new ConcurrentHashMap<>();
    }
 
    @Override
    public void addUpdatable(Updatable u) {
        DoOnUpdatable wrapper = new DoOnUpdatable(u, action);
        if (map.putIfAbsent(u, wrapper) != null) {
            throw new IllegalStateException("Updatable already registered");
        }
        source.addUpdatable(wrapper);
    }
 
    public void removeUpdatable(Updatable u) {
        DoOnUpdatable wrapper = map.remove(u);
        if (wrapper == null) {
            throw new IllegalStateException("Updatable already removed");
        }
        source.removeUpdatable(wrapper);
    }
 
    static final class DoOnUpdatable {
        final Updatable actual;
 
        final Runnable run;
 
        public DoOnUpdatable(Updatable actual, Runnable run) {
            this.actual = actual;
            this.run = run;
        }
 
        @Override
        public void update() {
            run.run();
            actual.update();
        }
    }
}
```

这导致一个争论点，在管道的每个阶段独立的下游``Updatabels``。是的，``RxJava's Subjects``和``ConnectableObservables``也有类似的争议点，但它们之后的链式操作符不会有争议。不幸的是，反应式流规范，在当前版本，``Publishers``也有类似的问题。现在``RxJava2.x``,``Rsc``和``Reactor``完全忽略了这些东西,结果是操作起来变得更严格了。

第二个微不足道的问题是你不能多次添加相同的``Updatable``。因为你无法在不同的``subscriptions``中通过``Map``区分它们，如果这么干，会出现异常。通常这是很少发生的，因为大多数的终端需求都很单一。

第三个比较大的问题：当``Updatable``不再注册在``Observable``时抛出。这在终端用户触发移除操作时，而这个操作又引发别的操作，它们当中一个会抛出异常。这就是为何现在反应式变成不能够被取消的缘故。

第四个理论问题,``addUpdatable``和``removeUpdatable``会相互竞争，一些下游的操作符想在某个上游操作符已经调用了``addUpdatable``之后断开。一个可能的问题是``removeUpdate``调用抛出异常而``addUpdatable``成功，这回导致信号流向任何地方和导致相关的对象不想要的持有。

### Agera Updatable
我们来从消费者角度看看API。``Updatable``是一个单一函数接口的方法，这让它能简单的给``Observable``添加监听

```
Observable source = ...
source.addUpdatable(() -> System.out.println("Something happened"));
```

相当简单，现在我们移除我们的监听

```
source.removeUpdatable(() -> System.out.println("Something happened");
```

这会产生一个异常:两个``lambdas``不是相同的对象。这在基于``addListener/removeListener``的API当中是相当常见的问题。解决方案是存储``lambda``，当需要的时候去用它：

```
Updatable u = () -> System.out.println("Something happened");
 
source.addUpdatable(u);
 
// ...
 
source.removeUpdatable(u);
```

有点不方便，但不会更糟了。如果你有很多``Observables``和很多``Updatables``呢？你需要记住谁注册在谁上，在相同的字段保持引用它们。``Rx.NET``的原始设计有个好主意来减少这种必须的单引用：

```
interface Removable extends Closeable {
    @Override
    void close(); // remove the necessity of try-catch around close()
}
 
public static Removable registerWith(Observable source, Updatable consumer) {
    source.addUpdatable(consumer);
    return () -> source.removeUpdatable(consumer);
}
```

当然，我们在调用``close()``时同样需要考虑：

```
public static Removable registerWith(Observable source, Updatable consumer) {
    source.addUpdatable(consumer);
    final AtomicBoolean once = new AtomicBoolean();
    return () -> {
        if (once.compareAndSet(false, true)) {
            source.removeUpdatable(consumer);
        }
    });
}
```

#### Agera MutableRepository
如果值发生变化，``Agera MutableRepository``会通过``update()``通知注册的``Updatables``。这和``BehaviorSubject``很像，区别是新值如果不调用``get()``不会流到消费者:

```
MutableRepository<integer> repo = Repositories.mutableRepository(0);
 
repo.addUpdatable(() -> System.out.println("Value: " + repo.get());
 
new Thread(() -> {
    repo.accept(1);
}).start();
```

当通过工厂方法创建仓库时，``Looper``中会调用``update()``。

```
Set<Integer> set = new HashSet<>();
 
MutableRepository<integer> repo = Repositories.mutableRepository(0);
 
repo.addUpdatable(() -> set.add(repo.get()));
 
new Thread(() -> {
    for (int i = 0; i < 100_000; i++) {
        repo.accept(i);
    }
}).start();
 
Thread.sleep(20_000);
 
System.out.println(set.size());
```

20秒之后``Set``最终会有多大？有人可能会认为是100,000。事实上，值可能在1~100,000之间!原因是``accept()``和``get()``并发运行，如果消费者运行比较慢，``accept()``会覆盖当前仓库的值。

#### 错误处理
使用异步通常意味着会碰到异步相关的问题。``RxJava``和其他类似的东西都会有这种问题：出现某种错误了，这个流程自动清掉，而开发者希望的是能够重新开始。错误和清理在某些情况下很复杂，成熟的库都花费了大量的精力来解决这些问题，因此开发者不需要在这上面浪费很多时间。

``Agera``基础API不会自己处理错误，你需要自己对错误进行处理。如果你用``Agera``组成多个服务，你需要建立自己的错误处理框架.由于并发和中断状态使它处理起来很笨重和延迟。

#### 终结
``Agera``对一个完成流不会发出通知，你需要自己知道它什么时候会完成。这在简单用户界面上不会有什么问题。然而，后台异步操作需要知道要发出多少信号，没有数据的话，你如何收到``update()``相关通知

### 如何设计现代化的无参反应式API
看看以下例子：

```
rx.Observable<Void> signaller = ...
 
rx.Observer<Void> consumer = ...
 
Subscription s = signaller.subscribe(consumer);
 
// ...
 
s.unsubscribe();
```

你很简单的就能获得它该有的功能。你如果要处理别的类型信号，将``Void``替换成你想要的类型就可以了。

如果一个库由于需要学习很多操作符而让人感到很笨重的话，你可以拷贝它，删除你不需要的东西。当然，你需要更新修复问题和性能优化后的代码。

如果拷贝和修改听起来不够有吸引力，你能按照``Reactive-Streams``规定开发自己的库;``Publisher<Void>``,``<Subscriber<Void>``和其他东西。你能很方便的与其他的``Reactive-Streams``库一起工作，你能通过它的兼容性测试来测试你的库。

当然，编写反应式库很麻烦，按照``Reactive-Streams``编写更麻烦。所以，综合考虑，你可以拓展API

如果你真的想要编写无参反应式流，这有几个你需要考虑的建议：

**1)不要分开``addListener``和``removeListener``。**单一的入口简化中间操作的开发。

```
interface Observable {
    Removable register(Updatable u);
}
 
interface Removable {
    void remove();
}
```

**2)考虑支持取消和移除而不是返回一个可以取消的令牌或者可移除的动作：**

```
interface Observable {
    void register(Updatable u);
}
 
interface Updatable {
    void onRegister(Removable remover);
    void update();
}
 
// or
 
interface Updatable {
    void update(Removable remover);
}
```

**3)考虑至少添加一个错误信号接收：**

```
interface Updatable {
    void onRegister(Removable remover);
    void update();
    void error(Throwable ex);
}
```

**4)考虑提供队列的异步操作.**

### 结论
其实作者就是来黑``Agera``的。