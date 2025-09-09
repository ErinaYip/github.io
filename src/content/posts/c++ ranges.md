---
title: Ranges
published: 2025-09-09
tags: ["C++", "算法"]
description: "C++20 在 std::ranges 命名空间中提供了大多数算法的受限版本，也就是受限算法，并且引入了范围库（Ranges Library），其可组合且不易出错，功能更加强大。"
category: 'C++'
draft: false
---
# Ranges

> C++20 在`std::ranges`命名空间中提供了大多数算法的受限版本，也就是受限算法，并且引入了范围库`（Ranges Library）`，其可组合且不易出错，功能十分强大  
> ~~让高贵的C++20玩家迭代器跟python的一样好用~~

## 视图 views

views 可以以用管道符 | 组合多个视图和算法，并且它们拥有惰性计算的特性，即不会立即计算结果，而是按需生成。

通常，我们直接在`range-based loops`中使用views过滤和更改特定的值，使代码更加简洁


* **示例(filter)：**
```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5, 6};

    //定义lambda函数even
    auto even = [](int n) { return n % 2 == 0; };

    // 过滤偶数
    for (auto& x : nums | std::views::filter(even)) {
        std::cout << x << " ";
    }
    // 输出：2 4 6
}
```

* **示例(transform)：**
```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5, 6};

    auto even = [](int n) { return n % 2 == 0; };
    auto pow = [](int n) { return n * n; };

    for (auto x : nums | std::views::filter(even) | std::views::transform(pow)) {
        std::cout << x << " ";
    }
    // 输出：4 16 36
}
```

* **示例(enumerate、reverse)：**
```cpp
#include <iostream>
#include <ranges>

int main() {
    auto const numbers = {3, 1, 4, 1, 5, 9, 2, 6};

    for (auto [index, value]: numbers | std::views::enumerate | std::views::reverse) {
        std::cout << index << ": " << value << '\n';
    }

    return 0;
}

```
* **输出：**
```
7: 6
6: 2
5: 9
4: 5
3: 1
2: 4
1: 1
0: 3
```


## 受限函数
C++20 在 `<algorithm>` 头文件中为大多数算法都增加了定义于`std::ranges::`的版本，这些函数可以直接传入容器、view 或自定义范围对象，而不需要手动指定迭代器，比如：
- `std::ranges::sort`
- `std::ranges::find`
- `std::ranges::min_element`
- `std::ranges::max_element`
- `std::ranges::reverse`
- `std::ranges::copy`
- `std::ranges::copy_if`
- `std::ranges::for_each`

* **示例(sort)**
```cpp
#include <vector>
#include <ranges>
#include <algorithm>
#include <iostream>

int main() {
    std::vector<int> v = {5, 2, 8, 1};
    std::ranges::sort(v); // 直接对容器排序

    for (const auto& i : v) { std::cout << i << ' '; }
    //输出 1 2 5 8
}
```

* **示例(min_element/min)**
```cpp
#include <vector>
#include <ranges>
#include <algorithm>
#include <iostream>

int main() {
    std::vector<int> v = {5, 2, 8, 1};

    auto it = std::ranges::min_element(v); //返回一个迭代器
    if (it != v.end()) { std::cout << *it << ' '; }

    auto min = std::ranges::min(v); //直接返回值
    std::cout << min;
}
```

* **示例(next_permutation)**

~~来猴戏一下全排列~~

```cpp
#include <algorithm>
#include <iostream>
#include <string>

int main() {
	std::string a {"abc"};
	do {
		for (const auto& i : a) { std::cout << i << " "; }
		std::cout << "\n";
	} while (std::ranges::next_permutation(a).found);
}
```

~~猴戏还得是python~~

```python title="python"
from itertools import permutations

a: str = "abc"
for p in permutations(a):
    print(' '.join(p))
```

* **输出：**
```
a b c
a c b
b a c
b c a
c a b
c b a
```

:::note
为了防止悬空迭代器，view（比如 filter 之后的结果）只返回`std::ranges::dangling`，正如其名，这个类型不是实际的容器，不能被受限函数们直接使用
:::

如果希望保存views的结果，我们需要手动收集，**例如**：

```cpp {11, 12, 14-18}
#include <iostream>
#include <vector>
#include <ranges>
#include <algorithm>

int main() {
    auto const numbers = {3, 1, 4, 1, 5, 9, 2, 6};

    auto even = [](int n) { return n % 2 == 0; };

    // 使用 std::ranges::to 收集 （C++23)
    std::vector<int> evens = std::ranges::to<std::vector<int>>(numbers | std::views::filter(even));

    // 直接遍历收集
    std::vector<int> evens;
    for (int n : numbers | std::views::filter(even)) {
        evens.push_back(n);
    }

    auto min = std::ranges::min_element(evens); //✔

    if (min != evens.end()) { std::cout << *min << std::endl; }

    return 0;
}

```

---

参考文献：

[Ranges library (since C++20) - cppreference.com](https://en.cppreference.com/w/cpp/ranges.html)  
[范围库（C++20 起）- cppreference.cn - C++参考手册](https://cppreference.cn/w/cpp/ranges)  
[受限算法（C++20 起）- cppreference.cn - C++参考手册](https://cppreference.cn/w/cpp/algorithm/ranges)  