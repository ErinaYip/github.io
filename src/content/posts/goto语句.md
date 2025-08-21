---
title: goto语句
published: 2023-07-13 20:39:46 
image: ''
tags: ['C++']
category: 'C++'
draft: false 
---
# 跳转语句

`C语言`的跳转语句主要包括`continue`,`break`,`retuen`,还有就是`goto`啦

# goto语句
> `goto`语句是在所有跳转语句中最自由的一种,
>
> 但在大型工程和多人协作工程中并不推荐,原因就在于它`太过于自由`,会导致代码的可读性变得`较差`
>
> 但这也无法撼动`goto`语句的地位
>
> 合理的使用goto会大大简化代码，并且使程序逻辑更加清晰

## 什么是goto语句
> `goto`,又称`无条件跳转语句`,使用goto语句可以直接跳转到label标注处,其语法为``goto lable;``

## 示例
```cpp
#include<iostream>
using namespace std;
int main(){
    for (int i = 1; i <= 10; ++ i){
        printf("%d ", i);
        if (i == 6){
            goto ERA;
        }
    }
    cout << "Before ending" << endl;
    ERA:
    cout << "end" << endl;
}
```

输出
```bash
1 2 3 4 5 6 end
```
