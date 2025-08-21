---
title: Collections库
published: 2023-10-22 14:55:02 
updated: 2023-12-08 12:58:32
description: ''
image: ''
tags: ['Python']
category: 'Python'
draft: false 
---
# Python Collections库详解

Python的 `collections` 库提供了一些额外的数据结构，用于扩展内置的列表、元组和字典等数据类型。这些数据结构针对特定的使用场景进行了优化，可以极大地简化编程任务。在本篇文章中，我们将探索一些常用的 `collections` 库中的数据类型。

## defaultdict
 `defaultdict` 是 `dict` 的一个子类，它为缺失的键提供了默认值。当访问一个不存在的键时，不会引发 `KeyError` 错误，而是返回一个默认值。这在计数、分组元素或构建复杂数据结构时特别有用。

使用示例：
```python
>>> from collections import defaultdict
>>> # 创建一个默认值为0的defaultdict
>>> counter = defaultdict(int)
>>> counter['apple'] += 1
>>> counter['banana'] += 1
>>> counter
defaultdict(<class 'int'>, {'apple': 1, 'banana': 1})
```
## Counter
`Counter` 是一个方便的工具，用于计数可哈希对象。它提供了简单的计数功能，可以用于统计元素出现的次数、查找列表中的重复元素等。
```python
>>> from collections import Counter
>>> # 创建一个Counter对象
>>> numbers = [1, 2, 3, 2, 1, 3, 4, 5, 1]
>>> counter = Counter(numbers)
>>> counter
Counter({1: 3, 2: 2, 3: 2, 4: 1, 5: 1})
```

## deque
`deque` 是一个双端队列，支持高效地在两端进行添加和删除操作。它比列表更适合用于实现队列、栈和循环缓冲区等数据结构。
```python
>>> from collections import deque
>>> # 创建一个deque对象
>>> queue = deque()
>>> queue.append(1)
>>> queue.append(2)
>>> queue.append(3)
>>> queue
deque([1, 2, 3])
```

## namedtuple
`namedtuple` 是一个用于创建具有命名字段的元组子类的工具。它可以帮助提高代码的可读性，使得元组更具有结构性。
```python
>>> from collections import namedtuple
>>> # 创建一个namedtuple类
>>> Person = namedtuple('Person', ['name', 'age', 'gender'])
>>> person = Person('Alice', 25, 'female')
>>> Person
<class '__main__.Person'>
>>> person
Person(name='Alice', age=25, gender='female')
```
