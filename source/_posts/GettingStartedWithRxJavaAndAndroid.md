---
title: ReativeX
date: 2015-02-20 15:16:24
tags: Android
---

### 什么是ReactiveX  
[ReactiveX](http://reactivex.io/)是一种API，主要关注使用观察者模式,迭代器模式和具有函数式编程特点的可观察的数据流或者事件的组合和操作。能够处理实时数据，具有高效，简洁，拓展性强的特点。使用观察者和操作者来操作它们，ReactiveX提供一种可组合和弹性API来创建和操作数据流，简化异步编程的一些顾虑，如线程创建和并发问题。
<!--more-->
### RxJava介绍  
[RxJava]()是开源的ReactiveX的Java实现.两个主要的类:``Observable``和``Subscriber``。``RxJava``中，被观察者是发出数据流或者事件流的类，订阅者是这些流的执行者。被观察者的标准流程是发出一个或多个流，然后是完全成功或者出错。一个被观察者能够有多个订阅者，对于被观察者发出的每个流，其将会被``Subscriber.onNext()``方法处理。一旦被观察者发布完了流，它将会调用``Subscriber.onCompleted()``方法，如果出错，那么会调用``Subscriber.onError()``。现在我们已经了解了被观察者和订阅者，我们继续介绍如何创建它们。

```
Observable integerObservable = Observable.create(new Observable.OnSubscribe(){ 
	@Override
	public void call(Subscriber subscriber) {
		subscriber.onNext(1);
		subscriber.onNext(2);
		subscriber.onNext(3);
		subscriber.onCompleted();
});
```

这个被观察者发出了数字1，2，3数据流，然后结束。现在我们需要创建一个订阅者来处理这些数据流。

```
Subscriber integerSubscriber = new Subscriber() {
	@Override
	public void onCompleted() {
		System.out.println("Complete!");
	}
	
	@Override
	public void onError(Throwable e) {
	
	}
	
	@Override
	public void onNext(Integer value) {
		System.out.println("onNext: " + value);
	}
};
```

订阅者将打印出所有被观察者发出的数据流以及通知我们流发送完成了。你可以将被观察者和订阅者用``Observable.subscribe``绑定。

```
observable.subscribe(subscriber);

output:
onNext 1
onNext 2
onNext 3
onCompleted
```

使用``Observable.just()``方法创建一个被观察者来发布定义的数据能够简化上述代码，把订阅者变成匿名内部类。得到如下代码：

```
Observable.just(1,2,3).subscribe(new Subscriber<Integer>() {
            @Override public void onCompleted() {
                System.out.print("onCompleted");
            }

            @Override public void onError(Throwable e) {

            }

            @Override public void onNext(Integer integer) {
                System.out.println("onNext " + integer);
            }
        });
```

### 操作符
创建和订阅被观察者很简单，但似乎没什么太大用处，但这仅仅只是``RxJava``的开始。任何的被观察者可以有自己的输出可以被操作符转换，能够同时使用多个操作符。例如，我们之前的被观察者，我们仅仅想发出奇数数据流。使用``filter()``操作符能够达到这个目的。

```
Observable.just(1, 2, 3, 4, 5, 6).filter(new Func1<Integer, Boolean>() {
            @Override public Boolean call(Integer integer) {
                return integer % 2 == 1;
            }
        }).subscribe(new Subscriber<Integer>() {
            @Override public void onCompleted() {
                System.out.print("onCompleted");
            }

            @Override public void onError(Throwable e) {

            }

            @Override public void onNext(Integer integer) {
                System.out.println("onNext " + integer);
            }
        });
        
Outputs:
onNext 1
onNext 3
onNext 5
onCompleted        
```

操作符``filter()``符会取走发出的数字，对奇数返回``true``，对偶数返回``false``，返回``false``的值不会发送给订阅者，我们不会看到它们的输出。注意``filter()``操作符返回一个被观察者，然后我们可以像之前一样订阅它。现在，我想要找出奇数的平方根。一种方式是在每个订阅者的``onNext()``中去计算。然而，这样子做的话，在以后可能无法进一步去转换数据。为了达到这个目的，我们使用``map()``操作符合``filter()``操作。

```
Observable.just(1, 2, 3, 4, 5, 6).filter(new Func1<Integer, Boolean>() {
            @Override public Boolean call(Integer integer) {
                return integer % 2 == 1;
            }
        }).map(new Func1<Integer, Object>() {
            @Override public Object call(Integer integer) {
                return Math.sqrt(integer);
            }
        }).subscribe(new Subscriber<Object>() {
            @Override public void onCompleted() {
                System.out.print("onCompleted");
            }

            @Override public void onError(Throwable e) {

            }

            @Override public void onNext(Object o) {
                System.out.println("onNext " + o);
            }
        });
        
//OutPuts
onNext 1.0
onNext 1.7320508075688772
onNext 2.23606797749979
onCompleted        
```

链式操作符是``RxJava``非常重要的一部分，它给予了你能够灵活的完成你想要做的任何事。带着被观察者和操作符是如何交互的这个问题，我们继续下一个主题: 应用``RxJava``到``Android``中。

### Android中的简单线程
在``Android``开发一个常见的场景是需要在后台进行一些工作，这些工作完成后，将结果通知给UI主线程来展示结果。在``Android``中，我们有几种方式可以实现这种操作，使用``AsyncTasks``, ``Loaders``,``Services``等。然而，这些方案通常都不是最好的选择。``AsyncTask``很容易导致内存泄露，``CursorLoaders``需要大量的配置，``Service``更倾向于长时间在后台运行的任务，而不是一些能够快速完成的任务如网络请求或者从数据库加载数据。使用``RxJava``能够这些问题。下面这个布局，有一个按钮来开始操作，有一个一直显示的进度条表明我们的操作一直在后台线程运行，而不是在UI线程。

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
   xmlns:app="http://schemas.android.com/apk/res-auto"
   android:id="@+id/root_view"
   android:layout_width="match_parent"
   android:layout_height="match_parent"
   android:fitsSystemWindows="true"
   android:orientation="vertical">

   <android.support.v7.widget.Toolbar
       android:id="@+id/toolbar"
       android:layout_width="match_parent"
       android:layout_height="?attr/actionBarSize"
       android:background="?attr/colorPrimary"
       app:popupTheme="@style/AppTheme.PopupOverlay"
       app:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar" />

   <Button
       android:id="@+id/start_btn"
       android:layout_width="wrap_content"
       android:layout_height="wrap_content"
       android:layout_gravity="center_horizontal"
       android:text="@string/start_operation_text" />

   <ProgressBar
       android:layout_width="wrap_content"
       android:layout_height="wrap_content"
       android:layout_gravity="center_horizontal"
       android:indeterminate="true" />

</LinearLayout>
```

一旦按钮被点击，按钮将被禁用并开始工作，工作完成后，``SnackBar``会显示，按钮重新能够使用。这儿有个简单的``AsyncTask``以及我们长时间的操作.

```
public String longRunningOperation() {
   try {
       Thread.sleep(2000);
   } catch (InterruptedException e) {
       // error
   }
   return "Complete!";
}

private class SampleAsyncTask extends AsyncTask {

   @Override
   protected String doInBackground(Void... params) {
       return longRunningOperation();
   }

   @Override
   protected void onPostExecute(String result) {
       Snackbar.make(rootView, result, Snackbar.LENGTH_LONG).show();
       startAsyncTaskButton.setEnabled(true);
   }
}
```

现在，我们如何把``AsyncTask``转变成``RxJava``? 首先，添加``RxJava, RxAndroid``的依赖。然后我们需要创建一个被观察者来调用我们的长时间操作。

```
final Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
            @Override public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext(longRunningOperation());
                subscriber.onCompleted();
            }
        });
