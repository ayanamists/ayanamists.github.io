---
title: "不为人知的故事 -- Church 和他的系统"
date: 2024-03-15T22:29:00+08:00
draft: false
categories:
  - 计算机历史
---

作为函数式编程爱好者，你一定知道 Church 和他的 $\lambda$ 演算。如果你学过递归论，也许还会知道 $\lambda$ 可定义性、$\lambda$ 演算的各种性质（Church-Rosser 定理、不动点定理……），以及他们结合起来是如何给出 $\beta$ 等价的不可判定证明的。

当然，你也许都不知道这些词语是在说什么，这也没关系。如果你学过一点点的计算机科学，那一定知道图灵机（Turing Machine）。有人把图灵和他的论文看作是计算机科学的起源，但实际上，Church 和他的 $\lambda$ 演算比图灵机更早，也更优雅[^2]。我们今天要讲的故事，实际上正是理论计算机科学的起源之一。

很少有人知道，Church 当年为什么要发明 $\lambda$ 演算，以及他当年发明的系统和我们现在所知的 $\lambda$ 演算有什么不同。事实上，Church 的系统要比今天的 $\lambda$ 演算复杂许多。为什么 Church 的系统被人遗忘了，而今天的 $\lambda$ 演算却成为了计算机科学的基石呢？

这是一个非常有趣的故事。我在 2023 年的 SICP 课[^3]上和同学们分享过这个故事，现在，我觉得还是应该写下来，让更多的人知道这段历史。

{{<  admonition "note" >}}
本文有一些技术细节，但重要的还是故事，如果看不懂可以跳过，不影响对于故事的理解。不过，我建议至少看明白第一节的内容，这样你就能知道 Church 大概在做什么。
{{< /admonition >}}


## 形式系统


Church 于 1932 年发表的系统[^1]，现在被称作一个 “形式系统”（formal system）。简单来说，形式系统有三个要素：

- 语法（syntax）- 一组符号和规则，用来描述合法的表达式（formula）。这里的 “语法” 和编程语言的 “语法” 没有区别，它们都是规定了这个系统中的表达式长什么样。
- 一组公理（axioms）- 一组被认为是真的表达式。这些公理是系统的基础，其他的定理都是从这些公理推导出来的。
- 推导规则（inference rules）- 一组规则，用来从已知的公理和定理推导出新的定理。

我可以给出一个极其简单的形式系统（记作 $P$）：

- 语法 - $a$ 和 $b$ 构成的字符串。如 $a$, $aa$, $ab$, $ba$ 等等
- 公理 - $a$, $b$
- 推导规则
  - (I.) 如果 $x$ 是一个定理，那么 $axa$ 和 $bxb$ 也是定理。

这个系统的所有定理就是所谓的 “回文串”（palindrome）。比如 $aba$ 是一个定理，它的推导过程（我们把它叫做“证明”，proof）是：

- $b$   (公理)
- $aba$ (I.)

在逻辑学里，“$aba$ 是系统 $P$ 的定理” 可以写作 $P \vdash aba$。这里的 $\vdash$ 是 “推导” 的意思。

相对地，$ab$ 不是一个定理，因为我们无法通过推导规则得到它。这可以写作 $P \nvdash ab$。

这里说的 “公理”、“定理”、“证明” 都是形式系统的术语。它们有精确的数学含义，比如在这里，“公理” 就是 $a$ 和 $b$，而 “定理” 就是回文串，“证明” 就是一个字符串的序列，满足推导规则。不过，显然地，我们从小到大都在接受数学教育，“公理”、“定理”、“证明” 已经有了自然语言的含义。请允许我稍微花一些时间，来解释一下它们之间的关系。

自然语言中的 “公理”，我们下面用 _公理_ 来表示，而形式系统中的 “公理”，我们用 `公理` 来表示。其他概念，我们都作类似处理。

