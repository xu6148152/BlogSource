---
title: 深入理解Python之数组序列
date: 2017-04-04 17:26:00
tags:
---


### 列表推导式和生成器表达式
#### 列表推导式可读性
```python
def test1():
    codes = []
    for symbol in symbols:
        codes.append(ord(symbol))
    print(codes)


def test2():
    codes = [ord(symbol) for symbol in symbols]
    print(codes)
```

* 上面两个代码的效果是一样的，``test2``使用列表推导式的可读性会更好

```python
x = 'my precious'
dummy = [x for x in 'ABC']
print(x)
```

* 对于上述代码，在``python2.x``中，``x``的值是``C``。而在``python3.x``中，``x``的值依然是``my precious``。在``python3.x``中，列表推导式的变量不再泄露，其有自己的内部作用域

#### 列表推导式和``map``,``filter``
列表推导式也能做到``map``和``filter``能做到的

```python
beyond_ascii = [ord(s) for s in symbols if ord(s) > 127]
print(beyond_ascii)

beyond_ascii = list(filter(lambda c: c > 127, map(ord, symbols)))
print(beyond_ascii)
```

* 上述两个表达式的结果一样，但使用列表推导式会更简洁。而且它们的效率也差不多。我们来测试一下它们的效率

```python
TIMES = 10000

SETUP = """
symbols = '$¢£¥€¤'
def non_ascii(c):
    return c > 127
"""


def clock(label, cmd):
    res = timeit.repeat(cmd, setup=SETUP, number=TIMES)
    print(label, *('{:.3f}'.format(x) for x in res))


def test_speed():
    clock('listcomp        :', '[ord(s) for s in symbols if ord(s) > 127]')
    clock('listcomp + func :', '[ord(s) for s in symbols if non_ascii(ord(s))]')
    clock('filter + lambda :', 'list(filter(lambda c: c > 127, map(ord, symbols)))')
    clock('filter + func   :', 'list(filter(non_ascii, map(ord, symbols)))')
```

* 运行结果:

```python
listcomp        : 0.015 0.015 0.015
listcomp + func : 0.019 0.019 0.021
filter + lambda : 0.017 0.017 0.017
filter + func   : 0.016 0.019 0.016
```

#### 笛卡尔积
列表推导式能从两个或多个迭代对象中生成笛卡尔积

```python
colors = ['black', 'white']
sizes = ['S', 'M', 'L']
tshirts = [(color, size) for color in colors for size in sizes]
print(tshirts)
```

* 结果

```python
[('black', 'S'), ('black', 'M'), ('black', 'L'), ('white', 'S'), ('white', 'M'), ('white', 'L')]
```

#### 生成器表达式

```python
colors = ['black', 'white']
sizes = ['S', 'M', 'L']

for tshirt in ('%s %s' % (c, s) for c in colors for s in sizes):
    print(tshirt)
```

* 通过使用生成器表达式可以省去构建数组的成本

### ``Tuples``不仅是不可变的序列

#### ``Tuple``作为``Records``

```python
traveler_ids = [('USA', '31195855'), ('BRA', 'CE342567'), ('ESP', 'XDA205856')]
for passport in sorted(traveler_ids):
    print('%s/%s' % passport)

for country, _ in traveler_ids:
    print(country)
```

##### ``Tuple``解包

```python
lax_coordinates = (33.9425, -118.408056)
latitude, longitude = lax_coordinates
print('{0}, {1}'.format(latitude, longitude))
print('%s, %s' % (latitude, longitude))

a, b, *rest = range(5)
print(a, b, rest)
```

* ``python3.x``之前还支持函数参数解包。

##### 命名``Tuple``

```python
from collections import namedtuple
City = namedtuple('City', 'name country population coordinates')
tokyo = City('Tokyo', 'JP', 36.933, (35.689722, 139.691667))
print(tokyo)

LatLong = namedtuple('LatLong', 'lat long')
delhi_data = ('Delhi NCR', 'IN', 21.935, LatLong(28.613889, 77.208889))
delhi = City._make(delhi_data)
print(delhi._asdict())
```

* ``_field``是``tuple``类的属性名
* ``_make``允许使用迭代器初始化``named tuple``
* ``_asdict``返回一个用``name tuple``构建的排过序的字典

### 切片

``python``中所有的序列类型都支持切片操作

```python
t = (1, 2, [30, 40])
t[2] += [50, 60]
print(t)
```

* 执行上述代码会报错，但事实上``t``已经变成了``[30, 40, 50, 60]``
* 不要将可变对象放到``tuple``中
* ``+=``不是一个原子操作。从上面我们看到，其实异常时在它执行完之后抛出的
* 使用``dis``能够很方便的查看``python``字节码

### 使用二分法管理序列

#### 查找

```python
def grade(score, breakpoints=[60, 70, 80, 90], grades='FDCBA'):
    i = bisect.bisect(breakpoints, score)
    return grades[i]

print([grade(score) for score in [33, 99, 77, 70, 89, 90, 100]])
```

#### 排序

```python
SIZE = 7
random.seed(1729)

my_list = []
for i in range(SIZE):
    new_item = random.randrange(SIZE * 2)
    bisect.insort(my_list, new_item)
    print('%sd ->' % new_item, my_list)
```

### 数组

使用``array.tofile和array.fromfile``，效率很高。读取使用
``array.tofile``创建的二进制文件中的1千万个双精度浮点数仅需0.1s。这速度是从文件读取数字的60倍。写操作是7倍。并且最后生成的文件小

> 注意，``Python3.4``之后``array``已经没有内置的排序方法

### MemoryView

```python
numbers = array('h', [-2, -1, 0, 1, 2])
memv = memoryview(numbers)
print(len(memv))
print(memv[0])
memv_oct = memv.cast('B')
print(memv_oct.tolist())
memv_oct[5] = 4
print(numbers)
```

### 双端队列和其他队列

```python
dq = deque(range(10), maxlen=10)
print(dq)
dq.rotate(3)
print(dq)
dq.rotate(-4)
print(dq)
dq.appendleft(-1)
print(dq)
```

* 除了``deque``之外，其他一些标准库也实现了``queues``: 
  * ``queue``: ``Queue``, ``LifoQueue``, ``PriorityQueue``
  * ``multiprocessing``: ``JoinableQueue``
  * ``asyncio``
  * ``heapq``

