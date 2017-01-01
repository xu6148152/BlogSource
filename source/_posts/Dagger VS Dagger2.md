---
title: Dagger VS Dagger2
date: 2015-02-05 15:16:24
tags: Android
---

## Dagger
[dagger](https://github.com/square/dagger)是square开源的JAVA[依赖注入](https://en.wikipedia.org/wiki/Dependency_injection)框架，为何我们要使用[依赖注入](https://en.wikipedia.org/wiki/Dependency_injection)，答案在[这里](http://stackoverflow.com/questions/130794/what-is-dependency-injection)，简而言之依赖注入就是为某个对象提供它所需要的东西（它的依赖），而不是由依赖自己构造，这对于测试很重要。
当然依赖注入不是仅仅只适用于测试，它同样也能很简单的创建可复用，高灵活性的模块。能够在整个应用程序中分享模块,也可以根据开发环境选择运行的模块。

### 使用Dagger
##### 声明依赖
dagger会构造应用程序类的实例并可以提供它所需的依赖。使用``javax.inject.Inject annotation``来定义那些构造方法和字段是应用程序需要的。

使用``@Inject``来注解构造方法来表示Dagger需要构造的类。当需要一个新的实例，Dagger会获取需要的参数并调用相应的构造方法。

```  
class Thermosiphon implements Pump {
  private final Heater heater;

  @Inject
  Thermosiphon(Heater heater) {
    this.heater = heater;
  }
}
```

Dagger能直接注入字段。这个例子获得一个``Header``实例和一个``Pump``实例

```
class CoffeeMaker {
  @Inject Heater heater;
  @Inject Pump pump;
}
```

如果你的类有``@Inject``的字段，但没有``@Inject``的构造方法,Dagger将会使用无参构造方法，类如果缺少``@Inject``将无法被Dagger构造.Dagger不支持方法注入。

### 满足依赖关系  
默认情况下，Dagger通过构造实例来满足如上所述的每个依赖.如果你需要一个``CoffeeMaker``,Dagger将通过``new CoffeeMaker()``并设置可注入的字段来获取这个实例。

但``@Inject``并不是随处都能有效

* 接口不能被构造  
* 第三方库的类不能被注解  
* 可配置的对象一定要配置！

对于上述情况``@Inject``是无法满足的，使用``@Provides``来注解方法满足一个依赖.方法的返回值类型定义了需要的依赖。
例如,当需要``Heater``时，``provideHeater()``会被调用。

```
@Provides Header provideHeader() {
	return new ElectricHeader();
}
```


对于``@Provides``来获取他们自己的依赖.当需要``Pump``时，也需要一个``Thermosiphon ``  

所有的``@Provides``方法必须属于一个模块.这些类有``@Module``

```
@Module
class DripCoffeeModule {
	@Provides Heater provideHeater() {
    	return new ElectricHeater();
  	}
  	
  	@Provides Pump providePump(Thermosiphon pump) {
    	return pump;
  	}
}
```

通常,``@Provides``方法以provide为前缀，模块的类以Module为后缀

#### 构建视图
含有``@Inject``和``@Provides``的类是对象的视图,被它们的依赖链接。通过``ObjectGraph.create()``来获取这个视图,这个视图可以接受一个或者多个modules:

```
ObjectGraph objectGraph = ObjectGraph.create(new DripCoffeeModule());
```

为了使用视图，我们需要启动注入.这通常需要注入到控制台程序的主入口类里或者是``android``中的``activity``。在``coffee``例子中，``CoffeeApp``开始依赖注入。我们要求视图提供一个被注入类的实例:

```
class CoffeeApp implements Runnable {
  @Inject CoffeeMaker coffeeMaker;

  @Override public void run() {
    coffeeMaker.brew();
  }

  public static void main(String[] args) {
    ObjectGraph objectGraph = ObjectGraph.create(new DripCoffeeModule());
    CoffeeApp coffeeApp = objectGraph.get(CoffeeApp.class);
    ...
  }
}
```

剩下的问题就是``CoffeeApp``无法被视图识别，我们需要显式的将它注册到``@Module``模块当中

```
@Module(
    injects = CoffeeApp.class
)
class DripCoffeeModule {
  ...
}

```

注入的选项在编译期被验证。这能够将问题前移到项目开发早期(重构)
既然视图已经被创建，根对象也已经注入，我们能开始运行``coffee maker app``了。

#### @Singletons
在``@Provides``方法或者可注入的类使用``@Singleton``.视图将会使用这个类对象作为单例。

```
@Provides @Singleton Heater provideHeater() {
  	return new ElectricHeater();
}
```

在一个可注解的类使用``@Singleton``能被作为文档.它能提醒维护者知道这个类可能被多线程共享

```
@Singleton
class CoffeeMaker {
  ...
}
```

#### 懒注入  
有时你会需要一个对象延后初始化。你可以延迟创建``Lazy<T>``直到``Lazy<T>``的``get()``方法被首次调用。如果T是单例，在ObjectGraph中的``Lazy<T>``都会是同一个实例。否则都会创建各自的``Lazy<T>``实例.

```
class GridingCoffeeMaker {
  @Inject Lazy<Grinder> lazyGrinder;

  public void brew() {
    while (needsGrinding()) {
      // Grinder created once on first call to .get()		and cached.
      lazyGrinder.get().grind();
    }
  }
}
```

#### Provider的注入
有时你需要获得多个实例。你有几种选择(工厂,建造者等)一种选择是注入``Provider<T>``而不是``T``.当``.get()``被调用时，一个``Provider<T>``创建一个``T``实例

```
class BigCoffeeMaker {
  @Inject Provider<Filter> filterProvider;

  public void brew(int numberOfPots) {
    ...
    for (int p = 0; p < numberOfPots; p++) {
      maker.addFilter(filterProvider.get()); //new filter every time.
      maker.addCoffee(...);
      maker.percolate();
      ...
    }
  }
}
```

#### 限定符
有时单独的类型无法满足定义一个依赖。例如一个``sophisticated coffee maker`` app想分离加热器的水喝托盘.
在这种情况下，我们添加了一个限定符注解。任何注解都有自己的一个``@Qualifier``注解

```
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {
  String value() default "";
}
```

你能够创建你自己的限定符注解，或者直接使用``@Named``.

```
class ExpensiveCoffeeMaker {
  @Inject @Named("water") Heater waterHeater;
  @Inject @Named("hot plate") Heater hotPlateHeater;
  ...
}
```

```
@Provides @Named("hot plate") Heater provideHotPlateHeater() {
  return new ElectricHeater(70);
}

@Provides @Named("water") Heater provideWaterHeater() {
  return new ElectricHeater(93);
}
```

#### 静态注入(不常用)

```
@Module(
    staticInjections = LegacyCoffeeUtils.class
)
class LegacyModule {
}
```

使用``ObjectGraph.injectStatics()``来赋值这些静态字段

#### 编译期验证
Dagger包含了注解处理器能否验证``modules``和``injections``.这个处理器很严格，如果有任何不合理的绑定都会导致编译器错误.例如这个``Module``缺少了执行器的绑定:

```
@Module
class DripCoffeeModule {
  @Provides Heater provideHeater(Executor executor) {
    return new CpuHeater(executor);
  }
}
```

可以通过为为Executor方法加上``@Provides``,或者标记这个``module``为``incomplete``.未完成的模块是可以缺少依赖。

```
@Module(complete = false)
class DripCoffeeModule {
  @Provides Heater provideHeater(Executor executor) {
    return new CpuHeater(executor);
  }
}
```

模块使用被注入类中没有的类型会触发错误。

```
@Module(injects = Example.class)
class DripCoffeeModule {
  @Provides Heater provideHeater() {
    return new ElectricHeater();
  }
  @Provides Chiller provideChiller() {
    return new ElectricChiller();
  }
}
```

上述例子Example里仅仅只是注入了``Heater``而没有``Chiller``
如果你的模块的绑定将会被外部使用，可以标记这个模块为``library``

```
@Module(
  injects = Example.class,
  library = true
)
class DripCoffeeModule {
  @Provides Heater provideHeater() {
    return new ElectricHeater();
  }
  @Provides Chiller provideChiller() {
    return new ElectricChiller();
  }
}
```

为了使编译期验证能够通过，创建一个模块时，可以包含所有项目中的模块。注解处理器将会检测到问题，并报告。

```
@Module(
    includes = {
        DripCoffeeModule.class,
        ExecutorModule.class
    }
)
public class CoffeeAppModule {
}
```

#### 编译期代码生成
Dagger的注解处理器可能会生成源文件，这些源文件的名字可能像``$InjectAdapter.java``或者``DripCoffeeModule$ModuleAdapter``.这些文件是Dagger的详细实现，你不应该需要直接使用它。虽然它们能够被逐步调试。

#### 模块复写
如果同样的依赖有多个竞争的``@Provides``方法,Dagger将会编译出错。但有时使用测试代码来替换产品代码是很有必要的。使用``@overrides = true``在模块注解中,能够用其他模块覆盖当前模块。

```
public class CoffeeMakerTest {
  @Inject CoffeeMaker coffeeMaker;
  @Inject Heater heater;

  @Before public void setUp() {
    ObjectGraph.create(new TestModule()).inject(this);
  }

  @Module(
      includes = DripCoffeeModule.class,
      injects = CoffeeMakerTest.class,
      overrides = true
  )
  static class TestModule {
    @Provides @Singleton Heater provideHeater() {
      return Mockito.mock(Heater.class);
    }
  }

  @Test public void testHeaterIsTurnedOnAndThenOff() {
    Mockito.when(heater.isHot()).thenReturn(true);
    coffeeMaker.brew();
    Mockito.verify(heater, Mockito.times(1)).on();
    Mockito.verify(heater, Mockito.times(1)).off();
  }
}
```

## Dagger2
Dagger2是由Google基于Dagger的基础开发的依赖注入框架.它与Dagger有什么不同呢？主要不同在于Dagger2将Dagger中的``Graph``替换成了``Component``

#### 构建视图
在Dagger2中，视图是一个包含多个不带参数的方法的接口,这些方法返回相应的类型.在接口上使用``@Componet``并传入``module``类型作为模块的参数，之后Dagger2会完全生成这个接口的实现代码。

```
@Component(modules = DripCoffeeModule.class)
interface CoffeeShop {
  CoffeeMaker maker();
}
```

实现有与接口一样的名字，前缀是Dagger.通过调用``builder()``获得一个实例

```
CoffeeShop coffeeShop = DaggerCoffeeShop.builder()
    .dripCoffeeModule(new DripCoffeeModule())
    .build();
```

```
class Foo {
  static class Bar {
    @Component
    interface BazComponent {}
  }
}
```

上述的``@Component``将会生成一个名为``DaggerFoo_Bar_BazComponent``的组件。


Any module with an accessible default constructor can be elided as the builder will construct an instance automatically if none is set. And for any module whose @Provides methods are all static, the implementation doesn't need an instance at all. If all dependencies can be constructed without the user creating a dependency instance, then the generated implementation will also have a create() method that can be used to get a new instance without having to deal with the builder.

```
CoffeeShop coffeeShop = DaggerCoffeeShop.create();
```

现在``CoffeeApp``能简便的使用Dagger生成的``CoffeeShop``实现代码来拿到一个完全注入的``CoffeeMaker``

```
public class CoffeeApp {
  public static void main(String[] args) {
    CoffeeShop coffeeShop = DaggerCoffeeShop.create();
    coffeeShop.maker().brew();
  }
}
```

#### 绑定视图
上述例子展示了如何构建一个带有额外类型绑定的组件,但有各种机制来绑定视图。

 * 那些直接通过``@Component.modules``或者通过``@Module.includes``引用的模块当中的方法
 * 任何带有``@Inject``构造器的类型或者带有``@Scope``
 * 组件提供了组件依赖的方法
 * 组件自己
 * 任何包含子组件的建造者
 * 以上绑定类型中的``Provider``或``Lazy``


#### 单例和作用域绑定

```
@Provides @Singleton static Heater provideHeater() {
  return new ElectricHeater();
}
```

```
@Singleton
class CoffeeMaker {
  ...
}
```

Dagger2将组件实现的实例与视图中的作用域实例联系起来了。组件需要声明他们想要对外展示的作用域。

```
@Component(modules = DripCoffeeModule.class)
@Singleton
interface CoffeeShop {
  CoffeeMaker maker();
}
```

Dagger支持方法注入  
Dagger2不支持静态注入  
Dagger2不支持Override

....其他的几乎都一样了。

## 总结
Dagger1中视图是由``ObjectGraph ``通过反射组成。而dagger2是由``@Component``,用户定义类型在编译期生成的。从``ObjectGraph``到``Component``的转变是Dagger1和Dagger2最大的不同。

```
@Component(
  modules = {
    ModuleOne.class,
    ModuleTwo.class,
    ModuleThree.class,
  }
)
interface MyComponent {
  /* Functionally equivalent to objectGraph.get(Foo.class). */
  Foo getFoo();
  /* Functionally equivalent to objectGraph.get(Bar.class). */
  Bar getBar();
  /* Functionally equivalent to objectGraph.inject(aBaz). */
  Baz injectBaz(Baz baz);
}
```