```

我们创建的被观察者将调用``longRunningOperation``并将结果传给``onNext()``,然后调用``onCompleted()``。下一步，我们需要订阅我们的被观察者。

```
startRxOperationButton.setOnClickListener(new View.OnClickListener() {
   @Override
   public void onClick(final View v) {
       v.setEnabled(false);
       operationObservable.subscribe(new Subscriber() {
           @Override
           public void onCompleted() {
               v.setEnabled(true);
           }

           @Override
           public void onError(Throwable e) {}

           @Override
           public void onNext(String value) {
               Snackbar.make(rootView, value, Snackbar.LENGTH_LONG).show();
           }
       });
   }
});
```

在``RxJava``中默认是单线程，你将需要使用``observeOn()``和``subscribeOn()``方法来实现多线程。``RxJava``的被观察者使用调度器，如``Schedulers.io()``(用于阻塞I/O操作),``Schedulers.computation()``(computational work),和``Schedulers.newThread()``(创建先线程).然而，从``Android``角度，你可能会好奇如何在UI主线程执行代码呢？我们使用``RxAndroid``能够实现这个目的。``RxAndroid``对``RxJava``进行了轻量级的拓展，其为UI主线程提供了一个调度器，也可以在任何``Handler``中运行。新的调度器中，被观察者在我们创建后台线程之前被创建，然后将结果通知UI主线程。

```
final Observable operationObservable = Observable.create(new Observable.OnSubscribe() {
   @Override
   public void call(Subscriber subscriber) {
       subscriber.onNext(longRunningOperation());
       subscriber.onCompleted();
   }
})
       .subscribeOn(Schedulers.io()) // subscribeOn the I/O thread
       .observeOn(AndroidSchedulers.mainThread()); // observeOn the UI Thread
