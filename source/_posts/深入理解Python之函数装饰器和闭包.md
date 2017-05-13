---
title: 深入理解Python之函数装饰器和闭包
tags: Python
date: 2017-05-06 16:58:34
---


### 前言

函数装饰器用于标识函数来增强功能。它很强大，掌握它需要理解闭包。本文主要讲述函数装饰器如何工作。

### 装饰器

#### 装饰器101


```python
@decorate
def target():
    print('running target()')

# the same
def target():
    print('running target()')
target = decorate(target)

```

* 装饰器就是以装饰的函数作为参数的函数调用

```python

def deco(func):
    def inner():
        print('running inner()')

    return inner


@deco
def target():
    print('running target()')

# result running innner()
```


#### Python什么时候执行装饰器

装饰器的一项关键功能是被装饰的函数定义之后立即被执行，通常在*import*的时候

```python
registry = []


def register(func):
    print('running register(%s)' % func)
    registry.append(func)
    return func


@register
def f1():
    print('running f1()')


@register
def f2():
    print('running f2()')


@register
def f3():
    print('running f3()')


if __name__ == '__main__':
    print('running main()')
    print('registry ->', registry)
    f1()
    f2()
    f3()

# result:
running register(<function f1 at 0x1022a8268>)
running register(<function f2 at 0x1022a82f0>)
running register(<function f3 at 0x1022a8378>)
running main()
registry -> [<function f1 at 0x1022a8268>, <function f2 at 0x1022a82f0>, <function f3 at 0x1022a8378>]
running f1()
running f2()
running f3()
```

* 装饰器在模块载入的时候执行，但被装饰的函数只有在显式触发时才会执行

### 变量作用域

```python
b = 6


def f1(a):
    global b
    print(a)
    print(b)
    # b is treat as local variable
    b = 9

```

* 在`f1(a)`中定于`b`，其会被当做本地变量，此时未赋值，直接引用会抛出异常
* 如果在函数体内加上`global`关键字，那么这个变量就表示引用的是全局变量

### 闭包

```python
def make_averager():
    series = []

    def averger(new_value):
        series.append(new_value)
        total = sum(series)
        return total / len(series)

    return averger

```

* *series*是自由变量，能被内部函数使用。闭包就是一种函数，其会保留自由变量的绑定，以便能够之后使用

```python
def make_averager_nonlocal():
    count = 0
    total = 0

    def averager(new_value):
        nonlocal count, total
        count += 1
        total += new_value
        return total / count

    return averager

```

* `nonlocal`是`Python3`引入的关键字，用于标识变量作为自由变量，当变量在函数内部需要被重新赋值使用

### 实现装饰器

```python
def clock(func):
    @functools.wraps(func)
    def clocked(*args, **kwargs):
        t0 = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - t0
        name = func.__name__
        arg_lst = []
        if args:
            arg_lst.append(', '.join(repr(arg) for arg in args))
        if kwargs:
            pairs = ['%s=%r' % (k, w) for k, w in sorted(kwargs.items())]
            arg_lst.append(', '.join(pairs))
        arg_str = ', '.join(arg_lst)
        print('[%0.8fs] %s(%s) -> %r' % (elapsed, name, arg_str, result))
        return result

    return clocked

@clock
def snooze(seconds):
    time.sleep(seconds)


@clock
def factorial(n):
    return 1 if n < 2 else n * factorial(n - 1)

```

### 标准库中的装饰器

#### functools.lru_cache

其实现内存优化，保存之前计算的结果，避免重复计算

```python

@functools.lru_cache()
@clock
def fibonacci(n):
    return n if n < 2 else fibonacci(n - 2) * fibonacci(n - 1)

# result 未加入lru_cache
[0.00000054s] fibonacci(0) -> 0
[0.00000082s] fibonacci(1) -> 1
[0.00008297s] fibonacci(2) -> 0
[0.00000038s] fibonacci(1) -> 1
[0.00000046s] fibonacci(0) -> 0
[0.00000041s] fibonacci(1) -> 1
[0.00001262s] fibonacci(2) -> 0
[0.00002402s] fibonacci(3) -> 0
[0.00012032s] fibonacci(4) -> 0
[0.00000041s] fibonacci(1) -> 1
[0.00000041s] fibonacci(0) -> 0
[0.00000044s] fibonacci(1) -> 1
[0.00001188s] fibonacci(2) -> 0
[0.00002324s] fibonacci(3) -> 0
[0.00000033s] fibonacci(0) -> 0
[0.00000041s] fibonacci(1) -> 1
[0.00001188s] fibonacci(2) -> 0
[0.00000039s] fibonacci(1) -> 1
[0.00000050s] fibonacci(0) -> 0
[0.00000038s] fibonacci(1) -> 1
[0.00001248s] fibonacci(2) -> 0
[0.00002375s] fibonacci(3) -> 0
[0.00004736s] fibonacci(4) -> 0
[0.00008152s] fibonacci(5) -> 0
[0.00021393s] fibonacci(6) -> 0

# result 加入lru_cache
[0.00000059s] fibonacci(0) -> 0
[0.00000067s] fibonacci(1) -> 1
[0.00006346s] fibonacci(2) -> 0
[0.00000114s] fibonacci(3) -> 0
[0.00007960s] fibonacci(4) -> 0
[0.00000084s] fibonacci(5) -> 0
[0.00009436s] fibonacci(6) -> 0

```

* *lru_cache*的加入，减少了一些重复的计算

### 带着*SingleDispatch*的通用函数

`Python3.4`中引入的`singledispatch`。其允许每个模块定制总体方案，为类提供特定的函数

```python
@singledispatch
def htmlize(obj):
    content = html.escape(repr(obj))
    return '<pre>{}</pre>'.format(content)


@htmlize.register(str)
def _(text):
    content = html.escape(text).replace('\n', '<br>\n')
    return '<p>{0}</p>'.format(content)


@htmlize.register(numbers.Integral)
def _(n):
    return '<pre>{0} (0x{0:x})</pre>'.format(n)


@htmlize.register(tuple)
@htmlize.register(abc.MutableSequence)
def _(seq):
    inner = '</li>\n<li>'.join(htmlize(item) for item in seq)
    return '<ul>\n<li>' + inner + '</li>\n</ul>'

```

* *singledispatch* 定义了通用模板，然后可以使用特定的方法定制化

### 堆放装饰器

```Python
@d1
@d2
def f():
    print('f')

# same to 
def f():
    print('f')
f = d1(d2(f))

```

### 接收多个参数的装饰器

```Python
def register(active=True):
    ...
    
@register(active=True)
def f1():
    print('running f1()')

```