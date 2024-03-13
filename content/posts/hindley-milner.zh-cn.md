---
title: 十分钟搞懂 Hindley-Milner 类型系统
author: ayanamists
tags: 
  - 类型推理
  - 编程语言
date: 2021-11-25
categories:
  - 学术
math: true
---
## 类型推理

### 什么是类型推理

程序员是懒惰的生物。很多时候，我们并不想啰嗦地给出类型：

```fsharp
let t = 1 + 1
let f x = x + 1
```

这里的 `t`、`f`、`x` 三个变量，每个的类型都是很明确的。当然，更多的时候，显示地给出类型是更好的。

还有一些时候，给出类型并不是很容易的事情，例如：

```fsharp
let s f g x = f x (g x)
let k x y = x
let i = s k k
let ϕ = s (k (s i)) k
```

请问，ϕ 的类型是什么？

ϕ 的类型是可以计算出来的，但是非常麻烦。自然的想法是找到一个算法，让计算机代替人脑计算。

事实上，把这段代码送到 haskell 或者 f# 的 repl 里，他们会告诉你ϕ的类型是`p -> (p -> q) -> q`. 而他们所采用的算法，就是 Hindley-Milner 类型推理算法。

### 类型推理算法

一言以蔽之，类型推理算法就是解方程。方程的建立过程如下：

1. 每个函数的参数，以及每次函数调用的返回值用一个类型变量`xᵢ`表示。
2. 在函数调用的时候建立方程。`f x`这个函数调用可以用以下的规则建立方程：

$$
\frac{f:a \rightarrow b\quad x:t_1 \quad (f\ x):t_2}{t_1 \rightarrow t_2 = a \rightarrow b}
$$

例如，这样的函数：

```fsharp
let f a b = a + b
```

就可以建立这样的方程：

$$
\begin{aligned}
a:\\ &t_1 \\\\
b:\\ &t_2 \\\\
f:\\ &t_3 \\\\
a + b:\\ &t_4 \\\\
(+):\\ & \mathsf{int} \rightarrow \mathsf{int} \rightarrow \mathsf{int} \\\\
t_1 \rightarrow t_2 \rightarrow t₄ =&\ \mathsf{int} \rightarrow \mathsf{int} \rightarrow \mathsf{int} \\\\
t_3 =&\\ t_1 \rightarrow t_2 \rightarrow t_4
\end{aligned}
$$

通过解下面的两个方程，就可以得到每个变量的类型了。现在的问题是，如何解这样的方程呢？

### Unification

unification是来自于自动定理证明的算法，它可以用来解上述的方程。它背后有一系列关于替换的理论。篇幅所限，我们不会在这里讨论这些理论，而是用很短的篇幅来介绍最自然的算法[^1]。

首先，上面的方程最多会有两种形式的项参与。一种是类型变量`xᵢ`，一种是箭头`xᵢ → xⱼ`. 事实上，箭头`xᵢ→ xⱼ`可以看作`f(xᵢ, xⱼ)`，这样我们就有两种形式：

1. 类型变量。
2. `f(v₀, v₁, v₂, ...)`的形式。上面的`int`可以看作`int()`

算法如下：

遍历每一个方程：

1. 进行自环检查，如果两边出现了
    $$
    x_i = \cdots x_i \cdots
    $$
  （例如 $x_i = f(x_i)$ ）的形式，那么算法失败。

2. 如果方程的形式为:
$$
\begin{aligned}
  x_i &= \star \\\\
  \star &= x_i
\end{aligned}
$$  
   那么无论 $\star$ 是什么，都用 $\star$ 替换掉*还没有遍历过的方程*中的 $x_i$

3. 如果不是情况1.，即
   $$
   f(x_0, x_1, x_2, \cdots) = g(y_0, y_1, y_2, \cdots)
   $$
   那么，
   * 如果 $f ≠ g$，算法失败。
   * 如果 $f = g$，加入 $n$（ $n$ 为 f 的参数个数 ）个新的待遍历方程 $x_i = y_i$.

[^1]: 文末给出的实现采用的是基于并查集的算法

## Let多态

现在我们已经有了一个类型推理算法。考虑这样的函数：

```fsharp
let f x = x
```

