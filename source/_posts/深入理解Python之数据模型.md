---
title: 深入理解Python之数据模型
date: 2017-04-03 17:23:12
tags:
---

### 数据模型(Data Model)

``python``提供了很多特殊的方法

#### 不含操作符

| 种类  | 方法名称 |
| ------------- | ------------- |
| 字符串/字节表示  | ``__repr__``, ``__str__``, ``__format__``, ``__bytes``  |
| 数值转换  | ``__abs__``, ``__bool__``, ``__complex``, ``__int__``, ``__float__``, ``__hash__``, ``__index__``  |
| 模拟集合 | ``__len__``, ``__getitem__``, ``__setitem__``, ``__delitem``, ``__contains__``|
| 迭代器 | ``__iter__``, ``__reversed__``, ``__next__``|
| 模拟调用 | ``__call__`` |
| 上下文管理 | ``__enter__``, ``__exit__`` |
| 实例创建和销毁 | ``__new__``, ``__init__``, ``__del__``| 
| 属性管理 | ``__getattr__``, ``__getattribute__``, ``__setattr__``, ``__delattr__``, ``__dir__``|
| 属性描述器 | ``__get__``, ``__set__``, ``__delete__``|
| 类服务 | ``__prepare__``, ``__instancecheck__``, ``__subclasscheck__``|

#### 操作符

| 种类 | 方法名和相关操作符 |
| -------------- | ---------------|
| 一元数值操作符 | ``__neg__``-, ``__pos__``+, ``__abs__``() |
| 比较操作符 | ``__lt__``<, ``__le__``<=, ``__eq__``=, ``__ne__``!=, ``__gt__``>, ``__ge__``>= |
| 算术操作符 | ``__add__``+, ``__sub__``-, ``__mul__``*, ``__truediv__``/, ``__floordiv__``//, ``__mod__``%, ``__divmod__ divmod()``, ``__pow__ **或者(pow())``, ``__round__ round()`` |
| 反向算术操作符 | ``__radd__``, ``__rsub__``, ``__rmul__``, ``__rtruediv__``, ``__rfloordiv__``, ``__rmod__``, ``__rdivmod__``, ``__rpow__`` |
| 自操作符 | ``_iadd__``, ``__isub__``, ``__imul__``, ``__itruediv__``, ``__ifloordiv__``, ``__imod__``, ``__ipow__`` |
| 位操作符 | ``__invert__``~, ``__lshift__``<<, ``__rshift__``>>, ``__and__``&, ``__or__``|, ``__xor__``^ |
| 反向位操作符 | ``__rlshift__``, ``__rrshift__``, ``__rand__``, ``__rxor__``, ``__ror__``
| 自增位操作符 | ``__ilshift__``, ``__irshift__``, ``__iand__``, ``__ixor__``, ``__ior__``
 
* 通过实现上述特殊的方法，你自己的对象能表现得像内置类型。例如为了让对象打印出来可读性更好，这个时候需要实现``__repr__``和``__str__`` [区别](http://stackoverflow.com/questions/1436703/difference-between-str-and-repr-in-python)