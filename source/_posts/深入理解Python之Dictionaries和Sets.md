---
title: 深入理解Python之Dictionaries和Sets
date: 2017-04-15 16:57:42
tags:
---


### Dictionaries

`Dictionary`是`Python`中最常用的数据结构之一

#### 推导式

```python
DIAL_CODES = [
    (86, 'China'),
    (91, 'India'),
    (1, 'United States'),
    (62, 'Indonesia'),
    (55, 'Brazil'),
    (92, 'Pakistan'),
    (880, 'Bangladesh'),
    (234, 'Nigeria'),
    (7, 'Russia'),
    (81, 'Japan'),
]

country_code = {country: code for code, country in DIAL_CODES}
# print(country_code)
print({code: country.upper() for country, code in country_code.items() if code < 66})
```

#### 使用`setdefault`处理无`key`

```python
import sys
import re

WORD_RE = re.compile('\w+')

index = {}
with open(sys.argv[1], encoding='utf-8') as fp:
    for line_no, line in enumerate(fp, 1):
        for match in WORD_RE.finditer(line):
            word = match.group()
            column_no = match.start() + 1
            location = (line_no, column_no)
            occurrences = index.get(word, [])
            occurrences.append(location)
            index[word] = occurrences
for word in sorted(index, key=str.upper):
    print(word, index[word])
```

* 可以使用`get(key, defaultValue)`和`setdefault(key, defaultValue)`来处理`key`不存在的情况,也可以使用`collections.defaultdict(value)`来设置创建默认`new-key`的字典。采用这方式，如果`dd`是一个`defaultdict`,那么当`dd[k]`不存在时，会调用`default_factory`来创建一个默认值，但返回结果依然是`None`

#### `__missing__`

这个方法默认是没有定义的，当`dict`的子类提供了这个方法的实现，那么当`key`不存在时，不会抛出异常`KeyError`，`__getitem__`会调用`__missing__`

```python
class StrKeyDict0(dict):

def __missing__(self, key):
    if isinstance(key, str):
        raise KeyError(key)
    return self[str(key)]

def get(self, k, d=None):
    try:
        return self[k]
    except KeyError:
        return default

def __contains__(self, item):
    return key in self.keys() or str(key) in self.keys()
```

* 这里需要注意`isintance`检查`key`是否为`str`,否则会陷入死循环。使用`k in dict.keys()`效率很高

#### 各种各样的`dict`
 
除了`defaultdict`之外，还会有很多别的`dict`
 
##### collections.OrderedDict
 
`key`有顺序的`dict`，类似`Java`中的`TreeMap`
 
##### collections.ChainMap

将多个`dict`当成一个`dict`，按顺序搜索`key`，只要搜索到`key`，结果返回成功

##### collections.Counter

持有`key`的计数

```python
ct = collections.Counter('dsakfjdjsfakdjfs')
print(ct)

result: Counter({'d': 3, 's': 3, 'f': 3, 'j': 3, 'a': 2, 'k': 2})
```

##### userDict

纯`Python`实现的标准`dict`

* 一般来说`userDict`被用来集成，其他几个直接使用

#### UserDict派生

`UserDict`内部提供了很多默认的实现方法，因此直接从`UserDict`继承会很方便。`UserDict`不是继承`dict`，但其内部有个`dict`实例。这是真正数据存储的地方

```Python
class StrKeyDict0(collections.UserDict):
def __missing__(self, key):
    if isinstance(key, str):
        raise KeyError(key)
    return self[str(key)]

def __contains__(self, item):
    return str(item) in self.data

def __setitem__(self, key, value):
    self.data[str(key)] = value
```

#### 不可改Mapping

标准库提供的映射类型都是可以修改的。你可能想让你的映射不能够被修改。`Python3.3`之后，标准库提供了只读的`MappingProxy`

```python
from types import MappingProxyType
d = {1: 'A'}
d_proxy = MappingProxyType(d)
print(d_proxy)
d_proxy[2] = 'x'
```

* 上面的代码会抛出异常，因为`d_proxy`是不允许被修改的

### Set

```Python
l = ['spam', 'spam', 'eggs', 'spam']
print(set(l))

result: {'spam', 'eggs'}
```

* 需要注意`Set`的元素必须是能哈希的
* 支持`&(intersetion)`

#### Set语法

`Python3`中，`set`标准表示方法`s = {1}`。使用这个方式会比使用`set([1])`效率高

```Python
from dis import dis
print(dis('{1}'))

result:

1           0 LOAD_CONST               0 (1)
            2 BUILD_SET                1
            4 RETURN_VALUE
            

print(dis('set([1])')

result:
1           0 LOAD_NAME                0 (set)
            2 LOAD_CONST               0 (1)
            4 BUILD_LIST               1
            6 CALL_FUNCTION            1
            8 RETURN_VALUE
```

* 使用`dis`来查看两种方式的字节码操作。第一种方式效率明显高于第二种

#### Set推导式

```python
from unicodedata import name
print({chr(i) for i in range(32, 256) if 'SIGN' in name(chr(i), '')})
```

### 其他

#### 性能


| 数据量 | 因子 | dict时间 | 因子 | 
| ------------- |-------------| -----| -------|
|1000| 1x | 0.000202s | 1.00x |
|10000 | 10x | 0.000140s | 0.69x| 
|100000 | 100x | 0.000228s | 1.13x |
|1000000 | 1000x | 0.000290s | 1.44x | 
|10000000 | 10000x | 0.000337s | 1.67x |

`dict`的高性能多亏了`hashtable`

![dict_hash_algorithm](./dict_hash_algorithm.png)

#### 好处和坏处

* `key`必须要能`hash`,需要满足
	* 支持`hash()`，同一个对象返回的值总是相同
	* 支持`eq()`
	* 如果`a==b`，那么`hash(a)==hash(b)`
	* 用户定义的类型的`hash`值是`id()`

* 内存开销大

* key搜索非常快
  * `dict`是典型的以空间换取时间

* key的顺序依赖插入顺序
* 添加数据会影响已存在`key`的顺序



