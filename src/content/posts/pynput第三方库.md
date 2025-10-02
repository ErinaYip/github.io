---
title: pynput第三方库
published: 2025-10-02
image: ''
tags: ['Python']
category: 'Python'
draft: false 
---

# pynput第三方库

> 此库允许您控制和监视输入设备。 ──官方文档
[pynput pypi下载页](https://pypi.org/project/pynput/)
[read the docs官方文档](https://pynput.readthedocs.io/en/latest/index.html)

:::warning
这篇博客最早写于14年，最近找回来清库存，有些内容可能已经过时
:::

~~话说为什么python包都喜欢用Read The Docs做文档~~

想要使用python控制/监听键盘/鼠标行为，可以尝试`pynput`

以下我来介绍`pynput`中常用的键盘类，即`pynput.keyboard`。

> 或者尝试`pyautogui`，除了对于键鼠的控制，它提供了更多包括消息窗口、截图等功能。

## Controller 模拟键盘按键

### 导入

使用前，导入`Controller`，并且实例化，同时，导入按键映射表`Key`

```python
from pynput.keyboard import Key, Controller 

keyboard = Controller()
```

### 模拟按键

#### 单个按键

使用`keyboard.press`来模拟按下按键，然后通过`keyboard.release`模拟松开，两个函数都接受传入字符串，所以可以直接打出大写字母。

```python
keyboard.press(Key.space) keyboard.release(Key.space)

keyboard.press('a')
keyboard.release('a')

keyboard.press('A')
keyboard.release('A')
```

或者使用按下`Shift`键来打出大写字母。

```python
# 直接按下shift不放
keyboard.press(Key.shift)
keyboard.press('a')
keyboard.release('a')
keyboard.release(Key.shift)

# 或者使用上下文管理语句with，更推荐
with keyboard.pressed(Key.shift):
    keyboard.press('a')
    keyboard.release('a')
```

#### 一串字符

使用`type()`方法来方便地打印一串字符，其内部实现其实也是通过`press()`和`release()`来遍历这个字符串。

```python
# 快速输入一段文本
keyboard.type('Hello, World!')

# 输入包含特殊字符的文本
keyboard.type('Special chars: !@#$%^&*()')
```

markdown




---
title: pynput第三方库
published: 2025-10-02
image: ''
tags: ['Python']
category: 'Python'
draft: false 
---

# pynput第三方库

> 此库允许您控制和监视输入设备。 ──官方文档
[pynput pypi下载页](https://pypi.org/project/pynput/)
[read the docs官方文档](https://pynput.readthedocs.io/en/latest/index.html)

~~话说为什么python包都喜欢用Read The Docs做文档~~

想要使用python控制/监听键盘/鼠标行为，可以尝试`pynput`。这是一个功能强大且易于使用的库，能够帮助你实现各种输入设备的控制和监听功能。

以下我来介绍`pynput`中常用的键盘类，即`pynput.keyboard`。

> 或者尝试`pyautogui`，除了对于键鼠的控制，它提供了更多包括消息窗口、截图等功能。

## Controller 模拟键盘按键

### 导入

使用前，导入`Controller`，并且实例化，同时，导入按键映射表`Key`

```python
from pynput.keyboard import Key, Controller 

keyboard = Controller()
```

### 模拟按键

#### 单个按键

使用`keyboard.press`来模拟按下按键，然后通过`keyboard.release`模拟松开，两个函数都接受传入字符串，所以可以直接打出大写字母。

```python
# 模拟空格键
keyboard.press(Key.space) 
keyboard.release(Key.space)

# 模拟普通字母
keyboard.press('a')
keyboard.release('a')

# 模拟大写字母
keyboard.press('A')
keyboard.release('A')
```

或者使用按下`Shift`键来打出大写字母。

```python
# 直接按下shift不放
keyboard.press(Key.shift)
keyboard.press('a')
keyboard.release('a')
keyboard.release(Key.shift)

# 或者使用上下文管理语句with，更推荐
with keyboard.pressed(Key.shift):
    keyboard.press('a')
    keyboard.release('a')
```

#### 一串字符

使用`type()`方法来方便地打印一串字符，其内部实现其实也是通过`press()`和`release()`来遍历这个字符串。

```python
# 快速输入一段文本
keyboard.type('Hello, World!')

# 输入包含特殊字符的文本
keyboard.type('Special chars: !@#$%^&*()')
```

#### 组合键

使用`pressed()`方法可以方便地实现组合键的操作：

```python
# Ctrl+C 复制
with keyboard.pressed(Key.ctrl):
    keyboard.press('c')
    keyboard.release('c')

# Alt+Tab 切换窗口
with keyboard.pressed(Key.alt):
    keyboard.press(Key.tab)
    keyboard.release(Key.tab)

# Ctrl+Alt+Delete 任务管理器
with keyboard.pressed(Key.ctrl):
    with keyboard.pressed(Key.alt):
        keyboard.press(Key.delete)
        keyboard.release(Key.delete)
```

## Listener 监听键盘事件

### 基本使用

`pynput`不仅可以模拟按键，还可以监听键盘事件。使用`Listener`类可以实现对键盘输入的监听。

```python
from pynput.keyboard import Listener

def on_press(key):
    print(f'按键 {key} 被按下')

def on_release(key):
    print(f'按键 {key} 被释放')
    if key == Key.esc:
        # 停止监听
        return False

# 创建监听器
with Listener(
        on_press=on_press,
        on_release=on_release) as listener:
    listener.join()
```

### 高级用法

#### 特殊按键处理

```python
from pynput.keyboard import Key, Listener

def on_press(key):
    try:
        print(f'字母键 {key.char} 被按下')
    except AttributeError:
        print(f'特殊键 {key} 被按下')

def on_release(key):
    if key == Key.esc:
        return False

with Listener(
        on_press=on_press,
        on_release=on_release) as listener:
    listener.join()
```

#### 热键注册

```python
from pynput.keyboard import Key, Listener, HotKey

# 定义热键功能
def on_activate():
    print('热键被触发！')

# 创建热键
hotkey = HotKey(
    Key.f1,  # 触发键
    on_activate  # 触发函数
)

def for_canonical(f):
    return lambda k: f(hotkey.canonical(k))

# 监听键盘
with Listener(
        on_press=for_canonical(hotkey.press),
        on_release=for_canonical(hotkey.release)) as listener:
    listener.join()
```

## 实际应用示例

### 1. 自动化文本输入器

```python
from pynput.keyboard import Controller, Key
import time

keyboard = Controller()

# 等待5秒，以便切换到目标窗口
time.sleep(5)

text = """
这是一个自动输入的示例。
可以用来快速输入重复性文本。
"""

keyboard.type(text)
```

### 2. 简单的按键记录器

```python
from pynput.keyboard import Listener
import logging

logging.basicConfig(
    filename='key_log.txt',
    level=logging.DEBUG,
    format='%(asctime)s: %(message)s'
)

def on_press(key):
    logging.info(str(key))

with Listener(on_press=on_press) as listener:
    listener.join()
```

### 3. 游戏辅助工具（示例：自动连发）

```python
from pynput.keyboard import Controller, Key, Listener
import threading
import time

keyboard = Controller()
auto_fire = False

def auto_fire_thread():
    while True:
        if auto_fire:
            keyboard.press('z')
            time.sleep(0.1)
            keyboard.release('z')
            time.sleep(0.1)

def on_press(key):
    global auto_fire
    if key == Key.f1:
        auto_fire = not auto_fire
        print(f'自动连发: {"开启" if auto_fire else "关闭"}')

# 启动自动连发线程
threading.Thread(target=auto_fire_thread, daemon=True).start()

# 监听键盘
with Listener(on_press=on_press) as listener:
    listener.join()
```

:::note
- 在某些系统上（特别是macOS），可能需要额外的权限才能使用键盘监听功能。
- Linux上似乎只支持X协议，或uniput必须以root运行
:::