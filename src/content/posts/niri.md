---
title: niri快速配置
published: 2025-08-29
tags: ["WM", "niri"]
description: "niri是一个Wayland窗口管理器，采用滚动平铺的窗口布局。并且提供内置的截图、屏幕录制、丰富的触控板和鼠标手势支持等功能"
category: 'WM'
draft: false
---
# niri快速配置

::github{repo="YaLTeR/niri"}

## 什么是niri
> niri是一个由Rust编写Wayland窗口管理器，采用滚动平铺的窗口布局。并且提供内置的截图、屏幕录制、丰富的触控板和鼠标手势支持等功能。

### 特性
- 类似 GNOME 的动态工作区
- - 提供Overview模式，可缩放查看工作区和窗口
- 通过 xdg-desktop-portal-gnome 支持显示器和窗口的屏幕录制
- - 可以在屏幕录制时屏蔽敏感窗口
- - 支持动态切换录制目标
- 支持触摸板和鼠标手势
- 可将窗口分组为标签页
- 可配置的布局：间距、边框、支柱、窗口大小
- 支持 Oklab 和 Oklch 的渐变边框
- 支持自定义着色器的动画效果
- 配置文件支持实时重载REDME
> 译自niri github REDME

## X11支持
要写在前面的是关于niri对于x11的支持情况。实际上，niri的作者直接表明[X11 is very cursed](https://yalter.github.io/niri/Xwayland.html#using-xwayland-run-to-run-xwayland)，所以niri暂时不会有内置的x11兼容方案，不同于Hyprlnd采用`wlroots`实现内置`XWayland`的解决方案，niri目前仅能通过第三方的软件支持运行`XWayland`，如`xwayland-satellite`、`xwayland-run`等 (下个版本niri将会内置xwayland-satellite，曲线救国了算是)，如果你的工作区离不开`X11`，或需要高性能的`X11`环境,niri可能不那么适合你😮‍💨

### xwayland-satellite解决方案
1. 根据[niri wiki](https://yalter.github.io/niri/Xwayland.html#using-xwayland-run-to-run-xwayland)介绍，首先通过你的包管理器下载`xwayland-satellite`

```bash
# Arch 需要启用extra存储库
sudo pacman -S xwayland-satellite
```

2. 设置xwayland-satellite为自启动
```
// ~/.config/niri/config.kdl
spawn-at-startup "xwayland-satellite"
```
> 关于这里语句的意义，我们在接下来的内容将会介绍

3. 配置显示环境变量
首先运行一次`xwayland-satellite`，观察输出，找到显示序号`Xwayland :n`：
```bash
INFO  xwayland_satellite         > Connected to Xwayland on :2
```
接着将其写入配置文件
```
// ~/.config/niri/config.kdl
environment {
    DISPLAY ":2"
}
```
或者，你还可以使用`xwayland-satellite :12`来要求xwayland-satellite使用指定的DISPLAY number

## 配置
niri 中 `Input`，`Outputs`，`Key Bindings`，`Switch Events`，`Layout`，`Named Workspaces`，`Miscellaneous`，`Window Rules`，`Layer Rules`，`Animations`，`Gestures`，`Debug Options`都支持在配置文件 `~/.config/niri/config.kdl` 中更改

配置文件使用kdl (递归命名`KDL Document Language`)，这是一个比较小众的配置文件语言，它具有类似`XML`的节点语法，在这里，我们不深入讨论KDL的具体语法，实际上，如果你使用过`XML`、`JSON`等配置文件，你可以很快掌握kdl的基本语法

下面，我将介绍一些常用的配置，更具体的内容可以自行在[wiki](https://github.com/YaLTeR/niri/wiki/Configuration:-Introduction)中查阅

### 开机启动
```
// ~/.config/niri/config.kdl
spawn-at-startup "xwayland-satellite"
spawn-at-startup "qs" "-c" "dms"
spawn-at-startup "~/.config/niri/scripts/Polkit.sh"
```
设置软件开机自启，需要传参的用空格和引号分开，也可以给入一个文件位置来打开

### Input

<details>
    <summary> 查看折叠的代码 </summary>

```
// ~/.config/niri/config.kdl
input {
    keyboard {
        repeat-delay 300
        repeat-rate 40

        numlock
    }

    touchpad {
        // off
        tap
        // dwt
        // drag false
        // drag-lock
        natural-scroll
        // accel-speed 0.2
        // accel-profile "flat"
        // scroll-method "two-finger"
    }

    mouse {
        // off
        // natural-scroll
        // accel-speed 0.2
        accel-profile "flat"
        // scroll-method "no-scroll"
    }

    // warp-mouse-to-focus

    // focus-follows-mouse max-scroll-amount="0%"
    workspace-auto-back-and-forth
}
```

</details>

#### Keyboard
1. `repeat-delay`: 长按某个键多久后触发连点
2. `repeat-rate`：每秒多少个字符
3. `numlock`：开机时自动打开数字键

#### touchpad
> `niri`触控板支持三指左右滑动滚动窗口，三指上下滑动切换工作空间，四指快速滑动打开`Overview`（默认快捷键为`Mod`+`O`）
1. `tap`：点击功能
2. `dwt`：disable-when-typing，打字时禁用
3. `drag`：点击后拖动，值为`true`或`false`
4. `drag-lock`：拖动时放手保持拖动一段时间
5. `natural-scroll`：反转双指滚动方向，设置时才是符合直觉的滚动方式

#### mouse
1. `natural-scroll`：同上，不设置时反而是符合直觉的滚动方式
2. `accel-speed`：速度
3. `accel-profile`：鼠标加速方案，值为`adaptive`表加速 或 `flat`不加速

#### 其他
1. `warp-mouse-to-focus`：切换焦点窗口时移动鼠标，默认值为`mode="center-xy"`，如果切换焦点后光标不在窗口内就跳转光标到窗口中心，反之不移动。`mode="center-xy-always"`为始终跳转
2. `focus-follows-mouse`：使焦点自动根据光标位置切换，可以设置`max-scroll-amount="n%"`，设置后如果焦点跟随光标会导致视图滚动超过设定的量，则不会切换焦点
3. `workspace-auto-back-and-forth`：使用索引切换工作空间时，如果通过索引切换到当前工作空间，将会自动切换到上一个工作空间


### Cursor
```
// ~/.config/niri/config.kdl
cursor {
    xcursor-theme "Bibata-Modern-Ice"
    xcursor-size 24
}
```
设置光标主题和大小

### Overview
```
// ~/.config/niri/config.kdl
overview {
    backdrop-color "#282828"
    workspace-shadow {
        off
    }
}
```
设置Overview背景色

### Layout
<details>
    <summary> 查看折叠的代码 </summary>

```
// ~/.config/niri/config.kdl
layout {
    gaps 8
    center-focused-column "never"

    preset-column-widths {
        proportion 0.2
        proportion 0.4
        proportion 0.6
        proportion 0.8
    }
    preset-window-heights {
        proportion 0.25
        proportion 0.5
        proportion 0.75
        proportion 1.0
    }

    default-column-width { proportion 0.5; }

    focus-ring {
        off
        width 4
        active-color "#7fc8ff"
        inactive-color "#505050"
        active-gradient from="#C65B82" to="#09157B" angle=135
        inactive-gradient from="#C65B82" to="#09157B" angle=135 relative-to="workspace-view"
    }

    border {
        // off
        width 4
        // active-color "#ffc87f"
        // inactive-color "#505050"
        urgent-color "#9b0000"
        active-gradient from="#C65B82" to="#09157B" angle=135 relative-to="workspace-view" in="oklch longer hue"
        inactive-gradient from="#505050" to="#808080" angle=45 relative-to="workspace-view"
    }

    shadow {
        //on
        softness 1
        spread 1
        offset x=0 y=5
        color "#0007"
    }

    struts {
        // left 64
        // right 64
        // top 64
        // bottom 64
    }
}
```

</details>


1. `gaps`：窗口之间的间隔
2. `center-focused-column`：控制窗口在显示器上居中的行为
- - `never`：（默认值）不居中，当聚焦的列会显示屏幕的左边缘或右边缘。
- - `always`：被聚焦的列将始终居中显示。
- - `on-overflow`：当聚焦的列与之前聚焦的列无法同时显示在屏幕上时，会将该列居中显示。


#### column
1. `preset-window-widths` 使用快捷键(默认是`Mod`+`R`)调整窗口时的宽度
2. `preset-window-heights` 使用快捷键(默认是`Mod`+`Shift`+`R`)调整窗口时的高度
3. `default-column-width` 打开新窗口时的默认宽度
- 值为`proportion 0.66667` 表对于的屏幕的占比，比如示例的2/3
- 或 `fixed 1280` 表固定的像素数

#### focus-ring、border
> focus-ring和border的区别在于：focus ring只围绕活动窗口绘制，而borders会围绕所有窗口绘制，并且会影响窗口的大小
1. `width`：显然是宽度，单位为像素，但是可以填入小数，在设置了缩放后会被计算
2. `active-color`：焦点窗口装饰的颜色
3. `inactive-color`：所有其他窗口装饰的颜色
4. `active-gradient`：`active-color`的渐变版
5. `inactive-gradient`：`inactive-color`的渐变版
- - 渐变颜色可以添加`relative-to="workspace-view"`值表示全部窗口一起渐变，不设置则为各自渐变。同时它们支持类似`css`的渐变插值，你可以使用类似 `in="srgb-linear"` 或 `in="oklch longer hue"` 的语法来设置
6. 所有的颜色支持以下的方式定义
- - CSS 命名颜色：如 `red`
- - RGB 十六进制：如 `#rgb`、`#rgba`、`#rrggbb`、`#rrggbbaa`
- - CSS 类似表示法：如 `rgb(255, 127, 0)`、`rgba()`、`hsl()` 等几种格式。

#### 窗口圆角
看到这儿，你可能迫不及待的想要设置窗口圆角了，我就在这里提前告诉你，我们可以通过设置窗口规则来定义圆角：
```
window-rule {
    draw-border-with-background false
    geometry-corner-radius 12
    clip-to-geometry true
}

prefer-no-csd
```
1. 通过不设置匹配来匹配全部的窗口
2. `draw-border-with-background`：border装饰窗口并不只是加一个边框，而是填充一个背景，如果你使用某个背景透明的软件就会发现，设置这项为`false`可以取消背景
3. `geometry-corner-radius`：圆角半径，也可以设置4个值来表示各个j角的圆角
4. `clip-to-geometry`：`geometry-corner-radius`只会设置`focus-ring`和`border`的圆角，这项参数将同时应用圆角至窗口
5. `prefer-no-csd`：有些软件可能自带窗口圆角和阴影，为了统一的效果，我们可以设置这项要求软件取消 client-side decorations

#### struts
```
struts {
        left 64
        right 64
        top 64
        bottom 64
    }
```
设置窗口四边的空白，也可以设置为负值