我们从小到大，学过各种 _定理_，比如 $1 + 1 = 2$. 这些 _定理_ 大都是现代逻辑建立之前就被 _证明_ 的。自然，在逻辑学之前，就有了 _证明_、_定理_ 这些概念。这些通过自然语言描述和构建的数学，我们大部分时候还是认为它就是 “存在” 而正确的。

而形式系统中的 `定理`，我们已经说过，是严格的数学对象。我们知道它的性质，我们知道它是怎么通过公理推导出来的。这个推导过程，是能够用计算机来验证的。总之，这些数学对象是清晰的、严格的。而自然语言的数学，至少没有这样的严格性。比如，随便选取一本数学教材，你能够想象存在一个程序能够轻易地验证其中的所有证明吗？

所以在我看来，形式系统的首要目标，就是用严格的数学对象把自然语言的数学 “固化住”。“固化住”，用一个更学术化的词语来表述，就叫做 “形式化”。

自然语言描述的数学和形式系统描述的数学，就像是算法和程序。算法是非形式的程序，程序是形式化的算法。

用形式系统来形式化自然语言的数学证明，是一个非常困难的问题。为了解决这个问题，我们需要更复杂、更麻烦的形式系统。那么，应该选用什么样的形式系统呢？

对于这个难题，大部分数学家都觉得 19 世纪后半叶发展的另一门学科--集合论--是答案。大部分的数学对象都可以用集合论来描述，而集合论可以用形式系统来描述。Russell 在 1910-1913 年间发表的 _Principia Mathematica_、以及 Zermelo-Fraenkel 集合论（1908, 1922），都是这种方案的实践。ZF 集合论基本上已经成为了现代数学的基础。

## Church 的系统

![Church](/imgs/alonzo-church.png)

Church 在 1920 年代，尝试用一种非集合论的方法来给出一个形式系统。Church 在 1932 年的论文中说，他觉得无论是 Russell 的 _Principia Mathematica_ 还是 ZF 集合论，都采取了 “不自然” 的方法来处理著名的 “罗素悖论”（Russell's Paradox）。

他给出的方法是，“限制排中律的使用”。在技术上来说，我个人反而觉得他的这个方法更加 “不自然”，以至于我很难在本文的篇幅内给出一个清晰的解释。总之，他设计了一套形式系统，这个系统有 37 条（是的，你没看错，就是 37 条）公理和 5 条推导规则。这 5 条推导规则导向了现代的 $\lambda$ 演算。这个系统在 1932 年发表的 _A Set of Postulates for the Foundation of Logic_ 中给出。

Church 的系统，语法中确实没有集合或者类，由这些符号构成：

> 5. Undefined terms. We are now ready to set down a list of the undefined terms of our formal logic. They are as follows:
$$
\\{\\}(),\\ \lambda\[\],\\ \Pi,\ \Sigma,\ \\&, \\ \sim, \\ \iota, \\ A
$$

其中，$\\{\\}()$ 代表的是现代 $\lambda$ 演算中的 “作用”（apply, 俗称函数调用），$\lambda [\ ]$ 自然是 $\lambda$ 抽象，$\Pi$ 是一个比较复杂的语法，$\Pi(P, Q)$ 大概是 $\forall x. (P(x) \to Q(x))$[^5]，$\Sigma$ 是存在，$\\&$ 是逻辑与，$\sim$ 是逻辑非，$\iota$ 类似于递归函数里的 $\mu$ 算子，$\iota(F)$ 指的是 “让 $\\{F\\}(x)$” 成立的 $x$”. 至于 $A$，这个符号特别抽象，这里不讨论了。

之后，Church 定义了一些语法糖，或者说，“宏” (macro)。比如，他定义了 $V$，也就是现代逻辑学里的 $\lor$，定义是这样的：

$$
V(P, Q) \equiv \sim . \sim P . \sim Q
$$

用今天的符号来表达就是：

$$
P \lor Q \equiv \neg (\neg P \land \neg Q)
$$

