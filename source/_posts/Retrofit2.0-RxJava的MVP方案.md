---
title: Retrofit2.0 and RxJava
date: 2015-02-21 15:16:24
tags: Android
---

## 使用Retrofit2.0和RxJava的MVP方案

设计符合android(MVP)方案的网络架构很有挑战的。当然，更具有挑战的是该设计也还完全符合Android的生命周期。我们能够使用``Retrofit``，``Volley``,或者``AsyncTask``能构建出我们的MVP方案,但我们如何处理当屏幕旋转时的网络请求？

本文的目的是介绍一个能够很简单的使用MVP模式的网络框架，并且能够保证``Activity/Fragment``生命周期安全。本文需要``RxJava``和``Retrofit``的基本认识。

<!--more-->

## MVP

我们往下讲之前，简单的描述一下``Android``MVP是怎么样的？以及为什么使用它？

MVP方案的目的是消除``view`` 对它数据的依赖。这样允许三层各自独立测试，使程序更佳健壮。至于实现MVP的合理方式，有很多种。事实上，``Google``快速搜索下，你会看到很多相关的主题，它们各自都会有差异。
下面，我会描述``Android`` 分层的常见方式以及我如何定义各层的职责。
![](../image/mvp.png)

### 表现层

表现层是视图层和模型层的通讯桥梁。表现层的方法被视图层使用。表现层也有从模型层返回的数据逻辑以便视图层展示给用户。

表现层可能实现一个接口，这个接口有时被称为``Interactor``，里面有视图层需要使用的方法。表现层的职责是从模型层请求数据并处理。

### 视图层
视图通常是一个``activity``或者一个``fragment``.我们的视图有一个``Presenter``的引用，这个引用由视图初始化。视图的主要职责是用户与视图的交互。视图会有接受从``Presenter``发送给UI元素的方法。

###模型层
模型层是我们想要展示在视图层的数据源。我们的例子里，模型层是``NetworkService``，使用``POJO``。最好将模型层用这种方式构建，表现层不必知道数据使用硬盘，内存或者网络获取。

### 使用Retrofit2.0和RxJava来构建模型层
既然我们对MVP的实现方式已经有一个基本的认识，那我们开始对模型层来进行特殊改造。像我之前提到的，最大的挑战之一是设计一个能很好处理``Android``生命周期的网络架构。

为什么这么困难？我不想深入讨论``Android``的``activity/fragment``生命周期，简单的说就是，当用户旋转屏幕时，``activity/fragment``被销毁和重建。
如果你已经尝试过用(Retrofit, Volley, AsyncTasks或者其他方案)来实现标准的网络请求，你可能已经注意到了正常情况下都能正常使用，但当用户想要在网络请求过程中间旋转屏幕时。你的请求将无效，所有的``listeners/interfaces``被销毁。

我们大多数人都已经发现咋么一个问题，一些人可能在网络请求期间禁止旋转屏幕，设置了一系列的``services``和``receivers``或者创建一个``fragment``，它的实例一直在这个过程被保存。然而这些方案都有各自的弊端。我们将会提供一个简单的方案。

### Retrofit2.0和RxJava请求

首先添加它们的依赖

```
compile 'com.squareup.retrofit2:retrofit:2.0.0-beta3'
compile 'com.squareup.retrofit2:converter-gson:2.0.0-beta3'
compile 'com.squareup.retrofit2:adapter-rxjava:2.0.0-beta3'
compile 'io.reactivex:rxandroid:1.1.0'
```

如果你曾经用过``Retrofit``，你对请求的接口创建应该很熟悉了。我们的例子将使用一个不带任何参数或者请求体的简单的``Get``请求

```
public interface NetworkAPI {
     @GET("FriendLocations.json")
     Observable<friendresponse> getFriendObservable();
} 
```

注意我们返回的是一个``FriendResponse``类型的``Observable``。

下一步,我需要创建``Retrofit``实例。例子中，我选择使用``GsonConverterFactory``。使用``Retrofit2.0``时，我们需要包含``RxJavaCallAdapter``

```
Retrofit retrofit = new Retrofit.Builder()
     .baseUrl(baseUrl)
     .addConverterFactory(GsonConverterFactory.create())
     .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
     .build();

NetworkAPI networkAPI = retrofit.create(NetworkAPI.class);
```

现在我们可以创建``Observable``和发送请求了。

```
Observable<FriendResponse> friendResponseObservable = networkAPI.getFriendsObservable()
     .subscribeOn(Schedulers.newThread())
     .observeOn(AndroidSchedulers.mainThread());

friendResponseObservable.subscribe(new Observer<FriendResponse>(){
     @Override
     public void onCompleted(){}

     @Override
     public void onError(Throwable e){
          //handle error
     }

     @Override
     public void onNext(FriendResponse response){
          //handle response
     }
});
```

上诉代码从网络API接口创建了``FriendResponse``的``Observable``。我定义这个可观察对象在新的线程中工作，并将结果返回到主线程。  

调用``subscribe()``时发出请求。``subscribe()``方法返回``Subscription``对象，我们将在之后使用它来防止内存泄露。  

