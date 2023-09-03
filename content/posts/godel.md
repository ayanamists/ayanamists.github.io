---
title: "用多线程证明 Gödel Incompleteness 是否搞错了什么"
date: 2023-09-01T12:34:12+08:00
draft: false
tags:
    - 数理逻辑
    - 可计算性
categories:
    - 学术
---

{{<  admonition "warning" >}}
本文不是严格的学术论文，可能存在一些错漏之处，见谅。
{{< /admonition >}}

## 前言

Kurt Gödel 的几个著名定理构成了现代逻辑学的基石。其中，Gödel （第一和第二）不完备性定理更是为人所称道。

一直以来，包括笔者在内的很多人，都觉得 Gödel 不完备性定理的证明十分繁琐，有非常复杂的技术细节，所以学习起来相当吃力。在 2023 年的 [∞-type Café 暑期学校](https://m4p1e.github.io/ntype-cafe-summer-school/)上，ocau 老师给出了一个非常优雅和简洁的综合 Gödel 不完备性定理证明讲座。这个讲座让我意识到，Gödel 不完备性定理其实与可计算性，或者说[递归论](https://plato.stanford.edu/entries/recursive-functions/)有着极为密切的联系。

在这篇文章中，我将结合自己对 Gödel 不完备性定理的理解，介绍一个最简单（也许不严格的）Gödel 不完备性定理证明，力图使得普通程序员在学习了简单的一阶逻辑后也能理解。

## 一阶逻辑

一阶逻辑（First order logic, FOL）是本文所讨论的逻辑系统。一阶逻辑有语法层面和语义层面，如果读者没有在学校里学过一阶逻辑，那么可以这么理解：

1. 一阶逻辑是一种语言，和我们常见的编程语言类似，它有一套形式化的语法（Syntax）
    - 一阶逻辑的“语句”，可以被看作一个语法树（Abstract Syntax Tree），或者是一个代数数据类型（ADT）
    - 这个语法树有两层，一层是“命题”（Statement），一层是“表达式”（Expression）
    - 表达式层包括变量、函数（调用）等等，$x$, $f(x)$, $g(x, y)$ 都是表达式
    - 命题层由表达式构成，表达式用等号或关系拼起来构成命题，比如 $f(x_1) = f(x_2)$, $P(x_1, f(x_2))$
    - 命题之间可以用逻辑连接词连接起来，表达特定的含义，比如 $f(x_1) = f(x_2) \land P(x_1, f(x_2))$ 也是一个命题。这样的逻辑连接词还有 $\rightarrow$, $\lor$, $\neg$
    - 命题内的自由变量（free variable）用量词绑定，可以得到新的命题。比如 $\forall x_1 \exists x_2. f(x_1) = f(x_2)$
2. 一阶逻辑这种语言，需要规定其语义，这是通过模型（Model）实现的。相关的内容比较复杂，暂时不做讨论。
3. 一阶逻辑有很多证明系统，比如 [Hilbert System](https://en.wikipedia.org/wiki/Hilbert_system)，这些证明系统和语义是独立的。简单来说，一个证明系统可以通过“计算”给出 $\Gamma \vdash \psi$ 这样的命题 [^1]，其中 $\Gamma$ 代表一个命题的集合，也就是一堆命题，$\psi$ 代表需要证明的命题。
4. 当我们使用一阶逻辑的时候，我们使用的是一阶理论（Theory）. 一个一阶理论由符号表 $S$ 和公理集合 $T$ 构成。符号表规定了所有在命题中可能出现的符号，包括常量、函数符号、关系符号三者；公理集合规定了在这个理论中不证自明的命题，它们是证明系统证明的起点[^2]。例如，著名的 Peano 算术数就可以作为一个一阶理论，它的符号表有：
    - 常量符号 $0$
    - 函数符号 $+$
    - 函数符号 $\times$
    - 函数符号 $S$（表示后继）
    - 关系符号 $\le$

对一阶逻辑的介绍，我们就进行到这里。一阶逻辑有非常丰富的数学性质，这是数理逻辑课程的主要讨论对象。如果读者在阅读本文的过程中感到了不适，那么不妨找一本靠谱的数理逻辑书籍，比如 [1, 2] 等等，学习一下相关知识。

## Gödel 不完备性定理

“完备性”是一个一阶理论的性质。一个一阶理论 $T$ 是完备的，就是说：

> 对任意由该理论符号集合构造的命题 $\psi$，要么 $T \vdash \psi$，要么 $T \vdash \neg \psi$

换句话说，在这个理论里的任何命题，要么能被证明，要么能被证否。

注意到这是一个纯语法概念，它似乎需要指定一个特定的证明系统。不过，各种一阶逻辑的证明系统，如果没有特别构造，那么在证明能力上都是等价的。又因为 Gödel 完备性定理的存在，$T \vDash \psi$ 当且仅当 $T \vdash \psi$，所以这实际上也是语义概念。这样一来，我们一般不需要指定一个证明系统。在 Gödel 不完备性定理的形式化 [3] 中，这种指定才有可能是必要的。

对于一个一阶理论，我们当然期望它是完备的。例如在 Peano 算术中，以下符合直觉的命题可以被证明：

- $\forall n. n \le S(n) $
- $\forall n_1, n_2 . n_1 + n_2 = n_2 + n_1$

而相应地，直觉不成立的命题可以被证否。

如果一阶理论足够简单，不完备性是很容易达成的。例如考虑一个符号表为 $\\{ c, e \\}$，公理为 $\emptyset$ 的一阶理论 $A$，显然有：

- $A \not\vdash c = e$
- $A \not\vdash c \neq e$

直观上看，这是因为理论的公理集太少了。在 $A$ 里加入一条公理（这往往被叫做“扩张”，extension）$c = e$，就能让 $A$ 变为一个完备的理论。

对于任何理论 $T$，如果 $\psi$ 在这个理论中既不能证明，又不能证否（称之为 $\psi$ 独立于 $T$），那么把 $\psi$ 或 $\neg \psi$ 加入 $T$ 的公理，就可以消除这种不完备性。

这似乎揭示了一个美好的图景：我们可以把整个数学建立在某种完备的一阶理论上，在这个终极理论中，任何命题要么能证明，要么能证否。这样一来，模糊、混乱的数学语言就完全可以被清晰、整洁的一阶逻辑所替代……

令人失望的是，1931 年，Gödel 在其发表的论文 "Über formal unentscheidbare Sätze der Principia Mathematica und verwandter Systeme I" [13] 中证明了，只要一个形式系统 $T$ 包含初等算术[^3]，那么在其中就一定有既不能证明，也不能证否的命题。注意，加公理是没有任何用处的，因为加入公理后，不完备性定理的前件仍然满足，仍然可以推导出后件。[^4]

这等于是给我们上面想象的美好图景判了死刑。当然，我们是程序员，不是数学家，这和我们有什么关系呢？

## 可判定性

在 Gödel 于 1931 年发表论文之后，Church 和 Turing 于 1936-1937 年分别提出了关于一阶逻辑的另一重大问题--可判定性的否定证明。

前文已经说过了，一阶逻辑的命题可以被看作一个语法树（或字符串），而一阶理论又是符号表和公理集合，在某些条件下（比如下文要讲的可递归公理化），这都可以看成程序里的数据。

显然我们会问：有没有一个程序 $\texttt{p}$，判断一个命题 $\psi$ 是否可以在 $T$ 内被证明？

这个问题的回答是否定的：**没有这样的程序**。证明的方法我们留到下文再谈。这个事实是稍有些反直觉的，因为一个公理系统中的证明，理论上来讲，是可以**枚举**的。

怎么枚举呢？考虑 Peano 算术和 Hilbert 系统。Peano 算术的公理中，归纳公理是无穷多的，但它有固定模式：

$$
\begin{aligned}
\forall \vec{y_k}. (& (\varphi(0, \vec{y_k})\ \land \\\\
    & (\varphi(x, \vec{y_k}) \rightarrow \varphi(S(x), \vec{y_k}))) \\\\ 
    & \rightarrow \forall x. \varphi(x, \vec{y_k}))
\end{aligned}
$$

这里需要枚举的就是 $k$ 和 $\varphi$. $k$ 自不必说，$\varphi$ 是所有的谓词，它的语法也是确定的，可以用固定的规则产生。不过，直接写一个 `gen` 函数还是相当复杂的。我们这里可以借鉴 Gödel 的做法。

首先，Peano 算术的符号集是无限的，但我们可以分为两个区域：

- 十个逻辑符号（经过简化后）
- 无限个变量符号（记作 $v_0$, $v_1$, $v_2$, ...）

把这两个区域的符号按照先后顺序进行编码，可以给出编码和解码函数。编码函数将符号映射为自然数，解码函数把自然数映射为符号。

```python
# put '∀' to 1 , instead of 0
LOGIC_SYM = [ 'dummy', '∀', '0', 'S', '+', '×', '(', ')', '¬', '→', '=' ]

def decode(n):
    if n <= len(LOGIC_SYM):
        return LOGIC_SYM[n]
    else:
        # append '$' to variable head
        return f"$v_{n - len(LOGIC_SYM)}"

def encode(sym):
    if sym[0] == '$':
        # is variable, start with '$'
        index = sym[3:]
        return int(index) + len(LOGIC_SYM)
    elif sym in LOGIC_SYM and sym != 'dummy':
        return LOGIN_SYM.index(sym)
    else:
        # error
        return -1
```

类似地，一个命题（也就是字符串，字符的列表）也可以进行编码和解码。Gödel 用的技术是这样的：

$$
\sharp [ a_1, a_2, a_3, ..., a_n ] = p_1^{\sharp a_1} \times p_2^{\sharp a_2} \times \cdots \times p_n^{\sharp a_n}
$$

其中，这里的 $\sharp a_i$ 表示我们刚才定义的 `encode(aᵢ)`. 而 $p_i$ 表示第 $i$ 个质数

```python
from sympy import nextprime, Pow, factorint

def encode_str(stmt):
    pᵢ = 2
    res = 1
    for aᵢ in stmt:
        res *= Pow(pᵢ, encode(aᵢ))
        pᵢ = nextprime(pᵢ)
    return res

def decode_str(n):
    """
    n must be of form `Πᵢ pᵢ^{aᵢ}, aᵢ ≠ 0`, or this function is undefined
    """
    res = []
    factors = factorint(n)
    # factors is like { pᵢ : aᵢ }
    for pᵢ, aᵢ in sorted(factors.items()):
        res.append(decode(aᵢ))
    return res
```

`decode_str` 的值域包括了所有的字符串。这样一来，对字符串的遍历就可以用对自然数的遍历实现。

一个字符串可能不是一个合法的命题。所以，还需要一个判断字符串是否是合法命题的函数。只要有了这个函数，就可以枚举所有的命题：

```python
def is_valid_stmt(stmt):
    # this function is hard to write
    pass

def gen_all_stmts():
    n = 0
    while True:
        stmt = decode_str(n)
        if is_valid_stmt(stmt):
            # do something here
        n += 1
```

函数 `is_valid_stmt` 类似于编程语言的 [parser](https://en.wikipedia.org/wiki/Parsing)，虽然写起来比较困难，但肯定是可以写出来的。

类似地，我们可以枚举所有的证明（也就是一个命题列表），进而构造一个函数判断一个命题是否可以被证明：

```python
def is_valid_proof(stmt_l):
    pass

def encode_proof(proof):
    # like encode_str()
    pass

def decode_proof(n):
    pass

def can_prove(ψ):
    n = 0
    while True:
        stmt_l = decode_proof(n)
        if is_valid_proof(stmt_l):
            stmt_last = stmt_l[-1]
            if stmt_last == ψ:
                return True
        n += 1
```

对程序 `can_prove` ，有以下观察：

- 如果列表的最后一个就是要证明的 $\psi$，那么显然，$T \vdash \psi$ 是成立的。
- 我们的枚举遍历了所有的证明，所以，如果 $T \vdash \psi$ ，也就是存在一个符合要求的证明，那么那个证明一定会被找到。
- 如果 $T \not\vdash \psi$，这个算法根本不会停机，它会一直循环下去。

这与停机问题的情况非常类似，在可计算性理论中被称为“半可判定性”（semi-decidable）. 半可判定性是我们下面的证明中所用到的核心概念，值得仔细讨论一下。

> 一个集合[^5] $S$ 是半可判定的，当且仅当存在一个程序 $\texttt{p}$，使得对于所有的 $e \in S$, $\texttt{p}(e) = \texttt{True}$.

半可判定的关键是，它没有讨论当 $e \not\in S$ 时程序是否停机。虽然如果 $\texttt{p}(e) = \texttt{True}$，也就是程序输出 $\texttt{True}$ 的时候，根据当且仅当，$e \in S$. 但如果输入 $e_1 \not\in S$，$\texttt{p}(e)$ 是否停机在定义中是没有提及的。所以，可判定的集合一定是半可判定的集合，而半可判定的集合却不一定是可判定的集合。

> 停机问题是半可判定的，不是可判定的

证明很简单，半可判定性存在的那个程序构造如下：

```python
def p(prog, x):
    exec(prog, x)
    return True
```

如果 `prog(x)` 停机，那么解释器 `exec(prog, x)` 也停机。

半可判定性有几个等价定义：

1. 递归可枚举（recursively enumerable, r.e.）。一个集合 $S$ 是递归可枚举的，当且仅当 $S = \emptyset$ 或存在一个递归全函数 $f$，使得 $S = \\{ y\ |\ \exists x. f(x) = y \\}$，也就是说，$S$ 是 $f$ 的值域.
2. $S$ 的部分[特征函数](https://zh.wikipedia.org/zh-hans/%E6%8C%87%E7%A4%BA%E5%87%BD%E6%95%B0)是部分递归的.
3. $S$ 是某个部分递归函数的值域.

什么是“部分函数”（partial function）呢？简单来说，就是在某些输入上没定义的函数。一个在某些输入不停机的程序，其“像”，也就是 $\\{ (x, y)\ |\ x \in A, \texttt{p}(x) = y \\}$ 就是一个部分函数。

这些等价性需要证明，可以参考 [1, 5].

`is_valid_stmt` 和 `is_valid_proof` 的构造，要求存在一个算法判断一个字符串在不在公理集中。这对公理集提出了要求。事实上，Gödel 不完备性定理的条件中，就有公理集合需要是递归集的要求，递归集与可判定是等价的，它在半可判定的要求上加入了对任何输入都要停机的限制。

## 对 Gödel 不完备定理的证明

我们已经知道了，至少对于 Peano 算术 $\mathsf{PA}$ 来说，$\mathsf{PA} \vdash \psi$ 的问题是半可判定的。根据这个结果和一点小小的编程技巧，我们立刻可以得到 Gödel 不完备性定理。

首先假设 $\mathsf{PA}$ 是完备的，也就是说对任意的 $\psi$，$\mathsf{PA} \vdash \psi$ 或 $\mathsf{PA} \vdash \neg \psi$. 记 $\mathsf{PA}$ 的半可判器为 $\texttt{p}$，可以构造一个 $\mathsf{PA}$ 的判定器 $\texttt{p\\_total}$. 构造如下：

```python
from multiprocessing.pool import ThreadPool

def can_prove_input(ψ):
    can_prove(ψ)
    return ψ

def p_total(ψ):
    pool = ThreadPool(processes=2)
    result = pool.apply_async(can_prove_input, (ψ, ¬ψ))
    which = result.get()
    if which == ψ:
        return True
    else:
        return False
```

没想到吧，这个程序的关键居然在于并发！并发的另一个名字叫做“非确定性计算”，其实，我这里只是想用这种方法模拟非确定性计算而已。（也就是“同时”计算 `can_prove_input(ψ)` 和 `can_prove_input(¬ψ)`）

仔细分析一下 $\texttt{p\\_total}$，考虑完备性，分情况讨论：

- 如果 $\mathsf{PA} \vdash \psi$，那么 `can_prove_input(ψ)` 就会停机，`result.get()` 返回 `ψ`，所以整个函数返回 `True`. 注意，这时 `can_prove_input(¬ψ)` 不会停机，所以 `result.get()` 一定会返回 `ψ`.
- 如果 $\mathsf{PA} \vdash \neg\psi$，那么 `can_prove_input(¬ψ)` 就会停机，`result.get()` 返回 `¬ψ`，整个函数返回 `False`. 根据 $\mathsf{PA}$ 的一致性，这时 $\mathsf{PA} \not\vdash \psi$.

所以，$\texttt{p\\_total}$ 是一个 $\mathsf{PA} \vdash \psi$ 的判定器。这样一来 $\mathsf{PA}$ 就是可判定的，这与不可判定的结果是矛盾的。所以，假设不成立，$\mathsf{PA}$ 不是完备的。

读者可能不太服气：这不是耍赖吗？ Gödel 哪里知道什么多线程？事实上，非确定性图灵机和确定性图灵机是等价的，多线程也可以用单线程模拟。不过，我们仍然可以给出一个比较“正常”的 $\texttt{p\\_total}$. 

```python
def can_prove_n(ψ, n):
    n0 = 0
    while n0 <= n:
        stmt_l = decode_proof(n0)
        if is_valid_proof(stmt_l):
            stmt_last = stmt_l[-1]
            if stmt_last == ψ:
                return True
        n += 1
    return False

def p_total(ψ):
    n = 0
    while True:
        if can_prove_n(ψ, n):
            return True
        elif can_prove_n(¬ψ, n):
            return False
        
        n += 1
```

这里，我们把原来的 `can_prove(ψ)` 改成了 `can_prove_n(ψ, n)`，也就是在 $n$ 步之内是否可以证明 $\psi$. 这样一来，`can_prove_n(ψ, n)` 就一定停机，所以可以在 `p_total` 里直接调用 `can_prove_n` 做顺序计算.

## 不可判定性证明

从上面的讨论中，我们可以看到：

- 虽然半可判定性要写一个比较繁杂的程序，但总体是直觉的
- 有了半可判定性和不可判定性，可以直接得到 Gödel 不完备性定理

问题的关键就成了如何得到 $\mathsf{PA}$ 的不可判定性。本文并不会严格地给出证明，而是简单介绍一下证明的思路。

### 利用 Church 的结果

要证明不可判定性，我们必须引入一个比较复杂的概念--可表示性（representability）。Gödel 本人似乎没有直接使用这个概念 [4]，而是用了它的一个等价形式。

Church 在 1936 年的论文 [11] 里首次证明了存在一个不可判定的问题（虽然有些学者不太满意他的证明），这个问题是：

> 给定 $a, b \in \Lambda^{\circ}$，$a \rightsquigarrow_{\beta} b$ 是否成立？

其中，$\Lambda^{\circ}$ 指的是“封闭的 lambda 项”，$\rightsquigarrow_{\beta}$ 指的是 β-规约（关系）的传递闭包，也就是任意次 β-规约。他的文章进一步指出，任何 ω-一致性的系统都无法构造判定器。[11, 12]

如果要沿用 Church 的证明，那么，我们首先需要把 $\rightsquigarrow_{\beta}$ 算术化。这也就是说，要找到一个整数上的关系 $P$，使得 $P(\sharp M, \sharp N)$ 当且仅当 $M \rightsquigarrow_{\beta} N$，其中 $\sharp$ 表示编码。我们显然需要 $P$ 有一些良好的表示，因为我们下面要找到一个 $\mathsf{PA}$ 中的句子 $\mathsf{P(x, y)}$ 使得对任意的 $n_1, n_2$

$$
\begin{aligned}
P(n_1, n_2) &\rightarrow \mathsf{PA \vdash P(n_1, n_2)} \\\\
\neg P(n_1, n_2)& \rightarrow \mathsf{PA \vdash \neg P(n_1, n_2)}
\end{aligned}
$$

$\mathsf{P(n_1, n_2)}$ 里的 $\mathsf{n_1}$，指的是元语言里的自然数 $n_1$ 在 $\mathsf{PA}$ 里对应的符号串，也就是

$$
\underbrace{S S S \cdots S}_{n_1 个}\ 0
$$

$\mathsf{P(x, y)}$ 是一个**确定的**、含有两个自由变量的 $\mathsf{PA}$ 句子。这样一来：

- 假设 $\mathsf{PA}$ 是可判定的，对于任何的 $a, b \in \Lambda^{\circ}$，
- 在判定器 $\texttt{p}$ 中输入 $\varphi = \mathsf{P(\sharp a, \sharp b)}$，$\psi = \mathsf{\neg P(\sharp a, \sharp b)} $
- 返回 $\texttt{p}(\varphi)$

显然，这个程序判定了 Church 证明为不可判定的问题。实际上，如果 $P$ 在 $T$ 中可表示，那么 $P$ 的判定问题就可以规约到 $T$ 的可判定性。

然而，想找到 $\mathsf{P}$，我个人觉得是很麻烦的工作。

### 利用 Kleene 的结果

Gödel 在 1931 年的论文中，证明了原始递归函数都是可表示的。关于函数的可表示性，教材 [1] 和材料 [9] 的说法不太一样，材料 [9] 把函数的可表示性和关系的可表示性等同了。我们暂时后者的说法[^7]，一个函数 $f$ 是可表示的，当且仅当存在一个 $\mathsf{P}$，使得

$$
f(\vec{m}) = n \rightarrow T \vdash \mathsf{P(\vec{m}, n)} \\\\
f(\vec{m}) \neq n \rightarrow T \vdash \mathsf{\neg P(\vec{m}, n)}
$$

Kleene 证明了范式定理：

> 存在一个三元原始递归函数 $C(x, y, z)$ 和一个一元原始递归函数 $U(x)$，使得任意的部分递归函数 $f$ 在 $e$ 值给定时可以表示为：
> $$
> f(n) = U(\mu z[C(e, n, z)])
> $$

这个定理看起来挺费解的，其实，它描述的是一个类似于解释器的系统：

```python
def U(x):
    # some calculation 
    pass

def C(e, y, z):
    # some calculation
    pass

def μ(f):
    n = 0
    while f(n) != 0:
        n += 1
    return n

def f(n):
    return n * 2

def f_another(n):
    # note: e may not be 10, we just use a fixed value here
    return U(μ(lambda z: C(10, n, n)))

# ∀ n, f(n) == f_anthor(n)
```

这里的 $e$ 大概可以理解为程序（递归函数）被编码之后的结果，通过这个定理，Kleene 用对角线方法证明了下面的定理：

> 不是每个部分递归函数都是潜在递归（potentially recursive）的

所谓的“潜在递归”，就是说，对于部分递归函数 $g$，存在一个全函数 $f$，使得在 $g$ 有定义的地方， $f(n) = g(n)$. 直觉上来看，这个定理类似于“不是所有的半可判定集都是可判定的”。

现在我们假设存在一个 $T$ 的判定器 $\texttt{p\\_t}$，根据 Gödel 的证明，存在一个公式 $\mathsf{C(e, n, z, t)}$ 表示了 $C(e, n, z)$. 那么，可以构造一个新的超级解释器，它能给所有的部分递归函数 $g$ 都找到对应的全函数 $f$. 考虑任意的 $g$，其 $e$ 值为 $e_1$，构造 $f$ 如下：

```python
def f(n):
    if p_t(∀ z. ¬C(e₁, n, z, 0)):
        # g(n) can never halt
        # return a random value, doesn't matter
        return 0
    else:
        # g(n) will halt
        # return g(n)
        return U(μ(lambda z: C(e₁, n, z)))
```

Kleene 的定理保证了，$g$ 唯一不停机的方式就是 $\forall z. C(e_1, n, z) \neq 0$. 所以，因为 $C$ 本身是可以在 $T$ 里表示的原始递归函数，一旦 $T$ 可判定，对于任意 $n$，$g(n)$ 是否停机就是可判定的。

## 与 Gödel 原始证明的比较

Gödel 的原始证明 [1, 13] 相当复杂，大概可以概括为：

1. 证明所有的原始递归函数是可表示的。
2. 将一阶逻辑和初等算术的语法用原始递归函数算术化，类似于用原始递归函数写一个我们这里写的 `can_prove_n(ψ, n)` 程序。Gödel 写了 45 个原始递归函数。
3. 由于 `can_prove_n(ψ, n)` 是原始递归函数，所以是可表示的。这样一来，能找到一个公式 $\mathsf{P(\sharp\psi, n)}$ 表示它。
4. Gödel 证明了不动点定理：对任意的一元公式 $\psi(x)$，都能找到一个语句 $\sigma$，使得 $T \vdash \sigma \leftrightarrow \psi(\mathsf{\sharp\sigma})$.
5. 所以，存在 $\sigma$ 使得 $T \vdash \sigma \leftrightarrow \neg \exists n. \mathsf{P}(\sharp\sigma, n)$，注意，这个句子构造了类似于说谎者悖论的命题。
6. 利用 ω-完备性，无论假设 $T \vdash \sigma$ 还是 $T \vdash \neg \sigma$，都会得到矛盾。这样一来，$\sigma$ 和 $T$ 就是独立的。

可见，我们的证明没有避免 1. 和 2.，而这是最复杂的步骤。不过，半可判定性直觉理解起来比较简单，如果把它当前提，那么证明会非常简短。1. 和 2. 其实都是为了证明半可判定性引入的。

## 延伸阅读

- 如果你想学习 Gödel 的原始证明，那么请直接阅读 [1] 或 Gödel 的原始论文 [13]. 杨睿之老师的书写得非常好。
- 这种证明的思路最早似乎来源于 Kleene，可以参考他的著作 [10] 和这份笔记 [9].
- 这篇文章 [4] 也用这种思路介绍不完备性定理
- ocau 老师的 Agda 形式化值得参考，不过他讲课的视频没有公开（原因不明）。他的思路可能来源于这份 Coq 形式化的工作 [6]，可以直接阅读这篇论文作为替代。
- 一个 Gödel 不完备性定理的完全 Coq 形式化 [3]（上面的论文是综合式的，和这里讨论的有比较大的区别）
- 有两本关于 Gödel 不完备性定理的书 [2, 8] 写得都还不错，可以选部分章节阅读。
- 关于递归函数相关的知识，可以参考 [1, 10] 和宋老师的 [7]. 类似地，可计算性相关的概念可以参考 [5]

## 参考文献

- [1] [数理逻辑：证明及其限度](https://book.douban.com/subject/35210053/)
- [2] [An Introduction to Gödel's Theorems](https://www.karlin.mff.cuni.cz/~krajicek/smith.pdf)
- [3] [Essential Incompleteness of Arithmetic Verified by Coq](http://r6.ca/Goedel/goedel1.html)
- [4] [Gödel's Incompleteness Phenomenon—Computationally](https://www.cairn.info/revue-philosophia-scientiae-2014-3-page-23.htm)
- [5] [Computability and Complexity: From a Programming Perspective.](http://hjemmesider.diku.dk/~neil/comp2book2007/book-whole.pdf)
- [6] [Gödel’s Theorem Without Tears: Essential Incompleteness in Synthetic Computability](https://drops.dagstuhl.de/opus/volltexte/2023/17491/pdf/LIPIcs-CSL-2023-30.pdf)
- [7] [计算模型导引](https://book.douban.com/subject/10785250/)
- [8] [Incompleteness and Computability: An Open Introduction to Gödel's Theorems](https://ic.openlogicproject.org/ic-screen.pdf)
- [9] [Kleene’s proof of Gödel's Theorem](https://www.logicmatters.net/resources/pdfs/KleeneProof.pdf)
- [10] [Introduction to Metamathematics](https://archive.org/details/BubliothecaMathematicaStephenColeKleeneIntroductionToMetamathematicsWoltersNoordhoffPublishing1971)
- [11] [An Unsolvable Problem of Elementary Number Theory](https://www.ics.uci.edu/~lopes/teaching/inf212W12/readings/church.pdf)
- [12] [A. Church on Computability](https://plato.stanford.edu/entries/church/supplementA.html)
- [13] [On Formally Undecidable Propositions of Principia Mathematica and Related Systems I](https://monoskop.org/images/9/93/Kurt_G%C3%B6del_On_Formally_Undecidable_Propositions_of_Principia_Mathematica_and_Related_Systems_1992.pdf)

[^1]: 这里的命题是“元语言”里的命题，不是我们讨论的语言，也就是“对象语言”，此处为一阶逻辑的命题。不要混淆。
[^2]: 当然，在 Hilbert System 中，还有“逻辑公理”，也就是任何一阶理论都成立的公理。这些公理同样可以作为推理的起点，推出任何逻辑都成立的“永真式”（tautology）。
[^3]: 原始的叙述比较复杂，这里的“初等算术”比 Peano 弱一些，这涉及到 essential incompleteness 的概念。
[^4]: 换句话说，就是能找到一个新命题，这个新命题与加入公理后的系统独立。
[^5]: 这里所称的“集合”，一般是程序数据集合的某个子集，递归论中一般认为是自然数的子集
[^7]: [1] 的说法似乎是正确的，不过它给出的是更强的命题，我们这里只需要比较弱的这种形式就够了