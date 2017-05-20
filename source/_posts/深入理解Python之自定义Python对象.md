---
title: 深入理解Python之自定义Python对象
tags: Python
date: 2017-05-20 16:25:39
---


### 对象展示

`Python`中有两种方式将对象以字符串的形式表示

`repr()`
> 返回开发者想要到的字符串形式

`str()`
> 返回用户想要的字符串形式

通过实现特殊的方法`__repr__`和`__str__`来支持`repr()`和`str()`

有两个额外的方法来支持对象的展示形式`__bytes__`和`__format__`.`__byte__`方法和`__str__`类似，它被`bytes()`调用来显示对象的字节序列。`__format__`用于格式化对象显示

```python
class Vector2d:
    typecode = 'd'

    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __iter__(self):
        return (i for i in (self.x, self.y))

    def __repr__(self):
        class_name = type(self).__name__
        return '{}({!r}, {!r})'.format(class_name, *self)

    def __str__(self):
        return str(tuple(self))

    def __bytes__(self):
        return bytes([ord(self.typecode)]) + bytes(array(self.typecode, self))

    def __eq__(self, other):
        return tuple(self) == tuple(other)

    def __abs__(self):
        return math.hypot(self.x, self.y)

    def __bool__(self):
        return bool(abs(self))

```

### 替代构造函数

因为能够以字节的形式导出`Vector2d`，因此需要一个方法能够从二进制序列中导出一个对象。标准库`array`中有这么一个方法`frombytes`

```python
@classmethod
def frombytes(cls, octets):
    typecode = chr(octets[0])
    memv = memoryview(octets[1:]).cast(typecode)
    return cls(*memv)
```

#### classmethod vs staticmethod

`classmethod`对类而不是实例进行操作，其改变了方法调用的方式，它接受类自身作为第一个参数，最常用于替代构造函数

   
`staticmethod`改变方法以便它收到的第一个参数不是特殊参数。静态方法就像一个纯净的函数存活在类中，而不是定义在模块层次

```python
class Demo:
    @classmethod
    def klassmeth(*args):
        return args

    @staticmethod
    def statmeth(*args):
        return args


print(Demo.klassmeth())

print(Demo.klassmeth('spam'))

print(Demo.statmeth())

print(Demo.statmeth('spam'))


result:

(<class '__main__.Demo'>,)
(<class '__main__.Demo'>, 'spam')
()
('spam',)
```

### 格式化显示

内置方法`format()`实际上调用`__format__(format_spec)`。

```python
brl = 1/2.43
format(brl, '0.4f')
'1 BRL = {rate:0.2f} USD'.format(rate=brl) => '1 BRL = 0.41 USD'

format(42, 'b') => '101010'
format(2/3, '.1%') => '66.7%'

now = datetime.now()
format(now, '%H:%M:%S') => '18:49:05'
"It's now {:%I:%M %p}".format(now) => "It's now 06:49 PM"
```

* 如果一个对象没有重写`__format__`那么将会调用`str()`，如果传入了格式化规格，那么会报错

自定义格式化

```python
    def __format__(self, format_spec=''):
        if format_spec.endswith('p'):
            format_spec = format_spec[:-1]
            coords = (abs(self), self.angle())
            outer_fmt = '<{}, {}>'

        else:
            coords = self
            outer_fmt = '({}, {})'
        components = (format(c, format_spec) for c in coords)
        return outer_fmt.format(*components)
```

### Hashable Vector2

```python
@property
    def x(self):
        return self.__x

    @property
    def y(self):
        return self.__y

    def __iter__(self):
        return (i for i in (self.__x, self.__y))

    def __hash__(self):
        return hash(self.__x) ^ hash(self.__y)

```

### Python中私有和保护的属性

`Python`无法像`Java`那样创建`private`属性。但可以通过`__`前缀来表示属性是私有的,`_`用于表示受保护的

### 使用__slot__节省空间

默认情况，`Python`存储对象属性在`__dict__`。当你处理大数据量的时候，`__slots__`能够节省很多内存

```python
__slots__ = ('__x', '__y')
```

通过定义`__slots__`来告诉解释器，这是这个所有的属性，`Python`会将它们存在每个对象的一个类似`tuple`的结构中

#### __slots__的弊端

* 必须为每个子类重新定义`__slots__`，因为继承属性会被忽略
* 对象只能拥有`__slots__`中的属性，除非将`__dict__`包括在`__slots__`中
* 对象不能够作为弱引用的目标，除非将`__weakref__`放到`__slots__`

