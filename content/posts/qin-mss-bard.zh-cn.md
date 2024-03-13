---
title: "用秦九韶算法求解最大子数组和问题"
date: 2023-12-23T20:44:33+08:00
draft: false
---

> 問有兩尖田一段，其尖長不等，兩大斜三十九步，兩小斜二十五步，中廣三十步，欲知其積幾何？
>
> 術曰：以少廣求之，翻法入之。置半廣自乘為半冪，與小斜冪相減相乘為小率， 以半冪與大斜冪相減相乘為大率。以二率相減餘自乘為實，併二率倍之為從上廉，以一為益隅，開翻法三乘方得積。

一直以来，有些朋友对函数式编程不屑一顾。他们会觉得，函数式编程只不过是增加了一些无谓的抽象，而代价则是性能的下降。持这种观点的朋友，恐怕是肯定没有读过 Richard Bird 所著 [_PEARLS OF FUNCTIONAL ALGORITHM DESIGN_](https://www.cambridge.org/core/books/pearls-of-functional-algorithm-design/B0CF0AC5A205AF9491298684113B088F). 这本书讨论的内容确实太过生涩，我也没有完整读过一遍。

幸运的是，Richard Bird 在 1989 年写过一篇仅仅只有 5 页的论文 [_Algebraic identities for program calculation_](https://academic.oup.com/comjnl/article/32/2/122/543545?login=false). 这篇论文用一个著名的问题--最大子数组和[^1]（Maximum subarray problem）淋漓尽致地展现了函数式编程思维对算法问题的独特理解：最大子数组和问题可以用秦九韶算法求解。

## 最大子数组和问题

最大子数组和问题的目标是在数列的一维方向找到一个连续的子数列，使该子数列的和最大。例如，对一个数列 `[−2, 1, −3, 4, −1, 2, 1, −5, 4]`，其连续子数列中和最大的是 `[4, −1, 2, 1]`, 其和为 6 [^2]。

显然，我们有平凡的算法，即枚举所有的子数组，然后求最大值：

```python
def mss(arr):
    r = 0
    for end in range(len(arr) + 1):
        for start in range(0, end):
            s = 0
            for k in range(start, end):
                s += arr[k]
            r = max(s, r)
```

这个程序用 Haskell 写会简洁一些：

```Haskell
import Data.List (tails, inits)

mss = max . sum . segs
segs = concat . map tails . inits
```

其中 `tails` 是“后缀”，`inits` 是“前缀”：

```Haskell
tails [1, 2, 3] = [[1, 2, 3], [2, 3], [3], []]
inits [1, 2, 3] = [[], [1], [1, 2], [1, 2, 3]]
```

易见，“所有前缀的所有后缀”，或者“所有后缀的所有前缀”就是一个列表所有的子列表。

现在我们回到算法本身。这个平凡算法的时间复杂度是 $O(n^3)$. 把所有的子数组枚举出来需要 $O(n^2)$，对每一个子数组求和又需要 $O(n)$，所以就构成了 $O(n^3)$.

事实上，最优秀的算法在 $O(n)$ 时间复杂度内就可以解决这个问题。Bird 的论文正是在讨论如何从平凡的算法出发，_推导_ 出最优秀的算法。首先，对算法稍作变形：

```python
def mss(arr):
    m = 0
    for end in range(len(arr) + 1):
        m = max(max_tails(arr, end), m)

def max_tails(arr, end):
    m = 0
    for start in range(0, end):
        s = 0
        for k in range(start, end):
            s += arr[k]
        m = max(m, s)
    return m
```

这似乎是一个平凡的变形，我只是把三层循环的里面两层拆出来了。`max_tails` 函数会返回数组 `arr` 里 `[0, end)` 区间的最大后缀和。我们用 $\uparrow$ 来表示最大值函数，`max_tails` 的函数式版本，或数学式版本，可以写成：

$$
\begin{aligned}
\textsf{arr\[0:end\]} &\equiv \[ x_1, x_2, \cdots, x_n \] \\\\
m &= (\sum_{i=1}^n x_i) \uparrow (\sum_{i=2}^n x_i) \uparrow \cdots \uparrow (x_n) \uparrow 0
\end{aligned}
$$

这第一眼看上去十分令人费解，下面用例子来说明。首先，考虑 `max_tails([1, 2, 3], 3)`, 也就是 `[1, 2, 3]` 的所有后缀和的最大值。

`max_tails` 的计算可以写成：

```text
(1 + 2 + 3) ↑ (2 + 3) ↑ (3) ↑ 0
```

也就是说，`max_tails` 会计算每个后缀和，并把它们的最大值求出来。当把最大值写成一个二元函数 $\uparrow$ 的时候，自然就可以用它连接每个后缀和以构成最后的表达式。

在朴素算法中，求后缀和的最大值需要 $O(n^2)$ 的复杂度。下面，我们用秦九韶算法将这个复杂度降低到 $O(n)$.

## 秦九韶算法

秦九韶在其著作《数术九章》中，提出了一种求高次方程近似解的方法。其中的一个部分被近现代研究者称作“秦九韶算法”。

考虑 $n$ 次多项式

$$
f(x) = \sum_{i=0}^{n} a_i x^{i}
$$

对于确定的 $t$, 有几种方法求 $f(t)$ 的值。最平凡的方法，就是分别求每项的值，再加起来。这种算法需要 $O(n^2)$ 次乘法。秦九韶算法注意到：

$$
\begin{aligned}
& a_n x^n + a_{n-1} x^{n-1} + \cdots + a_0 \\\\
= & ((a_n x^{n - 1}) + (a_{n-1} x^{n - 2}) + \cdots + a_{1}) x + a_0 \\\\
= & \cdots \\\\
= & (((a_n x + a_{n - 1}) x + \cdots) x + a_{1})x + a_0
\end{aligned}
$$

这可以写成一个迭代形式：

$$
\begin{aligned}
s_0 &= a_n \\\\
s_{i + 1} &= s_i t + a_{n - i - 1}
\end{aligned}
$$

这样一来 $s_n = f(t)$. 在这个计算中，乘法计算了 $n$ 次，加法也计算了 $n$ 次。

看上去，秦九韶算法只是一个 _数值计算_ 算法。它和 `max_tails` 函数有什么关系呢？我们要考虑“广义”的秦九韶算法形式。

首先，这个算法的输入不一定要是多项式，如下的式子仍然可以用秦九韶算法改写：

$$
\begin{aligned}
& x_1x_2x_3 + x_2x_3 + x_3 + 1 \\\\
=& (((x_1 + 1) x_2) + 1) x_3 + 1
\end{aligned}
$$

其次，秦九韶算法考虑的运算不一定是 $(+, \cdot)$, 这个算法提取公因式的时候，其实是在用乘法分配率改写算式。考虑任意的两个函数 $(\oplus, \odot)$，只要它们满足如下条件[^3]，那就可以运用秦九韶算法：

+ $\odot$ 对 $\oplus$ （右）分配
$$
\begin{aligned}
a \cdot c + b \cdot c &= (a + b) \cdot c \\\\
(a \odot c) \oplus (b \odot c) &= (a \oplus b) \odot c \\\\
\end{aligned}
$$
+ 存在 $\odot$ 的（左）单位元，即存在 $a$，使得 $\forall x, a \odot x = x$

以刚才的表达式为例：

$$
\begin{aligned}
& ((x_1 \odot x_2) \odot x_3) \oplus (x_2 \odot x_3) \oplus (x_3) \oplus (1) \\\\
= & ((((x_1 \oplus 1) \odot x_2) \oplus 1) \odot x_3) \oplus 1
\end{aligned}
$$

其中 $1$ 是 $\odot$ 的左单位元。

## Kadane 算法

刚才已经说过了，`max_tails` 可以写成：

$$
\begin{aligned}
\textsf{arr\[0:end\]} &\equiv \[ x_1, x_2, \cdots, x_n \] \\\\
m &= (\sum_{i=1}^n x_i) \uparrow (\sum_{i=2}^n x_i) \uparrow \cdots \uparrow (x_n) \uparrow 0
\end{aligned}
$$

这里的 $\uparrow$ 可以被看作 $\oplus$，而 $+$ 可以被看作 $\odot$. 加法存在单位元 $0$，而分配性也是显然的：

$$
(a \uparrow b) + c = (a + c) \uparrow (b + c)
$$

继续以 `[1, 2, 3]` 为例，

```text
  (1 + 2 + 3) ↑ (2 + 3) ↑ (3) ↑ 0
= ((1 + 2) ↑ 2 ↑ 0) + 3 ↑ 0
= (((1 ↑ 0) + 2) ↑ 0) + 3 ↑ 0
= ((((0 + 1) ↑ 0) + 2 ↑ 0) + 3) ↑ 0
```

根据这个性质，我们用秦九韶算法改写 `max_tails`:

```python
def step(s, a):
    # (s + a) ↑ 0
    return max(s + a, 0)

def max_tails(arr, end):
    s = 0
    for i in range(end):
        s = step(s, arr[i])
```

显然，改写后的 `max_tails` 的时间复杂度是 `O(n)`. 不仅如此，我们注意到：

$$
\begin{aligned}
& \textsf{max\\_tails}(\textsf{arr}, \textsf{end} + 1) \\\\
=\ & \textsf{step}(\textsf{step}(\cdots,\textsf{arr}\[\textsf{end} - 1\]), \textsf{arr}\[\textsf{end}\]) \\\\
=\ & \textsf{step}(\textsf{max\\_tails(arr, end)}, \textsf{arr}\[\textsf{end}\])
\end{aligned}
$$

由于 `mss` 需要求 `[1, end + 1)` 的 `max_tails`，所以改写后，时间复杂度是 $O(n^2)$. 但上面的发现告诉我们，`max_tails(arr, i)` 的值可以由 `max_tails(arr, i - 1)` 的值求得。这立刻就可以用来得到一个 $O(n)$ 的 `mss` 函数：

```python
def step1(state, a):
    (s, m) = state
    s_next = step(s, a)
    return (s_next, max(m, s_next))

def mss(arr):
    state = (0, 0)
    for a in arr:
        state = step1(state, a)
    return state[1]
```

毫无疑问，这就是大名鼎鼎的 Kadane 算法。

## 原始推导

Bird 教授将他的研究总结为一段话：

> I was interested in the specific task of taking a clear but inefficient functional program, a program that acted as a specification of the problem in hand, and using equational reasoning to calculate a more efficient one. [^4]

也就是说，Bird 教授喜欢从一个简单、清晰，却不高效的函数式程序出发，通过一般的规则对程序进行优化，从而得到一个更高效的程序。Bird 教授从最直觉的程序出发，一步步推导之后，甚至可以得到 KMP 算法这种著名算法。Bird 教授论文中对最大子数组和问题的原始推导是[^5]：

```text
mss = { by definition }                                     
      max . map sum . segs                                  O(n³)
    = { by definition of segs }
      max . map sum . concat . map tails . inits            O(n³)
    = { map promotion }
      max . concat . map (map sum) . map tails . inits      O(n³)
    = { definition of max and fold promotion }               
      max . map max . map (map sum) . map tails  . inits    O(n³)
    = { map distributivity }
      max . map (max . (map sum) . tails) . inits           O(n³)
    = { Horner's Rule }
      max . map (foldl (⊗) 0) . inits                       O(n²)
    = { scan theorem }
      max . scanl (⊗) 0                                     O(n)
    = { fold-scan fusion }
      fst . foldl (⊙) (0, 0)                                O(n)
    where a ⊗ b = (a + b) ↑ 0
          (u, v) ⊙ t = (w ↑ u, w), w = v ⊗ t
```

仅仅用了 8 条等式就得到了最后的算法，可以说是神乎其技。更可贵的是，无论是秦九韶算法还是最后的优化，都在推导过程中被展示为直接的定理。而这些定理就是关于函数式编程中常用函数（如 `map`, `foldl`, `concat`）的恒等式。例如秦九韶算法可以直接表示为：

```text
fold (⊕) b . map (fold (⊗) a) . tails = fold (⊙) a
u ⊙ v = (u ⊗ v) ⊕ a
```

那么，为什么函数式编程可以方便地进行这种程序推导呢？因为函数式编程使得我们更加地注意到程序的 *代数* 性质。Bird 教授还写过一本书，名字就叫做 [_The algebra of programming_](https://dl.acm.org/doi/10.5555/248932).

## 不可爱的另一面

如果读者尝试着把上面的代码交到 [LeetCode](https://leetcode.com/problems/maximum-subarray/) 里，也许会发现它会在一些测试上失败。这当然不是算法错了，而是最大子数组和问题的表述不同。

+ (LeetCode): 给定一个数组，给出它的最大子数组和。
+ (Programming Pearls): 给定一个数组，给出它的最大子数组和。如果数组的最大子数组和为负，那么返回 0.

我们的问题是 Programming Pearls 版本的，它永远不会返回负的最大子数组和。

如果要处理 LeetCode 版本的题目，一个简单的修复是：

```haskell
mssNeg l
  | r == 0    = max l
  | otherwise = r
    where r = mss l
```

这种做法略显笨重。我们能够对 Bird 教授的推导过程作修改，从而推导出能处理负数情况的算法吗？对于这个问题，我没有给出什么理想的解决方案。因为 Bird 教授的第一条式子就错了。

```haskell
--sum [] = 0
mss = max . sum . segs
```

`segs` 的结果中一定存在空表，而空表在用 `sum` 求值之后，就会得到 0. 所以显然，这个式子就无法处理负数最大和的情况。又考虑到在这里的秦九韶算法中，`0` 是 `+` 的单位元，我认为要修改到能处理负数的算法，需要的可能是完全推倒重来。

这也许就是函数式程序推导不可爱的一面。函数式程序推导依赖的是程序的代数性质，而这种代数性质有时候是脆弱的，可能问题稍加修改后，原有的代数性质就消失了。对命令式程序来说，也许它本身就不依赖这些代数性质，所以一点平凡的改变就可以 work，但对函数式、特别是用函数式程序推导推导出的程序来说，这种修改并不平凡。

> 可以发邮件问问 Bird 教授的看法吗？

也许可以吧。Bird 教授在 2022 年 4 月 4 日永远地离开了我们，有没有一个能向天堂寄信的邮局呢……


[^1]: Bird 把它叫做“最大子段和”（maximum segment sum problem），这也许是因为问题是用列表而非数组给出的
[^2]: 摘自维基百科
[^3]: 对于加法和乘法这种既交换又结合的函数来说，向左和向右提取公因式都是可以的。对于不交换的函数来说，向左和向右提取公因式得到的是不同的算法，需要不同的前提来保证正确性。
[^4]: Pearls of Functional Algorithm Design, preface, pp. ix
[^5]: 秦九韶算法在西方一般称为 Horner's Rule
