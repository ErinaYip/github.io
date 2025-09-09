---
title: template模板
published: 2023-07-13
updated: 2025-09-04
tags: ["C++", "算法"]
category: 'C++'
draft: false
---
# C++ 模板

> 模板函数/类可以处理任意类型（泛型），编译时自动生成对应类型的代码。

## 模板函数
```cpp
template <typename type>
return-type func-name(parameter list) {
   // 函数的主体
}
```
在这里，`type` 是函数所使用的数据类型的占位符名称。

### 实例
> 下面是函数模板的实例，返回两个数中的最大值：
```cpp
#include <iostream>
#include <string>

using std::cout;

template <typename T>
inline T const& Max (T const& a, T const& b) { 
    return a < b ? b:a; 
} 
int main () {
    int i = 39;
    int j = 20;
    cout << "Max(i, j): " << Max(i, j) << '\n'; 

    double f1 = 13.5; 
    double f2 = 20.7; 
    cout << "Max(f1, f2): " << Max(f1, f2) << '\n'; 

    std::string s1 = "Hello"; 
    std::string s2 = "World"; 
    cout << "Max(s1, s2): " << Max(s1, s2) << '\n'; 

    return 0;
}
```

* **输出**

``` 
Max(i, j): 39
Max(f1, f2): 20.7
Max(s1, s2): World
```

### 实例2
> 使用`template模板`实现的快读
```cpp
#include<iostream>

using std::cout;

template <typename T>
inline void read(T& a)
{
	int s = 0, w = 1;
	char ch = getchar();
	while (ch < '0' || ch>'9') {
		if (ch == '-')
			w = -1;
		ch = getchar();
	}
	while (ch >= '0' && ch <= '9') {
		s = s * 10 + ch - '0';
		ch = getchar();
	}
	a = s * w;
}

int main() {
    int n;
    read(n);
    cout << n << '\n';
}
```

## 模板类

### 实例
```cpp
template<typename T>
class Box {
public:
    T value;
};
```






---

参考文献：

[模板 - cppreference.cn - C++参考手册](https://cppreference.cn/w/cpp/language/templates)  
[Templates - cppreference.com](https://en.cppreference.com/w/cpp/language/templates.html)  
[C++ Templates Introduction | hacking C++](https://hackingcpp.com/cpp/lang/templates.html)  