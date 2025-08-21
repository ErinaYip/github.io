---
title: template模板
published: 2023-07-13
tags: ["C++", "算法"]
category: 'C++'
draft: false
---
# C++ 模板
> 模板是`泛型编程`的基础，`泛型编程`即以一种独立于`任何特定类型`的方式编写代码。

## 函数模板
> **模板函数定义的一般形式如下所示：**
```cpp
template <typename type> ret-type func-name(parameter list)
{
   // 函数的主体
}
```
在这里，`type` 是函数所使用的数据类型的占位符名称。

### 实例
> 下面是函数模板的实例，返回两个数中的最大值：
```cpp
#include <iostream>
#include <string>
using namespace std;
template <typename T>
inline T const& Max (T const& a, T const& b) { 
    return a < b ? b:a; 
} 
int main (){
    int i = 39;
    int j = 20;
    cout << "Max(i, j): " << Max(i, j) << endl; 

    double f1 = 13.5; 
    double f2 = 20.7; 
    cout << "Max(f1, f2): " << Max(f1, f2) << endl; 

    string s1 = "Hello"; 
    string s2 = "World"; 
    cout << "Max(s1, s2): " << Max(s1, s2) << endl; 

    return 0;
}
```
### 输出
```bash
Max(i, j): 39
Max(f1, f2): 20.7
Max(s1, s2): World
```

## 实例2
> 使用`template模板`实现的快读
```cpp
#include<iostream>
using namespace std;

template <typename T>
inline void read(T &a)
{
	int s = 0, w = 1;
	char ch = getchar();
	while (ch < '0' || ch>'9'){
		if (ch == '-')
			w = -1;
		ch = getchar();
	}
	while (ch >= '0' && ch <= '9'){
		s = s * 10 + ch - '0';
		ch = getchar();
	}
	a = s * w;
}

int main(){
    int n;
    read(n);
    cout << n;
}
```