但是，上面方式并没有解决生命周期问题。一旦设备旋转或者``activity/fragment``被杀掉，被观察对象被销毁，我们就没有这个请求的引用，无法做出相应的响应。事实上，你将它与标准的``Retrofit``请求对比，我们反倒更复杂了。

幸运的是，这仅仅只是解决我们问题的第一步。

### 生命周期安全的请求
我们该如何解决生命周期问题呢？我们在``Application``创建一个单例。 

```
public class RxApplication extends Application {

     private NetworkService networkService;

     @Override
     public onCreate() {
          super.onCreate();
          networkService = new NetworkService();
     }

     public NetworkService getNetworkService(){
          return networkService;
     }
}
```

下一步将``Retrofit api``接口放到这个单例中。

```
public class NetworkService {

     private static String baseUrl = "https://dl.dropboxusercontent.com/u/5770756/"
     private NetworkAPI networkAPI;

     public NetworkService() {
          this(baseUrl);
     }

     public NetworkService(String baseUrl) {
          Retrofit retrofit = new Retrofit.Builder()
               .baseUrl(baseUrl)
               .addConverterFactory(GsonConverterFactory.create())
               .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
               .build();

          networkAPI = retrofit.create(NetworkAPI.class);
     }

     public NetworkAPI getAPI() {
          return networkAPI;
     }

     public interface NetworkAPI {
          @GET("FriendLocations.json")
          Observable<FriendResponse> getFriendObservable();
     }
}
```

最后，我们把被观察者的创建逻辑放到单例中。我们只有在缓存对象不在的时候才创建一个被观察对象。我们需要缓存被观察对象。

首先，我们找个地方存放我们可能需要复用的被观察者对象。我在单例内创建一个``LruCache``对象。

```
private LruCache<Class<?>, Observable<?>> apiObservables = new LruCache<>(10);
```

下一步，创建一个方法以便获得被观察对象。这个对象会返回给``presenter``对象。

```
public Observable<?> getPreparedObservable(Observable<?> unPreparedObservable, Class<?> clazz, boolean cacheObservable, boolean useCache){
     Observable<?> preparedObservable = null;

      if(useCache) //this way we don't reset anything in the cache if this is the only instance of us not wanting to use it.
          preparedObservable = apiObservables.get(class);

     if(preparedObservable!=null)
          return preparedObservable;

     //we are here because we have never created this observable before or we didn't want to use the cache...

     preparedObservable = unPreparedObservable.subscribeOn(Schedulers.newThread())
          .observeOn(AndroidSchedulers.mainThread());

     if(cacheObservable){
            preparedObservable = preparedObservable.cache();
            apiObservables.put(clazz, preparedObservable);
     }

     return preparedObservable;
}
```

上诉方法接收任何类型的被观察者对象和对应的类参数，这些东西被使用来存储被观察对象。

``boolean``参数决定我们是否想要缓存可观察对象或者复用缓存可观察对象。没必要总是缓存并复用可观察对象

如果缓存的被观察者没有被使用，那么我们将一个未准备的被观察者传给方法。为了合理的缓存被观察者，我们需要调用``cache()``将它加入``LruCache``.如果我们没有在加入``LruCache``之前调用``cache()``，那么被观察实例会被缓存，但请求/响应都不会。
我们的表现层已经准备好从模型层请求数据了，模型层将会返回一个含有数据的被观察者。

下面的方法，我们每次调用``subscribe()``将会复用任何已经创建的请求/响应.这就意味着如果响应已经返回，那么``onNext()``和``onComplete``将立即执行，请求将只做一次。

```
Observable<FriendResponse> friendResponseObservable = (Observable<FriendResponse>)service.getPreparedObservable(service.getAPI().getFriendsObservable(), FriendResponse.class, true, true);

subscription = friendResponseObservable.subscribe(new Observer<FriendResponse>() {
     @Override
     public void onCompleted(){ }

     @Override
     public void onError(Throwable e){
          view.showRxFailure(e);
     }

     @Override
     public void onNext(FriendResponse response) {
          view.showRxResults(response);
     }
});
```

### 捕获

当然，也有概率很小的异常。使用``RxJava``，如果我们不够小心，很容易导致订阅者泄露。如我之前提到的，当你订阅一个被观察者时，一个订阅对象返回。这个对象有两个方法，``isUnsubscribed()``和``unsubscribe()``。这允许我们在``activity/fragment``被销毁之前释放任何存在的``subscription``。

因此，我们需要在表现层的``onPause()``或者``onDestroy()``方法调用下述方法。

```
public void rxUnSubscribe(){
     if(subscription!=null && !subscription.isUnsubscribed())
           subscription.unsubscribe();
}
```

上述方法检查订阅对象是否存在，如果存在并已经订阅，那么将取消订阅。
我们也需要追踪一个请求已经发出以便我们能够在视图重新创建后重新订阅我们的缓存被观察者。我们在``savedInstanceState()``中做这件事。这允许表现层关心是否继续等待请求完成或者立即用缓存的响应来刷新视图。

### 结语
``Retrofit 2.0``和``RxJava``配合``MVP``模式使用一种伟大的组合。我们创建请求，但不用关心它的响应。这种方式也简化了视图层的工作流程。

[原文](http://www.captechconsulting.com/blogs/a-mvp-approach-to-lifecycle-safe-requests-with-retrofit-20-and-rxjava)