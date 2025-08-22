---
title: datetime库
published: 2023-07-13 21:08:20 
updated: 2023-12-08 12:58:32
description: ''
image: ''
tags: ['Python']
category: 'Python'
draft: false 

---
# Python datetime库
- datetime是Python内置的一个处理日期和时间的标准库，可以轻松处理日期和时间，也可以进行日期和时间的格式化操作。下面是一些datetime库中常用的方法：

### datetime库中常用的方法：
|方法|描述|
|---|---|
datetime.date|返回表示日期的对象。|
datetime.time|返回表示时间的对象。|
datetime.datetime|返回日期和时间的对象。|
datetime.timedelta|表示两个日期或时间之间的差异（例如，两个日期之间的天数）。|
datetime.strptime|把格式化的字符串转换为日期对象。|
datetime.strftime|把日期对象格式化为字符串。|
datetime.timetuple|返回一个 time.struct_time对象，具有包含九个元素的命名元组接口。|


### time.struct_time 对象中存在以下值：


|索引|属性|值|
|-|-|-|
0|tm_year|(例如，1993)
1|tm_mon|范围 [1, 12)
2|tm_mday|范围 [1, 31)
3|tm_hour|范围 [0, 23)
4|tm_min|范围 [0, 59)
5|tm_sec|范围 [0, 61)
6|tm_wday|范围 [0, 6)，星期一为 0
7|tm_yday|范围 [1, 366)
8|tm_isdst|0、1 或 -1

### 以下代码示例展示了如何使用datetime库：
```python
import datetime
# 获取当前时间并打印
now = datetime.datetime.now()
print("当前时间为：", now)
# 创建一个表示指定日期和时间的datetime对象
d = datetime.datetime(2021, 10, 12, 15, 0)
print("指定的日期和时间为：", d)
# 获取两个日期之间的差异
delta = datetime.timedelta(days=7)
next_week = now + delta
print("一周后的时间为：", next_week)
# 把字符串转换为日期对象
date_string = "2022-01-01"
date_object = datetime.datetime.strptime(date_string, "%Y-%m-%d")
print("转换后的日期为：", date_object)
# 把日期对象转换为字符串
date_str = date_object.strftime("%d/%m/%Y")
print("转换后的字符串为：", date_str)
```
> 输出为
```
当前时间为： 2023-05-24 19:51:43.019975
指定的日期和时间为： 2021-10-12 15:00:00
一周后的时间为： 2023-05-31 19:51:43.019975
转换后的日期为： 2022-01-01 00:00:00
转换后的字符串为： 01/01/2022
```
### 例题讲解

**题目描述**
```输入某一年的日期，输出该天是本年的第多少天。```

**输入**
```输入一行，表示某年的日期。```

**输出**
```输出一个正整数，表示这一天是该年的第几天。```

**样例输入**
```2023-05-22```

**样例输出**
```142```

### 题解
```python
from datetime import datetime
a = input().strip()
d = datetime.strptime(a,'%Y-%m-%d')
ans = d.timetuple().tm_yday
print(ans)
```
- 当然，运用__import__函数我们可以很轻易地压行
```python
print(__import__('datetime').datetime.strptime(input(),'%Y-%m-%d').timetuple().tm_yday)
```