上面的推理算法会推出`f:x₀ → x₀`，这里有一个自由变量 `x₀`. 采用 System F 的类型记法，对这样的函数进行“泛化”：

$$
\frac{f:\psi(x_0)}{f:\forall x_0.\psi(x₀)}
$$

这样一来，`f` 的类型就成了“多态类型”，可以进行“例化”：

$$
\frac{f:\forall x.\psi(x)}{f:\psi(a)}
$$

一个麻烦的问题是：在什么时候进行泛化和例化呢？

自然的想法是，在`fun x -> ...`这样的 lambda 表达式外部进行泛化，在引用一个泛化过的变量的时候进行例化。E.g.

```fsharp
// 以下都是类型正确的项
(λ x. x) 10
(λ x. x) true
```

这样带来的问题是，只有 λ 表达式可以是“多态类型”的，在 lambda 内部的变量仍然只能是单一类型的。

```fsharp
(λf.(λ t. f true)(f 0))(λx.x)
```

以上面的表达式为例，即使 `λx.x` 是多态类型，`f` 也不能被推理为多态类型。实际上，`f`在`f 0`被推理为`int → x₀`，在`f true`被推理为`bool → x₀`. 这样会产生类型错误。

回忆一下，在F#或者ocaml中，`f`真正被赋予泛型的方法是：

```fsharp
let f x = x in
    let t = f 0 in f true
```

在无类型或简单类型λ表达式中，上面的两段代码完全等价，`let`只是语法糖。但是在ML系语言中，`let`却不是语法糖。事实上，ML系语言中，进行“泛化”的时机恰恰是在`let`绑定这里！

也就是说：

1. 在表达式`let var = e₁ in e₂`中，`e₁`的类型被泛化。泛化方法为：
    * 当前环境 $Γ$ 中的自由变量为 $F(Γ)$
    * $e_1$ 的自由变量为 $F(e_1)$
    * 遍历每一个在 $F(e_1) - F(Γ)$ 的自由变量 $x_i$，$e_1:t$ 变为 $e_1:∀x_i.t$
2. 每当从当前环境 $Γ$ 中引用一个泛型变量 $e:t_1$ 时，用一个新的类型变量例化 $e:t_1$ 直到 $t_1$ 不再含有 $∀$