值得一提的是，Church 沿用的是当时流行的、Russell 采用的 “点记法”（dot notation）来表示逻辑运算。简单来说，$.$ 决定了 “优先级”，这样可以少写括号。比如，$\sim . \sim P . \sim Q$ 是 $\sim (\sim P \land \sim Q)$ 的简写。不过，据我所知，Principia Mathematica 里的点符号似乎不允许这样使用，这个命题在 PM 里应该是 $\sim (\sim P . \sim Q)$.（当然，这里也可能是我的误解，我参考的是 [The Notation in Principia Mathematica](https://plato.stanford.edu/entries/pm-notation/#Exam)）. 

另外，Church 也把蕴含符号 $\supset$ 定义成了一个宏。

总之，Church 的系统语法大概就是这样。他的 37 条公理，则更为复杂。像 15. 条这样的可能还比较好理解：

$$
\text{15.}\ pq \supset_{pq} q 
$$

这个指的是 $p \land q \to q$. 但大部分公理都是很难理解的。比如，第 1 条：

$$
\text{1.}\ \Sigma(\varphi) \supset_{\varphi} \Pi(\varphi, \varphi)
$$

$\Pi(\varphi, \varphi)$ 应该是不需要前提就成立的（按照 $\forall x. (\varphi(x) \to \varphi(x))$ 理解），但是由于 Church 的限制，这里必须得保证 $\Sigma(\varphi)$ 才能成立。

相比于 37 条不知所云的公理，在今天的我们看来，他的 5 条推导规则就显得非常简单了。这 5 条规则和现代的 $\lambda$ 演算比较相似。这 5 条规则（含有我的一些重新表述，但和 Church 的原始论述没有区别）是：

**I.** 在一个真公式 $J$ 中，如果 

- $x$ 在 $J$ 里的所有出现都是绑定出现
- $y$ 不在 $J$ 中自由出现，

那么，在 $J$ 中，用 $y$ 替换 $x$ （记作 $\text{S}_{y}^{x}\ J$）的结果 $K$ 也是真公式。

**II.** 如果 

- $(\lambda x. M) N$ 是真公式 $J$ 的一个子公式
- $x$ 不在 $M$ 中绑定出现
- $x$ 在 $M$ 中自由出现 (*)
- $x$ 不在 $N$ 中自由出现

那么，在 $J$ 中，用 $\text{S}_{N}^{x}\ M$ 替换一个特定的 $(\lambda x. M) N$ 的结果 $K$ 也是真公式。

**III.** 如果

- $\text{S}_{N}^{x}\ M$ 是真公式 $J$ 的一个子公式
- $x$ 不在 $M$ 中绑定出现
- $x$ 在 $M$ 中自由出现 (*)
- $x$ 不在 $N$ 中自由出现

那么，在 $J$ 中，用 $(\lambda x. M) N$ 替换一个特定的 $\text{S}_{N}^{x}\ M$ 的结果 $K$ 也是真公式。

**IV.** 如果 $F(A)$，那么 $\Sigma(F)$.

**V.** 如果 $\Pi(F, G)$，且 $F(A)$，那么 $G(A)$.

这 5 条规则，**IV.** 和 **V.** 类似于一阶逻辑的 “存在量词引入” 和 modus ponens. 重头戏是 **I.**, **II.** 和 **III.**:

- **I.** 是 $\alpha$ 等价
- **II.** 是 $\beta$ 等价的一个方向，可以看作是 $\beta$ 规约
- **III.** 是 $\beta$ 等价的另一个方向，可以看作是 $\beta$ 抽象

这三条规则，构成了现代 $\lambda$ 演算的核心。值得一提的是打 (*) 的规则，它并不在现代的 $\lambda$ 演算中。这件事我们按下不表，如果你感兴趣，可以自行思考一下。

## Kleene 和 $\lambda$ 可定义性

不知道上面这些乱七八糟的符号，你看懂了多少。如果你给几年前的我看，我肯定是一头雾水。更何况，Church 的系统有 37 条公理，在这个系统里做证明，想想就头大。相比于 ZF 集合论，Church 的系统显然是过分复杂了。更何况，他这种直接跳过一阶逻辑的做法，又增加了一些学习的难度。

幸运的是，Church 有一位学生，名叫 Kleene.

![Kleene](/imgs/Kleene_3.jpeg)

Kleene 在 1931 年，22 岁本科毕业的时候，来到 Princeton 大学读 Church 的博士生。惊人的是，他在 1934 年就得到了博士学位，时年 25 岁。不得不感叹一句，他真是名副其实的 “天才、超天才”[^6] 啊！

在 1931 年的秋季学期，Church 给 Kleene 和其他学生讲了他的新发明--也就是我们上面说的系统。从 Kleene 的回忆中，Church 当时就有了 $\beta$ 规约的想法。什么是 $\beta$ 规约呢？简单来说，就是把 $(\lambda x. M) N$ 视为一个函数调用，把它替换成 $\text{S}_{N}^{x} M$。换句话说，就是对他的系统里的一个公式，运用公理 **II.** 进行推导。

重要的是，这种推导可以看成一种 “计算”。如果给定一个 $\lambda$ 项 $a$，进行任意多次 $\beta$ 规约，得到 $b$，这可以看成 $\lambda$ 演算的 “计算”。实际上，这种 “计算” 也定义了一个关系，$a \rightarrow_{\beta} b$，意思是 $a$ 可以通过 $\beta$ 规约得到 $b$。

Church 在当时的讲座里还提到，他已经发现了在他的系统里定义正整数的方法，也就是今天的说的 Church 数。$1, 2, 3$ 定义为：

$$
\mathsf{1} \equiv (\lambda f. \lambda x. f x) \\\\
\mathsf{2} \equiv (\lambda f. \lambda x. f (f x)) \\\\
\mathsf{3} \equiv(\lambda f. \lambda x. f (f (f x)))
$$

结合上面 $a \to_{\beta} b$ 的想法，Church 发现，一些函数可以用 $\lambda$ 项来表示，比如后继函数 $S(n) = n + 1$，可以用 $\lambda$ 项表示为：

$$
\mathsf{S} \equiv \lambda n. \lambda f. \lambda x. f (n f x)
$$

它具有性质 $\mathsf{S}\ \mathsf{1} \to_{\beta} \mathsf{2}$，更一般地，Church 提出了 $\lambda$ 可定义性（$\lambda$ definability）的概念。他的想法是，如果一个函数 $f$ 可以用 $\lambda$ 项表示，那么 $f$ 就是 $\lambda$ 可定义的。

形式化地说，对于一个（偏）函数 $f$，如果存在一个 $\lambda$ 项 $\mathsf{F}$，对于在 $f$ 定义域里的自然数 $n$，有

$$
\mathsf{F}\ \mathsf{n} \to_{\beta} \mathsf{f(n)}
$$

那么 $f$ 就是 $\lambda$ 可定义的。上面的 $\mathsf{f(n)}$ 指的是 $f(n)$ （一个自然数）对应的 Church 数。

Church 发现，不仅是后继函数，加法函数 $f(x, y) = x + y$ 也是 $\lambda$ 可定义的。但是，Church 不知道 “前驱” 函数（$P(n) = n - 1$）是不是 $\lambda$ 可定义的。当然，他的系统允许他 “作弊”，不用 $\lambda$ 定义性，也能得到前驱函数。他的定义是这样的：

$$
\lambda r. \iota x. (\mathsf{S}\ x = r)
$$

这个 $\iota$ 的定义见上一节，简单来说就是，Church 通过 “满足 $\mathsf{S}\ x = r$ 的 $x$” 来定义 $r$ 的前驱。

1932 年初，Church 让 Kleene 想办法把自然数的公理（Peano Axioms）在他的系统里证明出来。Kleene 发现这里必须得有一个前驱函数，他对 Church 的 “作弊” 不是特别满意，于是他开始研究前驱函数是不是 $\lambda$ 可定义的。

在 Kleene 的回忆中（见 [Kleene 1981]），他一开始想要改变一下自然数的定义，但是 Church 还是想用他的 Church 数。于是，他绞尽脑汁去思考如何给出一个前驱函数。终于，在 1932 年 1 月末或 2 月初，Kleene 在看牙科诊所的时候，突然想到了一个办法来解决 $\lambda$ 定义前驱函数的问题。进而，Kleene 给出了一个 $\lambda$ 项，$\lambda$ 定义了前驱函数。具体的定义我就不展示了，有兴趣的同学可以先自己思考一下。

当 Kleene 把他的发明告诉 Church 的时候，Church 说，他几乎要放弃了：

> When I brought this result to Church, he told me that he had just about convinced himself that there is no $\lambda$-definition of the predecessor function.

一个原本被认为不能被 $\lambda$ 定义的函数，被 Kleene 证明是可以定义的。Kleene 和 Church 开始思考起一个新的问题：

> 到底什么样的函数是 $\lambda$ 可定义的？

对于这个问题，90 年后的今天，我们脱口而出：**一个函数 $f$ 是 $\lambda$ 可定义的，当且仅当 $f$ 是可计算的**。

当时，Kleene 的主业还是在 Church 的系统里证明一些数学命题。副业就是研究 $\lambda$ 可定义性。他发现几乎所有直觉上可以计算的自然数函数，都是 $\lambda$ 可定义的。在 1933 年，Gödel 来到了 Princeton. 1934 年，他报告了一种定义递归函数的方法，这就是现在所谓 Herbrand-Gödel 可计算函数。Kleene 看到了这个报告，他意识到，这个方法和他的 $\lambda$ 可定义性有关。

最终，Kleene 和 Church 在 1935-36 年发表的论文中，证明了一系列问题：

- $\approx_{\beta}$ ，也就是 $\beta$ 等价，是 $\lambda$ 不可定义的
- $\lambda$ 可定义的函数和 Gödel 可计算函数是等价的
  
后面的故事我们都无比熟悉了。Turing 在 1936 年发表了他的论文，提出了 Turing Machine，证明了 Turing Machine 和 Church 的 $\lambda$ 演算是等价的。理论计算机科学就这样诞生了。

可惜的是，当时很多人觉得 $\lambda$ 演算不太 “数学”，所以 Kleene 之后的那些伟大工作，都基于他定义的 $\mu$-递归函数；而说到 “计算” 模型，大家也更喜欢图灵机。

## Church 的系统是怎么被遗忘的

$\lambda$ 可定义性的研究开启了现代计算机科学的大门，但是 Church 的系统呢？

Kleene 在回忆中，有些俏皮地说：

> This paper contains Gödel’s celebrated proof of the existence of undecidable sentences in formal systems embodying the usual elementary number theory, and his “second theorem” on the impossibility of a proof of the consistency of such a system within the system itself. Church’s immediate reaction was that his formal system, about which I am going to say more, is sufficiently different from the systems Gödel dealt with that Gödel’s second theorem might not apply to it (see Church 1933 top p. 843). Indeed, Church was right!

Church 似乎还觉得他的系统能够不受 Gödel 的限制（我们现在知道，这是不可能的）。而 Kleene 居然也赞成？这是怎么回事呢？

哦，原来 Kleene 的下句就解释了原因：

> In his system there is a proof of its own consistency, since in fact it is inconsistent (so all its sentences are provable), as Church had thought possible (1933 top p. 842) and as Rosser and I showed later (Kleene and Rosser 1935).

Kleene 啊，有你这么拿老师的系统开玩笑的吗。这多少有点 “地狱笑话” 的味道。没错，Church 的系统确实超越了 Gödel 的限制，因为它根本就是不一致的！

所谓 “不一致”，就是说，这个系统 $T$，可以证明某个命题 $P$，同时也可以证明 $\neg P$. 这在直觉上已经很糟糕：这个系统有某种 “矛盾性”。在技术上则更糟糕，因为这意味着这个系统可以证明任意命题。假设我们有 $T \vdash P$ 和 $T \vdash \neg P$，要证明一个任意的命题 $Q$，可以作如下推导：

1. $P$
2. $P \lor Q$
3. $\neg P$
4. $Q$

这个推导用到的推理规则是 “或” 的引入和消去，几乎任何形式系统都有这个规则。所以，如果一个系统是不一致的，那么它他就能证明任意命题。一般来说，我们认为这样的系统是没有意义的。

Kleene 和 Church 的另一位学生，Rosser，在 1935 年证明了 Church 的系统是不一致的。他们的证明被叫做 “Kleene-Rosser 悖论”。这个悖论在技术上比较复杂，不过直觉上看，Church 的系统已经能编码任意的可计算函数了，这确实强烈地暗示了不一致性。

在 1984 年，Rosser 写的一篇文章里，还介绍了怎么用 Curry 发明的技术来证明 Church 的系统的不一致性，这个证明很简单。

实际上，Church 在 1933 年发表的同名（和 1932 年同名）论文中，就发现了他的系统是不一致的。为了解决这种不一致性，他删除了公理 19，然后又加入了两条公理，这样公理就来到了 38 条。他觉得这个新系统是一致的，然而我们现在知道，这个系统也是不一致的。

所以，简单来说，由于 Church 的系统是不一致的，而且极其复杂，所以它被人遗忘了。

## 结语

Church 一开始的目的是，用他的系统来形式化数学，但这个系统现在早已无人问津；而他系统的一个副产品，$\lambda$ 演算，却成为了计算机科学的基石。这些伟大的名字，Church、Kleene、Turing、Gödel、...，永远地留在了计算机科学的历史上。

他们的工作，总称应该还是 “递归论”。特别是 Kleene 的很多研究，是在研究不可计算集与它们之间的关系。不过，我还是觉得，如果一门叫做 “计算理论” 的学科，不去讲 $\lambda$ 演算、递归函数，而去讲什么自动机，那也确实缺失了很多有趣的东西。如果你的 “计算理论” 也是一堆自动机的话，希望这篇文章能让你对 $\lambda$ 演算有了一些兴趣。

## 小测试

1. Church 数最早发表在 Church 1933 年的论文里，这篇文章还给出了 Peano 公理在他的系统里的表述。然而，无论是他表述的 Church 数还是 Peano 公理，都是从 $1$ 开始的，而不是从 $0$ 开始的。你能想一想，为什么 Church 不把 $0$ 放到他的系统里吗？这不只是他的 taste，更多地是一个技术问题。
2. 现代 $\lambda$ 演算里，我们一般把（二）元组的构造子（constructor）定义为
   $$(a, b) \equiv \lambda a \lambda b \lambda f. f a b$$
   fst 和 snd 定义为
   $$
   \begin{aligned}
   \text{fst} &\equiv \lambda p. p (\lambda a \lambda b. a) \\\\ 
   \text{snd} &\equiv \lambda p. p (\lambda a \lambda b. b)
   \end{aligned}
   $$
   元组的定义是 Kleene 定义前驱函数的一个重要步骤，但是，Kleene 可以这么定义元组吗？为什么？如果不能，你能给出 Kleene 能用的元组定义吗？（提示： 答案参见 [Kleene 1981]）
3. 如果说，Church 的原始系统是不一致的，那为什么 $\lambda$ 演算本身不受这个问题影响呢？不会有人觉得 $\lambda$ 演算也会 “不一致” 吗？

## 延伸阅读

1. Kleene, S. C. (1981). Origins of recursive function theory. Annals of the History of Computing, 3(1), 52-67.
2. Church, A. (1933). A set of postulates for the foundation of logic. Annals of Mathematics, 34(4), 839-864.
3. Church A. (1936). An unsolvable problem of elementary number theory. American Journal of Mathematics, 58(2), 345-363.
4. Church A. (1941). The Calculi of Lambda-Conversion. Princeton University Press.
5. Church A. (1932). A set of postulates for the foundation of logic. Annals of Mathematics, 33(2), 346-366.
6. Rosser J. B. (1984). Highlights of the history of the lambda calculus. Annals of the History of Computing, 6(4), 337-349.

## 附录：Church 的 37 条公理

感谢深度学习技术，我能够简单地把 Church 的 37 条公理的 latex 代码提取出来。

1. $\boldsymbol{\Sigma}(\varphi) \supset_{\varphi} \Pi(\varphi, \varphi)$.
2. ' $x . \  \varphi(x) \supset_{\varphi} . \  \Pi(\varphi, \psi) \supset_\psi \psi(x)$.
3. $\mathbf{\Sigma}(\sigma) \supset_\sigma . \ \left[\sigma(x) \supset_x \varphi(x)\right] \supset_p . \  \Pi(\varphi, \psi) \supset_\psi . \  \sigma(x) \supset_x \psi(x)$.
4. $\mathbf{\Sigma}(\varrho) \supset_{\varrho} . \Sigma y\left[\varrho(x) \supset_x \varphi(x, y)\right] \supset_p . \ \left[\varrho(x) \supset_x \Pi(\varphi(x), \psi(x))\right] \supset_\psi$. $\left[\varrho(x) \supset_x \varphi(x, y)\right] \supset_y . \  \varrho(x) \supset_x \psi(x, y)$.
5. $\Sigma(\varsigma) \supset_{\varphi} . \  \Pi(\varphi, \psi) \supset_\psi . \  \varphi(f(x)) \supset_{f x} \psi(f(x))$.
6. ' $x . \  \varphi(x) \supset_p . \  \Pi(\varsigma, \psi(x)) \supset_\psi \psi(x, x)$.
7. $\varphi(x, f(x)) \supset_{q f x} . \  \Pi(\varphi(x), \psi(x)) \supset_\psi \psi(x, f(x))$.
8. $\mathbf{\Sigma}(\varrho) \supset_{\varrho} . \Sigma y\left[\varrho(x) \supset_x \varphi(x, y)\right] \supset_p . \ \left[\varrho(x) \supset_x \Pi(\varphi(x), \psi)\right] \supset_\psi$. $\left[\varrho(x) \supset_x \varphi(x, y)\right] \supset_y \psi(y)$.
9. ' $x . \  \varphi(x) \supset_{\varphi} \Sigma(\varphi)$.
10. $\Sigma x \varphi(f(x)) \supset_{f \varphi} \Sigma(\varphi)$.
11. $\varphi(x, x) \supset_{\varphi x} \Sigma(\varphi(x))$.
12. $\Sigma(\varphi) \supset_{\varphi} \Sigma x \varphi(x)$.
13. $\Sigma(\varphi) \supset_{\varphi} . \ \left[\varphi(x) \supset_x \psi(x)\right] \supset_\psi \boldsymbol{\Pi}(\varphi, \psi)$.
14. $p \supset_p . \  q \supset_q p q$.
15. $p q \supset_{q p} p$.
16. $p q \supset_{p q} q$.
17. $\Sigma x \Sigma \theta[\varphi(x) . \  \sim \theta(x) . \  \Pi(\psi, \theta)] \supset_{p \psi} \sim \Pi(\varphi, \psi)$.
18. $\sim \Pi(\varphi, \psi) \supset_{\varphi \psi} \Sigma x \Sigma \theta . \  \varphi(x) . \  \sim \theta(x) . \  \Pi(\psi, \theta)$.
19. $\Sigma x \Sigma \theta\left[\sim \varphi(u, x) . \sim \theta(u) . \Sigma(\varphi(y)) \supset_y \theta(y)\right] \supset_{\varphi u} \sim \Sigma(\varphi(u))$.
20. $\sim \boldsymbol{\Sigma}(\varphi) \supset_{\varphi} \boldsymbol{\Sigma} x \sim \varphi(x)$.
21. $p \supset_p \sim q \supset_q \sim p q$.
22. $\sim p \supset_p . \  q \supset_q \sim . \  p q$.
23. $\sim p \supset_p \sim q \supset_q \sim p q$.
24. $p \supset_p . \ [\sim . \  p q] \supset_q \sim q$.
25. $[\sim . \varphi(u) . \  \psi(u)] \supset_{\varphi \psi u} . \ \left[\left[[\varphi(x) . \  \sim \psi(x)] \supset_x \varrho(x)\right]\right.$.
$\left.\left[[\sim \varphi(x) . \  \psi(x)] \supset_x \varrho(x)\right] . \ [\sim \varphi(x) . \  \sim \psi(x)] \supset_x \varrho(x)\right] \supset_{\varrho} \varrho(u)$.
26. $p \supset_p \sim \sim p$.
27. $\sim \sim p \supset_p p$.
28. $\sim \Sigma(\varphi) \supset_{\varphi} . \  \Sigma(\psi) \supset_\psi \Pi(\varphi, \psi)$.
29. $\sim \boldsymbol{\Sigma}(\varphi) \supset_{\varphi} \sim \boldsymbol{\Sigma}(\psi) \supset_\psi \boldsymbol{\Pi}(\varphi, \psi)$.
30. ' $x . \  \varphi(x) \supset_{\varphi} . \ \left[\theta(x) . \  \psi(x) \supset_\psi \Pi(\theta, \psi)\right] \supset_\theta \varphi(\iota(\theta))$.
31. $\varphi(\iota(\theta)) \supset_{\theta \varphi} \boldsymbol{\Pi}(\theta, \varphi)$.
32. $E(\iota(\theta)) \supset_\theta \Sigma(\theta)$.
33. $\left[\varphi(x, y) \supset_{x y} . \  \varphi(y, z) \supset_z \varphi(x, z)\right]\left[\varphi(x, y) \supset_{x y} \varphi(y, x)\right] \supset_{\varphi} . \  \varphi(u, v) \supset_{u v}$. $\psi(A(\varphi, u)) \supset_\psi \psi(A(\varphi, v))$.
34. $\left[\varphi(x, y) \supset_{x y} . \  \varphi(y, z) \supset_z \varphi(x, z)\right]\left[\varphi(x, y) \supset_{x y} \varphi(y, x)\right] \supset_{\varphi}$. $\left[\varphi(x, y) \supset_{x y} \theta(x, y)\right] \supset_\theta \sim \theta(u, v) \supset_{u v} \sim . \psi(A(\varphi, u)) \supset_\psi \psi(A(\varphi, v))$.
35. $\left[\psi(A(\varphi, u)) \supset_\psi \psi(A(\varphi, v))\right] \supset_{\varphi u v} \varphi(u, v)$.
36. $E(A(\varphi)) \supset_{\varphi} . \  \varphi(x, y) \supset_{x y} . \  \varphi(y, z) \supset_z \varphi(x, z)$.
37. $E(A(\varphi)) \supset_{\varphi} \varphi(x, y) \supset_{x y} \varphi(y, x)$.


[^1]: 我建议不要把它叫做 $\lambda$ 系统，因为 $\lambda$ 系统现在很多时候指的是构造 $\lambda$ 等价性的形式系统，而不是 Church 的原始系统。
[^2]: 这是我的个人观点。不过，无论是从语法还是语义来看，$\lambda$ 演算都比图灵机要简单得多。
[^3]: 南京大学《计算机程序的构造与解释》课程。
[^5]: 事实上，Church 的论文重点强调了 $\Pi(P, Q)$ 和 $\forall x. P(x) \to Q(x)$ 的区别，这和他 “对于排中律的限制” 有关。不过，这个细节超出了本文的范围。 
[^6]: http://dangshi.people.com.cn/n/2014/0415/c85037-24898922-2.html
