---
title: 深入理解Pyhton之函数设计模式
date: 2017-04-29 18:50:28
tags:
---


### 前言

虽然设计模式是语言无关。但并不意味着每种设计模式都适用于任何语言。`Python`中可以多用一些设计模式如: 策略,命令,模板方法和访问者模式。

### 案例学习: Pyhton中的策略模式

`Python`使用策略模式能让你的代码更简单。

####  传统策略模式

![](./straetgy_uml.png)


```Python

Customer = namedtuple("Customer", 'name fidelity')


class LineItem:
    def __init__(self, product, quantity, price):
        self.product = product
        self.quantity = quantity
        self.price = price

    def total(self):
        return self.price * self.quantity


class Order:  # the context

    def __init__(self, customer, cart, promotion=None):
        self.customer = customer
        self.cart = list(cart)
        self.promotion = promotion

    def total(self):
        if not hasattr(self, '__total'):
            self.__total = sum(item.total() for item in self.cart)
        return self.__total

    def due(self):
        if self.promotion is None:
            discount = 0
        else:
            discount = self.promotion.discount(self)
        return self.total() - discount

    def __repr__(self):
        fmt = '<Order total: {:.2f} due: {:.2f}>'
        return fmt.format(self.total(), self.due())


class Promotion(ABC):  # the Strategy: an abstract base class

    @abstractclassmethod
    def discount(self, order):
        """Return discount as a positive dollar amount"""


class FidelityPromo(Promotion):  # first Concrete strategy
    """%5 discount for customers with 1000 or more fidelity points"""

    def discount(self, order):
        return order.total() * .05 if order.customer.fidelity >= 1000 else 0


class BulkItemPromo(Promotion):  # second Concrete strategy
    """10% discount for each LineItem wit 20 or more units"""

    def discount(self, order):
        discount = 0
        for item in order.cart:
            if item.quantity >= 20:
                discount += item.total * .1
        return discount


class LaregOrderPromo(Promotion):  # third concrete startegy
    """%7 discount for orders with 10 or more distinct items"""

    def discount(self, order):
        distinct_items = {item.product for item in order.cart}
        if len(distinct_items) >= 10:
            return order.total() * .07
        return 0


if __name__ == '__main__':
    joe = Customer('John Doe', 0)
    ann = Customer('Ann Smith', 1100)
    cart = [LineItem('banana', 4, .5),
            LineItem('apple', 10, 1.5),
            LineItem('watermellon', 5, 5.0)]

    print(Order(joe, cart, FidelityPromo()))
```

* 令`Promotion`继承`ABC`作为抽象基类，以便使用`@abstractclassmethod`装饰器

#### 面向函数的策略模式

对于上述的例子，每个实体策略有一个单独的方法`discount`。进一步讲，策略对象没有状态。它们看起来就像纯净的函数。因此我们重构上述代码

对于`Order`类进行如下改造
```Python
def due(self):
    if self.promotion is None:
        discount = 0
    else:
        # discount = self.promotion.discount(self)
        #2
        discount = self.promotion(self)
    return self.total() - discount

```

添加如下方法

```Python

def fideilty_promo(order):
    return order.total() * .05 if order.customer.fidelity >= 1000 else 0


def bulk_item_promo(order):
    discount = 0
    for item in order.cart:
        if item.quantity >= 20:
            discount += item.total * .1
    return discount


def large_order_promo(order):
    distinct_items = {item.product for item in order.cart}
    if len(distinct_items) >= 10:
        return order.total() * .07
    return 0

```

* 将原来的`Promotion`以及其子类由上述方法替代，不需要使用抽象函数

#### 选择最佳策略: 简单方案


```Python
promos = [fideilty_promo, bulk_item_promo, large_order_promo]


def best_promo(order):
    return max(promo(order) for promo in promos)
```

* 使用`best_promo`方法来查询最佳方案

#### 系统模块中寻找策略

内置`globals`
> 返回表示当前全局符号表的字典

```python
promos = [globals()[name] for name in globals() if name.endswith('_promo') and name != 'best_promo']

promos = [func for name, func in inspect.getmembers(promos, inspect.isfunction)]
```

### 命令模式

```python
class MacroCommand:
    def __init__(self, commands):
        self.commands = list(commands)

    def __call__(self):
        for command in self.commands:
            command()
``
