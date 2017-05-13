---
title: 深入理解Python之对象引用，修改，回收
tags: Python
date: 2017-05-13 16:58:50
---


## 变量不是盒子

`Python`变量就像`Java`的引用变量。`Python`变量应该当成是对象的标签

## 定义，相等，别名

```Python
charles = {'name': 'Charles L. Dodgson', 'born': 1832}

lewis = charles
print(lewis is charles)

print(id(charles), id(lewis))

lewis['balance'] = 950
print(charles)

alex = {'name': 'Charles L. Dodgson', 'born': 1832, 'balance': 950}
print(alex == charles)
print(alex is charles)

```

* `charles`和`lewis`引用的是相同的对象
* `alex`对象的内容与`charles`相同，但不是引用相同的对象,`Python`的`==`，比较的是对象的属相。`is`操作符比较的是对象的`id`，对象的`id`是创建时就确定了，永远不会改变了。

### ==和is

`==`操作符比较对象的值

判断对象是否为`None`，更合理的方式是

```Python
x is not None
```

`is`操作符速度比`==`快，因为它不能被重载，因此`Python`不用寻找特殊的方法来计算它。只需要简单的比较它们的`id`就可以了。对于`a==b`，其实背后使用的是`a.__eq__(b)`,`__eq__`是从`Object`继承用于比较对象的`id`，因此效果和`is`一样.但是很多内置的类型重载了`__eq__`，做了更多的操作比较对象属性的值。因此`==`可能会产生额外的计算。

### Tuple相对不可修改

元组跟大多数的`Python`的集合--`lists,dicts,sets`一样，都是持有对象们的引用。如果引用类型和修改，那么元组自身不可修改，而内部条目可以修改。

```Python
t1 = (1, 2, [30, 40])
t2 = (1, 2, [30, 40])
print(t1 == t2)
print(id(t1[-1]))
t1[-1].append(99)
print(t1)
print(t1 == t2)
```

* `t1`是不可修改的，`t[-1]`可修改
* `t1`和`t2`的内容一样。修改`t2`的条目的内容，这个时候`t1`和`t2`不相同了

## 默认浅拷贝

```Python
l1 = [3, [55, 44], (7, 8, 9)]
l2 = list(l1)
print(l2)
print(l1 == l2)
print(l1 is l2)

l3 = l1[:]
```

* `l2`由`l1`拷贝，值相等，但对象不相同
* `l3`由`l1`拷贝
* 两种方式的拷贝都属于浅拷贝。

```Python
l1 = [3, [66, 55, 44], (7, 8, 9)]
l2 = list(l1)
l1.append(100)
l1[1].remove(55)
print('l1:', l1)
print('l2:', l2)
l2[1] += [33, 22]
l2[2] += (10, 11)
print('l1:', l1)
print('l2:', l2)
```

* `l2`是`l1`的浅拷贝。`l1[1]`移除55。这也会影响到`l2`

### 任意对象的深浅拷贝

```Python

class Bus:
    def __init__(self, passengers=None):
        if passengers is None:
            self.passengers = []
        else:
            self.passengers = list(passengers)

    def pick(self, name):
        self.passengers.append(name)

    def drop(self, name):
        self.passengers.remove(name)


import copy

bus1 = Bus(['Alice', 'Bill', 'Claire', 'David'])
bus2 = copy.copy(bus1)
bus3 = copy.deepcopy(bus1)

print(id(bus1), id(bus2), id(bus3))

bus1.drop('Bill')
print(bus1.passengers)
print(bus2.passengers)
print(id(bus1.passengers), id(bus2.passengers), id(bus3.passengers))
print(bus3.passengers)

```

* `bus2`是`bus1`的浅拷贝,`bus3`是`bus1`的深拷贝

## 删除和垃圾回收

`del`表达式仅删除名字，不会删除对象。执行`del`表达式之后，对象可能会被回收。但只有在删除了最后一个引用对象的变量，才会被回收掉。`CPython`垃圾回收的基本算法是引用计数。`CPython2.0`，引入了新的垃圾回收算法，可以删除循环引用的对象组

```Python
s1 = {1, 2, 3}
s2 = s1


def bye():
    print('Gone with the wind')


ender = weakref.finalize(s1, bye)
print(ender.alive)
del s1
print(ender.alive)
s2 = 'spam'
print(ender.alive)

```

* `s1`,`s2`指向相同的对象
* 使用`weakref`来监控`s1`的状态
* 执行`del s1`,对象还被`s2`引用没有被回收
* 改变`s2`的引用，这个时候对象被回收

## 弱引用

弱引用不增加引用计数。弱引用不阻止垃圾回收

```Python
a_set = {0, 1}
wref = weakref.ref(a_set)
print(wref)
print(wref())

a_set = {2, 3, 4}
print(wref())
print(wref() is None)
print(wref() is None)

```

### WeakValueDictionary

```Python

class Cheese:
    def __init__(self, kind):
        self.kind = kind

    def __repr__(self):
        return 'Cheese(%r)' % self.kind


import weakref

stock = weakref.WeakKeyDictionary()
catalog = [Cheese('Red Leicester'), Cheese('Tilsit'), Cheese('Brie'), Cheese('Parmesan')]

for cheese in catalog:
    stock[cheese.kind] = cheese

print(sorted(stock.keys()))

del catalog
print(stock.keys())
del cheese
print(sorted(stock.keys()))

```

### 弱引用的限制

不是所有的`Python`对象都能使用弱引用。基本`list`和`dict`都不能被引用。但它们的纯净子类可以。`Set`也可以