而这就是 Hindley-Milner 算法的基本思想。至于具体实现则有algorithm J和algorithm W，请参考维基百科或者 Milner 的[原始论文](https://www.sciencedirect.com/science/article/pii/0022000078900144?via%3Dihub)。[我的实现](https://github.com/ayanamists/TypeInf)采用了alogrithm J.

## 值限制

如果你真的试了之前的代码：

```fsharp
let s f g x = f x (g x)
let k x y = x
let i = s k k
let ϕ = s (k (s i)) k
```

你会发现，在Haskell中，这是没问题的：

![1](https://pic.imgdb.cn/item/619f4b162ab3f51d911a8f2f.jpg)

但是在F#中，这会报错：

![2](https://pic.imgdb.cn/item/619f4b552ab3f51d911ab740.jpg)

为什么要报错呢？这就要从一个更加tricky的程序说起。F#中有可变类型：

```fsharp
let mutable x = 10
x <- 20
printfn "%A" x
```

如果给出这样的程序：

```fsharp
let mutable x = []
x <- [3]
List.map (fun x -> x = true) x
```

那么第一行会报错。但是，假设第一行不报错，第二行会不会报错呢？我们要仔细研究一下。

首先，`<-`的类型虽然F#被隐去了，但是F#有一套[已经弃用](https://aka.ms/fsharp-refcell-ops)的操作符`!`、`:=`和`ref`，它们的类型是：

```fsharp
ref  : 'a -> 'a ref
(!)  : 'a ref -> 'a
(:=) : 'a ref -> 'a -> unit
```

上面的程序写作：

```fsharp
let x = ref []
x := [3]
List.map (fun x -> x = true) x
```

在第二行被引用的时候，`x`的类型是`'t list ref`，也就是`∀t.ref[list[t]]`，它应该被例化为`ref[list[b]]`，这样一来得到两个等式：

$$
\begin{aligned}
\mathsf{ref}[\mathsf{list}[b]] =& \mathsf{ref}[a] \\\\
a =& \mathsf{list}[\mathsf{int}]
\end{aligned}
$$

上面的方程得到 $b = \mathsf{int}, a = \mathsf{list}[\mathsf{int}]$ ，**这是不会报错的**。而且更大的问题是，在第二行的类型检查之后，`b`确实变成了`int`，但这只是 **例化的** `x`，而 $Γ$ 中的`x`，类型一直都是`'t list ref`，也就是说，**第三行照样不会报错！**

这就非常糟糕了，因为它引入了类型系统的 unsound. 第三行毫无疑问会出现运行时错误。

为了解决这个问题，ANDREW K. WRIGHT[^2]引入了值限制 (value restriction)。值限制的含义很简单：仅当`let var = e₁ in e₂`的`e₁`是语法值(syntactic value)的时候，泛化`e₁`。如果`e₁`不是语法值，且他具有不在 $\Gamma$ 中的自由变量，那么报错。

语法值，在ML系语言中就是常量、变量、λ表达式、类型构造器。当然，一个`ref[a]`类型的值不能被视为语法值。这样一来问题显然已经解决了，因为上面的错误本质上是因为副作用的出现，而对一个语法值求值，在ML中不可能出现副作用。

这个解决方案是过近似的，很多正确的程序（比如之前提到的的程序）也会被拒绝。Haskell由于没有副作用，自然也就不存在这个问题。这也是Haskell能够大量的使用类似于`f . g`的函数组合，而ML中不能的一大原因。

[^2]: Simple imperative polymorphism, ANDREW K. WRIGHT

## 量词的位置？

在 System F 中，量词可以出现在一个类型的任意位置。比如 $∀x.(x → ∀y.y)$. 事实上，System F 虽然无法构造 Y 组合子（有悖于强规范性），但却可以编码类似于 无类型 lambda 演算的 $(x x)$ 结构：

$$
λ(x:∀α.α→α).(x (∀α.α→α))x
$$

这个λ表达式的类型是 $(∀α.α→α)→(∀α.α→α)$. 它第一个参数的类型决定了它只能作用于`Λα.λ(x:α).x`这个`id`函数。把它作用到`id`的形式用无类型λ表达式写出来就是：

$$
(λx.xx)(λx.x)
$$

毫无疑问它会被β规约为$λx.x$.

由此可见，System F中λ表达式，其参数的类型可以含有“多态类型”。那么为什么之前我们的类型推理算法没法推理出它的多态类型呢？

这是因为，之前我们的推理算法可以看作是“简单类型λ表达式”的类型推理算法。而HM类型系统则是简单类型λ表达式的扩展。

说到这里，自然会产生一个疑问：System F 可不可以进行类似的推理呢？或者说，我们这样写：

$$
λ(x:t_1).(x\ t_2)x
$$

有没有一个算法，可以得到通过这个表达式得到 $t_1$ 和 $t_2$ 的值呢？

1985年，Hans-J. Boehm的论文 *PARTIAL POLYMORPHIC TYPE INFERENCE IS UNDECIDABLE* 告诉我们，如果加入`fix`组合子，那么上面的问题是不可判定的。1995年，J.B.Wells的论文 *Typability and type checking in System F are equivalent and undecidable* 进一步告诉我们，如果不给出显式的 $t₁$ 和 $t₂$，只要求检查最后的类型，那么这个问题还是不可判定的。

所以，HM 类型系统可以被看成两种 λ 表达式的变种：

* 如果看成简单类型 λ 表达式的变种，那么HM类型系统加入了`let var = e₁ in e₂`这种形式，并且允许`e₁`泛化，实现了参数多态。
* 如果看成System F的变种，那么HM类型系统限制了量词的位置--只能在类型的最前面。这个限制使得HM类型系统可以实现推理，而 System F 无法实现类型推理。

不过，这并不是说，更强的类型系统就不能进行类型推理了。事实上，Haskell 实现了更强的类型推理，而像 Scala、Typescript、Rust 等语言使用的是一种称作 “局部类型推理”（local type inference）的技术，它在程序员给出一些类型标记的基础上，可以消除大部分不必要的类型标记。