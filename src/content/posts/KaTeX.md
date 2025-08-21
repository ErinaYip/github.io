---
title: KaTeX
published: 2023-07-13 21:08:51 
updated: 2023-12-08 12:58:32
description: ''
image: ''
tags: []
category: 'Others'
draft: false
---

# KaTeX语法介绍

KaTeX是一个流行的用于Web上高质量数学排版的渲染库。它与LaTeX语法兼容，但具有自己的一套渲染方程式的规则。下面是一份常用的KaTeX语法指南。


## 基础语法

要使用KaTeX渲染方程式，您可以使用两个美元符号把方程式括起来，就像这样：

```
$f(x) = x^2 - 3x + 5$
```

这将被渲染为：$f(x) = x^2 - 3x + 5$

## 基本数学运算
KaTeX支持广泛的数学运算，包括：
- 指数： `x^n` 或 `x^{n}` 
- 下标： `x_n` 或 `x_{n}` 
- 分数： `\frac{numerator}{denominator}` ，或可选 `\dfrac` 以获得一个更大的分式


例如：
- `$x^{2n}$`
- `$C_{n-1}$`
- `$\frac{3}{4}$`
- `$\dfrac{1}{0}$`

它们将分别被渲染为：

- $x^{2n}$
- $C_{n-1}$
- $\frac{3}{4}$
- $\dfrac{1}{0}$


## 希腊字母
KaTeX支持许多希腊字母，包括：
- Alpha： `\alpha` 
- Beta:  `\beta` 
- Gamma:  `\gamma`  ( `\Gamma` 为大写字母)
- Delta:  `\delta`  ( `\Delta` 为大写字母)
- Epsilon:  `\epsilon` 
- Zeta： `\zeta` 
- Eta:  `\eta` 
- Theta:  `\theta`  ( `\Theta` 为大写字母)
- Iota:  `\iota` 
- Kappa:  `\kappa` 
- Lambda:  `\lambda`  ( `\Lambda` 为大写字母)
- Mu:  `\mu` 
- Nu:  `\nu` 
- Xi:  `\xi`  ( `\Xi` 为大写字母)
- Omicron:  `\omicron` 
- Pi:  `\pi`  ( `\Pi` 为大写字母)
- Rho:  `\rho` 
- Sigma:  `\sigma`  ( `\Sigma` 为大写字母)
- Tau:  `\tau` 
- Upsilon:  `\upsilon`  ( `\Upsilon` 为大写字母)
- Phi:  `\phi`  ( `\Phi` 为大写字母)
- Chi:  `\chi` 
- Psi:  `\psi`  ( `\Psi` 为大写字母)
- Omega:  `\omega`  ( `\Omega` 为大写字母)


例如：
- `$\gamma+\delta=\epsilon$`
- `$\theta+\Theta+\tau=\Pi$`

它们将被渲染为：

- $\gamma+\delta=\epsilon$
- $\theta+\Theta+\tau=\Pi$


## 其他常见语法

除了上述语法之外，KaTeX还支持其他常见的数学运算符和语法，例如：
- 根号： `\sqrt` 
- 积分符号： `\int` 
- 和符号： `\sum` 
- 极限符号： `\lim` 
- 向量符号： `\vec` 
- 绝对值： `\lvert x \rvert` 
- 括号： `( )` ， `[ ]`  和  `{\{  \}}` 

例如：

- `$\sqrt{2+\sqrt{2}}$`
- `$\int_0^1 x^2\, dx$`
- `$\sum_{n=1}^{\infty} 2^{-n} = 1$`
- `$\lim_{x \to 0} \frac{\sin x}{x} = 1$`
- `$\vec{a} \cdot \vec{b} = \lvert a \rvert \lvert b \rvert \cos \theta$`
- `$(a+b)^2=a^2+2ab+b^2$`
- `$[a+b,c+d]=[a,c]+[a,d]+[b,c]+[b,d]$`


它们将分别被渲染为：

- $\sqrt{2+\sqrt{2}}$
- $\int_0^1 x^2\, dx$
- $\sum_{n=1}^{\infty} 2^{-n} = 1$
- $\lim_{x \to 0} \frac{\sin x}{x} = 1$
- $\vec{a} \cdot \vec{b} = \lvert a \rvert \lvert b \rvert \cos \theta$
- $(a+b)^2=a^2+2ab+b^2$
- $[a+b,c+d]=[a,c]+[a,d]+[b,c]+[b,d]$
