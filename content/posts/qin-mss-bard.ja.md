---
title: "秦九韶アルゴリズムを用いて最大部分配列和問題を解く"
date: 2023-12-23T20:44:33+08:00
draft: false
---

{{<  admonition "warning" >}}
この記事はChatGPTによって中国語から翻訳されたもので、いくつかの誤りが含まれているかもしれません。不正確な部分があればご了承ください。
{{< /admonition >}}

> 二つの先端が異なる長さの田んぼがあり、大きな斜面は39歩、小さな斜面は25歩、中央の幅は30歩です。その積は何ですか？
>
> 方法は次のとおりです。少ない幅で求め、それを反転させて入力します。半幅を自己乗算して半冪を設定し、それを小さい斜面の冪と相殺し、乗算して小さい率を得ます。半冪と大きな斜面の冪を相殺し、乗算して大きな率を得ます。二つの率を相殺し、残りを自己乗算して実数を得ます。二つの率を合わせて倍にし、それを上の列にします。一つを利益の角として、反転法を開き、三乗して積を得ます。

一部の人々は、関数型プログラミングを軽視しています。彼らは、関数型プログラミングが無意味な抽象化を増やすだけで、その代償はパフォーマンスの低下だと感じるかもしれません。このような見解を持つ人々は、おそらくRichard Birdの著書 [_PEARLS OF FUNCTIONAL ALGORITHM DESIGN_](https://www.cambridge.org/core/books/pearls-of-functional-algorithm-design/B0CF0AC5A205AF9491298684113B088F) を読んだことがないでしょう。この本の内容は確かに難解で、私も一度も完全に読み通したことはありません。

幸運なことに、Richard Birdは1989年にわずか5ページの論文 [_Algebraic identities for program calculation_](https://academic.oup.com/comjnl/article/32/2/122/543545?login=false) を書いています。この論文では、最大部分配列和[^1]（Maximum subarray problem）という有名な問題を用いて、関数型プログラミング思考がアルゴリズム問題に対する独特の理解を見事に示しています：最大部分配列和問題は秦九韶アルゴリズムで解くことができます。

## 最大部分配列和問題

最大部分配列和問題の目的は、数列の一次元方向で連続した部分配列を見つけ、その部分配列の和が最大になるようにすることです。例えば、数列 `[-2, 1, -3, 4, -1, 2, 1, -5, 4]` に対して、連続した部分配列の中で最大の和を持つのは `[4, -1, 2, 1]` で、その和は 6 です[^2]。

明らかに、私たちは平凡なアルゴリズムを持っています、すなわち、すべての部分配列を列挙し、最大値を求めます：

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

このプログラムはHaskellで書くとシンプルになります：

```Haskell
import Data.List (tails, inits)

mss = maximum . map sum . segs
segs = concat . map tails . inits
```

ここで `tails` は「接尾辞」、`inits` は「接頭辞」を意味します：

```Haskell
tails [1, 2, 3] = [[1, 2, 3], [2, 3], [3], []]
inits [1, 2, 3] = [[], [1], [1, 2], [1, 2, 3]]
```

明らかに、「すべての接頭辞のすべての接尾辞」、または「すべての接尾辞のすべての接頭辞」はリストのすべての部分リストになります。

さて、アルゴリズム自体に戻りましょう。この単純なアルゴリズムの時間複雑度は $O(n^3)$ です。すべての部分配列を列挙するのに $O(n^2)$ 、各部分配列の和を求めるのに $O(n)$ が必要なので、結果として $O(n^3)$ になります。

実際には、最も優れたアルゴリズムは $O(n)$ の時間複雑度でこの問題を解決できます。Birdの論文は、単純なアルゴリズムから出発して、最も優れたアルゴリズムを_導き出す_方法について議論しています。まず、アルゴリズムを少し変形します：

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

これは単純な変形のように見えますが、私は3つのループの内側の2つを取り出しただけです。`max_tails` 関数は、配列 `arr` の `[0, end)` の範囲の最大の接尾辞の和を返します。最大値関数を $\uparrow$ で表すと、`max_tails` の関数バージョン、または数学的バージョンは次のように書けます：

$$
\begin{aligned}
\textsf{arr\[0:end\]} &\equiv \[ x_1, x_2, \cdots, x_n \] \\\\
m &= (\sum_{i=1}^n x_i) \uparrow (\sum_{i=2}^n x_i) \uparrow \cdots \uparrow (x_n) \uparrow 0
\end{aligned}
$$

これは一見すると非常に難解に見えますが、以下の例で説明します。まず、`max_tails([1, 2, 3], 3)`、つまり `[1, 2, 3]` のすべての接尾辞の和の最大値を考えてみましょう。

`max_tails` の計算は次のように書けます：

```text
(1 + 2 + 3) ↑ (2 + 3) ↑ (3) ↑ 0
```

つまり、`max_tails` は各接尾辞の和を計算し、それらの最大値を求めます。最大値を二項関数 $\uparrow$ として書くと、それを使って各接尾辞の和をつなげて最終的な式を構成することが自然になります。

朴素アルゴリズムでは、接尾辞の最大値を求めるために $O(n^2)$ の複雑さが必要です。以下では、秦九韶のアルゴリズムを用いてこの複雑さを $O(n)$ に減らします。

## 秦九韶のアルゴリズム

秦九韶は彼の著作「数術九章」で、高次方程式の近似解を求める方法を提案しました。その一部は現代の研究者によって「秦九韶のアルゴリズム」と呼ばれています。

$n$ 次多項式を考えてみましょう。

$$
f(x) = \sum_{i=0}^{n} a_i x^{i}
$$

特定の $t$ に対して、$f(t)$ の値を求める方法はいくつかあります。最も単純な方法は、各項の値を個別に求めてから合計することです。このアルゴリズムでは $O(n^2)$ 回の乗算が必要です。しかし、秦九韶のアルゴリズムは次のように注意しています：

$$
\begin{aligned}
& a_n x^n + a_{n-1} x^{n-1} + \cdots + a_0 \\\\
= & ((a_n x^{n - 1}) + (a_{n-1} x^{n - 2}) + \cdots + a_{1}) x + a_0 \\\\
= & \cdots \\\\
= & (((a_n x + a_{n - 1}) x + \cdots) x + a_{1})x + a_0
\end{aligned}
$$

これは反復形式で書くことができます：

$$
\begin{aligned}
s_0 &= a_n \\\\
s_{i + 1} &= s_i t + a_{n - i - 1}
\end{aligned}
$$

このように、$s_n = f(t)$ となります。この計算では、乗算は $n$ 回、加算も $n$ 回行われます。

一見すると、秦九韶のアルゴリズムは単なる _数値計算_ のアルゴリズムに過ぎません。それは `max_tails` 関数と何の関係があるのでしょうか？ここでは、「一般化」された秦九韶のアルゴリズム形式を考える必要があります。

まず、このアルゴリズムの入力は多項式である必要はありません。以下の式は秦九韶のアルゴリズムで書き換えることができます：

$$
\begin{aligned}
& x_1x_2x_3 + x_2x_3 + x_3 + 1 \\\\
=& (((x_1 + 1) x_2) + 1) x_3 + 1
\end{aligned}
$$

次に、秦九韶のアルゴリズムが考慮する演算は $(+, \cdot)$ である必要はありません。このアルゴリズムが公因数を抽出するとき、実際には乗法分配法則を用いて式を書き換えています。任意の二つの関数 $(\oplus, \odot)$ を考えると、それらが以下の条件[^3]を満たす限り、秦九韶のアルゴリズムを適用することができます：

+ $\odot$ は $\oplus$ に対して（右）分配します
$$
\begin{aligned}
a \cdot c + b \cdot c &= (a + b) \cdot c \\\\
(a \odot c) \oplus (b \odot c) &= (a \oplus b) \odot c \\\\
\end{aligned}
$$
+ $\odot$ の（左）単位元が存在します、つまり、すべての $x$ に対して $a \odot x = x$ となる $a$ が存在します。

以下の式を例にとります：

$$
\begin{aligned}
& ((x_1 \odot x_2) \odot x_3) \oplus (x_2 \odot x_3) \oplus (x_3) \oplus (1) \\\\
= & ((((x_1 \oplus 1) \odot x_2) \oplus 1) \odot x_3) \oplus 1
\end{aligned}
$$

ここで $1$ は $\odot$ の左単位元です。

## Kadaneのアルゴリズム

先ほど述べたように、`max_tails` は次のように書くことができます：

$$
\begin{aligned}
\textsf{arr\[0:end\]} &\equiv \[ x_1, x_2, \cdots, x_n \] \\\\
m &= (\sum_{i=1}^n x_i) \uparrow (\sum_{i=2}^n x_i) \uparrow \cdots \uparrow (x_n) \uparrow 0
\end{aligned}
$$

ここでの $\uparrow$ は $\oplus$ と見なすことができ、$+$ は $\odot$ と見なすことができます。加法には単位元 $0$ が存在し、分配性も明らかです：

$$
(a \uparrow b) + c = (a + c) \uparrow (b + c)
$$

続けて `[1, 2, 3]` を例にとります，

```text
  (1 + 2 + 3) ↑ (2 + 3) ↑ (3) ↑ 0
= ((1 + 2) ↑ 2 ↑ 0) + 3 ↑ 0
= (((1 ↑ 0) + 2) ↑ 0) + 3 ↑ 0
= ((((0 + 1) ↑ 0) + 2 ↑ 0) + 3) ↑ 0
```

この性質に基づいて、秦九韶のアルゴリズムを用いて `max_tails` を書き換えます：

```python
def step(s, a):
    # (s + a) ↑ 0
    return max(s + a, 0)

def max_tails(arr, end):
    s = 0
    for i in range(end):
        s = step(s, arr[i])
```

明らかに、書き換えた後の `max_tails` の時間複雑度は `O(n)` です。さらに、次のことに注意します：

$$
\begin{aligned}
& \textsf{max\\_tails}(\textsf{arr}, \textsf{end} + 1) \\\\
=\ & \textsf{step}(\textsf{step}(\cdots,\textsf{arr}\[\textsf{end} - 1\]), \textsf{arr}\[\textsf{end}\]) \\\\
=\ & \textsf{step}(\textsf{max\\_tails(arr, end)}, \textsf{arr}\[\textsf{end}\])
\end{aligned}
$$

`mss`が `[1, end + 1)` の `max_tails` を求める必要があるため、改写後の時間複雑度は $O(n^2)$ となります。しかし、上記の発見から、`max_tails(arr, i)` の値は `max_tails(arr, i - 1)` の値から求めることができます。これをすぐに $O(n)$ の `mss` 関数を得るために使用することができます：

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

間違いなく、これは大名鼎鼎の Kadane アルゴリズムです。

## 元の推論

Bird 教授は彼の研究を一つの段落にまとめています：

> 私は、明確だが非効率的な関数型プログラム、つまり手元の問題の仕様として機能するプログラムを取り、等式推論を用いてより効率的なものを計算するという特定のタスクに興味がありました。[^4]

つまり、Bird 教授は、シンプルで明確だが効率的でない関数型プログラムから出発し、一般的なルールを用いてプログラムを最適化し、より効率的なプログラムを得ることを好んでいます。Bird 教授は最も直感的なプログラムから出発し、一歩一歩推論を進めることで、KMP アルゴリズムのような有名なアルゴリズムを得ることができます。Bird 教授の論文では、最大部分配列和問題に対する元の推論は次のようになっています[^5]：

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

たった8つの等式で最終的なアルゴリズムを得ることができ、その技術は驚異的と言えます。さらに価値あることは、秦九韶のアルゴリズムや最終的な最適化が、推論過程で直接的な定理として示されていることです。これらの定理は、関数型プログラミングでよく使われる関数（`map`、`foldl`、`concat`など）の等式に関するものです。例えば、秦九韶のアルゴリズムは次のように直接表現することができます：

```text
fold (⊕) b . map (fold (⊗) a) . tails = fold (⊙) a
u ⊙ v = (u ⊗ v) ⊕ a
```

では、なぜ関数型プログラミングはこのようなプログラムの推論を容易にするのでしょうか？それは、関数型プログラミングが私たちにプログラムの *代数的* 性質により注意を向けさせるからです。Bird 教授はまた、[_The algebra of programming_](https://dl.acm.org/doi/10.5555/248932)という本も書いています。

## 可愛さの裏側

読者が上記のコードを [LeetCode](https://leetcode.com/problems/maximum-subarray/) に提出しようとすると、いくつかのテストで失敗するかもしれません。これはもちろん、アルゴリズムが間違っているわけではなく、最大部分配列和の問題の表現が異なるからです。

+ (LeetCode): 配列が与えられたとき、その最大部分配列和を出力します。
+ (Programming Pearls): 配列が与えられたとき、その最大部分配列和を出力します。もし配列の最大部分配列和が負であれば、0を返します。

私たちの問題は Programming Pearls のバージョンで、最大部分配列和が負になることはありません。

LeetCode バージョンの問題を処理するには、以下のような簡単な修正が可能です：

```haskell
mssNeg l
  | r == 0    = max l
  | otherwise = r
    where r = mss l
```

この方法は少し重たい感じがします。Bird 教授の導出過程を修正して、負数のケースを処理できるアルゴリズムを導出することは可能でしょうか？この問題に対して、私は理想的な解決策を提供していません。なぜなら、Bird 教授の最初の式が間違っているからです。

```haskell
--sum [] = 0
mss = maximum . map sum . segs
```

`segs` の結果には必ず空のリストが存在し、その空のリストを `sum` で評価すると、0が得られます。したがって、この式は負の最大和のケースを処理することはできません。また、ここでの秦九韶のアルゴリズムでは、`0` は `+` の単位元であるため、負数を処理できるアルゴリズムに修正するには、完全に一からやり直す必要があるかもしれません。

これは、関数型プログラムの導出の可愛らしさの裏側かもしれません。関数型プログラムの導出はプログラムの代数的性質に依存していますが、この代数的性質は時として脆弱で、問題を少し修正するだけで、元の代数的性質が消失することがあります。命令型プログラムにとっては、それ自体がこれらの代数的性質に依存していないため、些細な変更がすぐに機能するかもしれませんが、関数型、特に関数型プログラムの導出から導出されたプログラムにとっては、このような変更は些細なものではありません。

> Bird 教授にメールで意見を聞いてみてもいいですか？

それもいいかもしれません。しかし、Bird 教授は2022年4月4日に永遠に私たちを去りました。天国に手紙を送ることができる郵便局があるのでしょうか……


[^1]: Birdはこれを「最大セグメント和問題」（maximum segment sum problem）と呼んでいます。これはおそらく、問題が配列ではなくリストで表現されているからです。
[^2]: ウィキペディアからの引用
[^3]: 加法や乘法のように、交換と結合の両方が可能な関数については、左右どちらからでも公因数を取り出すことができます。交換ができない関数については、左右から公因数を取り出すと得られるアルゴリズムは異なり、その正確性を保証するためには異なる前提が必要です。
[^4]: Pearls of Functional Algorithm Design, preface, pp. ix
[^5]: 秦九韶のアルゴリズムは西洋では一般的に Horner's Rule と呼ばれています。