```

修改后的被观察者使用``Schedulers.io()``来订阅，将使用``AndroidSchedulers.mainThread()``在UI线程观察结果。先前所有的例子的被观察者会发出结果，我们需要其他的选项，用于当一个操作仅仅发出一个结果，然后就结束。``The Single``能够用来实现这种需求。

```
Subscription subscription = Single.create(new Single.OnSubscribe<Object>() {
            @Override public void call(SingleSubscriber<? super Object> singleSubscriber) {
                String value = longRunningOperation();
                singleSubscriber.onSuccess(value);
            }
        }).subscribeOn(Schedulers.io()).observeOn(Schedulers.io()).subscribe(new Action1<Object>() {
            @Override public void call(Object o) {

            }
        }, new Action1<Throwable>() {
            @Override public void call(Throwable throwable) {

            }
        });
```

当使用``Single``，只有一个``onSuccess``动作和一个``onError``动作。单例有不同的操作器集合，允许将单例转化成被观察者。例如，使用``Single.mergeWith()``操作器，两个或更多相同类型的单例能够被一起合并来创建一个被观察者，发出每个单例的结果给被观察者。

### 防止内存泄露
因为``AsyncTask``在``Activity/Fragment``的生命周期中很容易导致类存泄露。不幸的是，使用``RxJava``也不能很好的消除内存泄露，但能够比较简单的防止其发生。如果你从头到尾都看了代码，你可能已经注意到，当你调用``Observable.subscribe()``时一个订阅者对象被返回。订阅者类只有两个方法``unsubscribe()和isUnsubscribed()``。为了防止内存泄露，在``Activity/Fragment``的``onDestroy``中，检查``Subscription.isUnsubscribed()``是否已经取消订阅。取消订阅将停止发送数据流到订阅者，将允许垃圾回收器回收相关的对象，防止任何与``RxJava``的内存泄露。如果你正在处理多个观察者和订阅者，所有的订阅者对象能够被添加到CompositeSubscription，使用``CompositeSubscription.unsubscribe()``来同时取消所有的订阅。

### 结语
不错，``RxJava``为``Android``提供了一种可供选择的多线程处理方法。能够简化后台处理，刷新UI操作。然而``RxJava``要求使用者对它的特性有深入的理解才能更好的使用它，在它身上花的时间越多，回报越大。有关``RxJava``更深层次的主题，本文就不在加以描述，如热和冷被观察者，处理后退，Rx的``Subject``类。使用``RxJava``转化``AsyncTask``的相同代码能够在[github](https://github.com/alex-townsend/GettingStartedRxAndroid)上找到


[原文](http://www.captechconsulting.com/blogs/getting-started-with-rxjava-and-android)
