---
title: Fromat
published: 2025-11-11
tags: ["C++"]
description: "C++ format库是一个现代、安全且快速的字符串格式化库，它提供了类似Python的格式化语法，并作为std::format被纳入C++20标准。"
category: 'C++'
draft: true
---

`C++ 20`基于`Python`引入了一系列基本类型和字符串类型的格式规范[^1]，
同时引入了`<format>`库，替代了传统的 sprintf和字符串流方法，
使得C++具有了快速且灵活的字符串格式化语法。

`format库`采用`{}`作为占位符，类似于Python的字符串格式化。

`format`库的前身就是著名的`{fmt}`库，它是一个开源的C++格式化库。你可以把它理解为一个C++版本的`printf`的现代化、类型安全的替代品，即使现在被吸纳进`C++20`标准，也仍然保留着与之前几乎相同的接口。

:::note
C++ >= 23 时引入`<ostream>`或`<print>`库就会引入`<format>`库
:::

# std::format

## 参数

基本格式为`std::wstring format<_Args...>(std::wformat_string<_Args...> __fmt, _Args &&...__args)`，
其中，`fmt`表示格式化字符串的对象，它必须是一个编译时常量，剩下若干个参数`arg`表示要格式化的参数

## 字符串

```cpp
std::format("{} {}!", "Hello", "world"); // Hello world!
std::format("{{ asd }}"); // { asd }

// 显示设置arg-id来指定参数顺序
std::format("{1} {0}!", "Hello", "world"); // world Hello!

// 同时arg-id可以多次复用
std::format("{1} {0} {1}!", "Hello", "world"); // world Hello world!

// 多余了参数会被忽略而不引起报错，但是少了会
std::format("{} {}!", "Hello", "world", "something"); // Hello world!
```

## 对齐和填充

同时，就像python中一样，format能灵活地格式化不同类型的参数，并灵活的调整格式：

```cpp
// 固定占位6字符
// 默认右对齐数字类型（int, double 等）和格式化为数字的布尔类型
// 默认左对齐字符类型（char, char8_t, char16_t, char32_t, wchar_t 和字符串类型（string, string_view）

std::format("{:6}", 42);       // "    42"
std::format("{:6}", 'x');      // "x     "
std::format("{:6}", true);     // "true  "
std::format("{:6d}", true);    // "     1"

// 设定占位字符，主动规定对齐方式
std::format("{:*<6}", 'x');    // "x*****"
std::format("{:*>6}", 'x');    // "*****x"
std::format("{:*^6}", 'x');    // "**x***"

// 截断
std::format("{:.3}", "hello");  // "hel"
```

## 数字格式化

```cpp
std::format("{:d}", 42);     // 十进制
std::format("{:x}", 255);    // 十六进制小写
std::format("{:X}", 255);    // 十六进制大写
std::format("{:o}", 64);     // 八进制
std::format("{:b}", 5);      // 二进制
```

## 浮点型
```cpp
std::format("{:.2f}", 3.14159);   // 固定点，2位小数
std::format("{:.2e}", 1234.5);    // 科学计数法
std::format("{:.2g}", 1234.5);    // 自动选择
```

## 符号控制

对于数字类型，format还能控制数字符号的显示方式，包括科学计数法、浮点型等：

| 选项 | 说明 | 示例（值为42） | 示例（值为-42） |
|-|-|-|
| + | 总是显示符号 | +42 | -42 |
| - | 仅负数显示符号（默认） | 42 | -42 |
| 空格 | 负数显示-，正数显示空格 | 42 | -42 |

```cpp
std::format("{0:},{0:+},{0:-},{0: }", 1) // "1,+1,1, 1"
std::format("{0:},{0:+},{0:-},{0: }", -1) // "-1,-1,-1,-1"
std::format("{0:e},{0:+e},{0:-e},{0: e}\n", 1234567.89); // "1.234568e+06,+1.234568e+06,1.234568e+06, 1.234568e+06"
std::format("{0:e},{0:+e},{0:-e},{0: e}\n", 0.000123); // "1.230000e-04,+1.230000e-04,1.230000e-04, 1.230000e-04"
```

同时，对于无法用数字表示的数字类型，`format`也能灵活的格式化：

```cpp
double inf = std::numeric_limits<double>::infinity();
double nan = std::numeric_limits<double>::quiet_NaN();
std::format("{0:},{0:+},{0:-},{0: }", inf)    // "inf,+inf,inf, inf"
std::format("{0:},{0:+},{0:-},{0: }", nan)    // "nan,+nan,nan, nan"
```

## 日期和时间
```cpp
std::format("{:%Y-%m-%d}", std::chrono::system_clock::now());
```
# std::vformat

前面提到，`format`只能支持格式化到编译时常量字符串，
而`vformat`则是`format`的可变参数版本，它接受`std::format_args`参数包，能够将内容格式化到运行时量

## 运行时构建参数包
```cpp
#include <iostream>
#include <string>
#include <string_view>

template<typename... Args>
std::string dyna_print(std::string_view rt_fmt_str, Args&&... args) {
    return std::vformat(rt_fmt_str, std::make_format_args(args...));
}

int main() {
    std::cout << std::format("Hello {}!\n", "world");

    std::string fmt;
    for (int i = 0; i <= 3; ++i) {
        fmt += "{} ";
        std::cout << dyna_print(fmt, "alpha", 'Z', 3.14, "unused") << '\n';
    }
}
```
```bash title=输出
alpha 
alpha Z 
alpha Z 3.14
```

---

[^1]:[Python 中的格式规范](https://docs.pythonlang.cn/3/library/string.html#formatspec)

[标准库头文件 <format> (C++20) - cppreference.cn - C++参考手册](https://cppreference.cn/w/cpp/header/format)
[标准格式规范 (C++20 起) - cppreference.cn - C++参考手册](https://cppreference.cn/w/cpp/utility/format/spec)
[std::format - cppreference.cn - C++参考手册](https://cppreference.cn/w/cpp/utility/format/format)