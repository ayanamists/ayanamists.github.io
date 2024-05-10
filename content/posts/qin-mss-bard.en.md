---
title: "Solving the Maximum Subarray Sum Problem Using the Qin Jiushao Algorithm"
date: 2023-12-23T20:44:33+08:00
draft: false
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

> 問有兩尖田一段，其尖長不等，兩大斜三十九步，兩小斜二十五步，中廣三十步，欲知其積幾何？
>
> 術曰：以少廣求之，翻法入之。置半廣自乘為半冪，與小斜冪相減相乘為小率， 以半冪與大斜冪相減相乘為大率。以二率相減餘自乘為實，併二率倍之為從上廉，以一為益隅，開翻法三乘方得積。

All along, some friends have looked down on functional programming. They feel that functional programming just adds some unnecessary abstraction, and the cost is a decrease in performance. Friends who hold this view, I'm afraid, have definitely not read Richard Bird's [_PEARLS OF FUNCTIONAL ALGORITHM DESIGN_](https://www.cambridge.org/core/books/pearls-of-functional-algorithm-design/B0CF0AC5A205AF9491298684113B088F). The content discussed in this book is indeed too obscure, and I have not read it all the way through.

Fortunately, Richard Bird wrote a paper in 1989 that is only 5 pages long, [_Algebraic identities for program calculation_](https://academic.oup.com/comjnl/article/32/2/122/543545?login=false). This paper uses a famous problem--the maximum subarray sum[^1] (Maximum subarray problem) to fully demonstrate the unique understanding of algorithmic problems by functional programming thinking: the maximum subarray sum problem can be solved by the Qin Jiushao algorithm.

## Maximum Subarray Sum Problem

The goal of the maximum subarray sum problem is to find a continuous subsequence in the one-dimensional direction of the sequence, so that the sum of this subsequence is the largest. For example, for a sequence `[−2, 1, −3, 4, −1, 2, 1, −5, 4]`, the continuous subsequence with the largest sum is `[4, −1, 2, 1]`, and its sum is 6 [^2].

Obviously, we have a trivial algorithm, that is, enumerate all subarrays, and then find the maximum value:

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

This program would be more concise if written in Haskell:

```Haskell
import Data.List (tails, inits)

mss = maximum . map sum . segs
segs = concat . map tails . inits
```

Where `tails` is "suffix", `inits` is "prefix":

```Haskell
tails [1, 2, 3] = [[1, 2, 3], [2, 3], [3], []]
inits [1, 2, 3] = [[], [1], [1, 2], [1, 2, 3]]
```

It is easy to see that "all suffixes of all prefixes", or "all prefixes of all suffixes" are all sublists of a list.

Now let's return to the algorithm itself. The time complexity of this trivial algorithm is $O(n^3)$. Enumerating all subarrays requires $O(n^2)$, and summing each subarray requires $O(n)$, so it constitutes $O(n^3)$.

In fact, the best algorithm can solve this problem within $O(n)$ time complexity. Bird's paper is discussing how to _derive_ the best algorithm from the trivial one. First, slightly modify the algorithm:

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

This seems to be a trivial transformation, I just broke down the two layers inside the three loops. The `max_tails` function will return the maximum suffix sum of the array `arr` in the `[0, end)` interval. We use $\uparrow$ to represent the maximum value function, the functional version of `max_tails`, or the mathematical version, can be written as:

$$
\begin{aligned}
\textsf{arr\[0:end\]} &\equiv \[ x_1, x_2, \cdots, x_n \] \\\\
m &= (\sum_{i=1}^n x_i) \uparrow (\sum_{i=2}^n x_i) \uparrow \cdots \uparrow (x_n) \uparrow 0
\end{aligned}
$$

This looks very puzzling at first glance, let's explain it with an example. First, consider `max_tails([1, 2, 3], 3)`, that is, the maximum value of all suffix sums of `[1, 2, 3]`.

The calculation of `max_tails` can be written as:

```text
(1 + 2 + 3) ↑ (2 + 3) ↑ (3) ↑ 0
```

That is to say, `max_tails` will calculate each suffix sum and find their maximum value. When the maximum value is written as a binary function $\uparrow$, it can naturally connect each suffix sum to form the final expression.

In the naive algorithm, finding the maximum suffix sum requires a complexity of $O(n^2)$. Below, we use the Qin Jiushao algorithm to reduce this complexity to $O(n)$.

## Qin Jiushao Algorithm

In his book "Mathematical Art in Nine Chapters", Qin Jiushao proposed a method for finding approximate solutions to high-degree equations. One part of this is referred to as the "Qin Jiushao algorithm" by modern researchers.

Consider the $n$-degree polynomial

$$
f(x) = \sum_{i=0}^{n} a_i x^{i}
$$

For a given $t$, there are several ways to find the value of $f(t)$. The most trivial method is to calculate the value of each term separately and then add them up. This algorithm requires $O(n^2)$ multiplications. The Qin Jiushao algorithm notes that:

$$
\begin{aligned}
& a_n x^n + a_{n-1} x^{n-1} + \cdots + a_0 \\\\
= & ((a_n x^{n - 1}) + (a_{n-1} x^{n - 2}) + \cdots + a_{1}) x + a_0 \\\\
= & \cdots \\\\
= & (((a_n x + a_{n - 1}) x + \cdots) x + a_{1})x + a_0
\end{aligned}
$$

This can be written in an iterative form:

$$
\begin{aligned}
s_0 &= a_n \\\\
s_{i + 1} &= s_i t + a_{n - i - 1}
\end{aligned}
$$

In this way $s_n = f(t)$. In this calculation, multiplication is performed $n$ times, and addition is also performed $n$ times.

At first glance, the Qin Jiushao algorithm seems to be just a _numerical calculation_ algorithm. What does it have to do with the `max_tails` function? We need to consider the "generalized" form of the Qin Jiushao algorithm.

First, the input to this algorithm does not necessarily have to be a polynomial. The following expression can still be rewritten using the Qin Jiushao algorithm:

$$
\begin{aligned}
& x_1x_2x_3 + x_2x_3 + x_3 + 1 \\\\
=& (((x_1 + 1) x_2) + 1) x_3 + 1
\end{aligned}
$$

Secondly, the operations considered by the Qin Jiushao algorithm are not necessarily $(+, \cdot)$. When this algorithm extracts common factors, it is actually rewriting the formula using the distributive law of multiplication. Consider any two functions $(\oplus, \odot)$, as long as they satisfy the following conditions[^3], the Qin Jiushao algorithm can be applied:

+ $\odot$ distributes over $\oplus$ (right)
$$
\begin{aligned}
a \cdot c + b \cdot c &= (a + b) \cdot c \\\\
(a \odot c) \oplus (b \odot c) &= (a \oplus b) \odot c \\\\
\end{aligned}
$$
+ There exists a (left) unit element for $\odot$, i.e., there exists $a$ such that $\forall x, a \odot x = x$

Taking the previous expression as an example:

$$
\begin{aligned}
& ((x_1 \odot x_2) \odot x_3) \oplus (x_2 \odot x_3) \oplus (x_3) \oplus (1) \\\\
= & ((((x_1 \oplus 1) \odot x_2) \oplus 1) \odot x_3) \oplus 1
\end{aligned}
$$

Where $1$ is the left identity element of $\odot$.

## Kadane Algorithm

As mentioned earlier, `max_tails` can be written as:

$$
\begin{aligned}
\textsf{arr\[0:end\]} &\equiv \[ x_1, x_2, \cdots, x_n \] \\\\
m &= (\sum_{i=1}^n x_i) \uparrow (\sum_{i=2}^n x_i) \uparrow \cdots \uparrow (x_n) \uparrow 0
\end{aligned}
$$

Here, $\uparrow$ can be seen as $\oplus$, and $+$ can be seen as $\odot$. Addition has the identity element $0$, and the distributive property is obvious:

$$
(a \uparrow b) + c = (a + c) \uparrow (b + c)
$$

Continuing with the example of `[1, 2, 3]`,

```text
  (1 + 2 + 3) ↑ (2 + 3) ↑ (3) ↑ 0
= ((1 + 2) ↑ 2 ↑ 0) + 3 ↑ 0
= (((1 ↑ 0) + 2) ↑ 0) + 3 ↑ 0
= ((((0 + 1) ↑ 0) + 2 ↑ 0) + 3) ↑ 0
```

Based on this property, we rewrite `max_tails` using the Qin Jiushao algorithm:

```python
def step(s, a):
    # (s + a) ↑ 0
    return max(s + a, 0)

def max_tails(arr, end):
    s = 0
    for i in range(end):
        s = step(s, arr[i])
```

Obviously, the time complexity of the rewritten `max_tails` is `O(n)`. Not only that, we notice:

$$
\begin{aligned}
& \textsf{max\\_tails}(\textsf{arr}, \textsf{end} + 1) \\\\
=\ & \textsf{step}(\textsf{step}(\cdots,\textsf{arr}\[\textsf{end} - 1\]), \textsf{arr}\[\textsf{end}\]) \\\\
=\ & \textsf{step}(\textsf{max\\_tails(arr, end)}, \textsf{arr}\[\textsf{end}\])
\end{aligned}
$$

Since `mss` needs to find the `max_tails` of `[1, end + 1)`, the time complexity is $O(n^2)$ after rewriting. However, the above discovery tells us that the value of `max_tails(arr, i)` can be obtained from the value of `max_tails(arr, i - 1)`. This can immediately be used to get an $O(n)$ `mss` function:

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

Without a doubt, this is the famous Kadane algorithm.

## Original Derivation

Professor Bird summarized his research in a paragraph:

> I was interested in the specific task of taking a clear but inefficient functional program, a program that acted as a specification of the problem in hand, and using equational reasoning to calculate a more efficient one. [^4]

In other words, Professor Bird likes to start with a simple, clear, but inefficient functional program, optimize the program through general rules, and thus obtain a more efficient program. Starting from the most intuitive program, Professor Bird can even derive famous algorithms like the KMP algorithm after step-by-step derivation. The original derivation of the maximum subarray sum problem in Professor Bird's paper is[^5]:

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

With only 8 equations, the final algorithm was obtained, which can be said to be magical. What's more valuable is that both the Qin Jiushao algorithm and the final optimization are shown as direct theorems in the derivation process. And these theorems are identities about commonly used functions in functional programming (such as `map`, `foldl`, `concat`). For example, the Qin Jiushao algorithm can be directly expressed as:

```text
fold (⊕) b . map (fold (⊗) a) . tails = fold (⊙) a
u ⊙ v = (u ⊗ v) ⊕ a
```

So, why can functional programming conveniently carry out this kind of program derivation? Because functional programming makes us pay more attention to the *algebraic* properties of the program. Professor Bird also wrote a book called [_The algebra of programming_](https://dl.acm.org/doi/10.5555/248932).

## The Unlovable Side

If readers try to submit the above code to [LeetCode](https://leetcode.com/problems/maximum-subarray/), they may find that it fails on some tests. This is not because the algorithm is wrong, but because the statement of the maximum subarray sum problem is different.

+ (LeetCode): Given an array, give its maximum subarray sum.
+ (Programming Pearls): Given an array, give its maximum subarray sum. If the maximum subarray sum of the array is negative, then return 0.

Our problem is the Programming Pearls version, which will never return a negative maximum subarray sum.

If you want to handle the LeetCode version of the problem, a simple fix is:

```haskell
mssNeg l
  | r == 0    = max l
  | otherwise = r
    where r = mss l
```

This approach is a bit clumsy. Can we modify Professor Bird's derivation process to derive an algorithm that can handle negative numbers? For this problem, I haven't given any ideal solutions. Because Professor Bird's first equation is wrong.

```haskell
--sum [] = 0
mss = maximum . map sum . segs
```

There must be an empty table in the result of `segs`, and after the empty table is evaluated with `sum`, it will get 0. So obviously, this equation cannot handle the situation of negative maximum sum. Considering that in the Qin Jiushao algorithm here, `0` is the unit element of `+`, I think that to modify it to be able to handle negative numbers, it may need to be completely overhauled.

This may be the unlovable side of functional program derivation. Functional program derivation depends on the algebraic properties of the program, and these algebraic properties are sometimes fragile. A slight modification of the problem may cause the original algebraic properties to disappear. For imperative programs, perhaps they do not depend on these algebraic properties, so a trivial change can work, but for functional, especially programs derived from functional program derivation, this modification is not trivial.

> Can you email Professor Bird for his opinion?

Maybe. Professor Bird left us forever on April 4, 2022, is there a post office that can send letters to heaven...

[^1]: Bird calls it the "maximum segment sum problem", perhaps because the problem is given with a list rather than an array
[^2]: Excerpted from Wikipedia
[^3]: For functions like addition and multiplication that are both commutative and associative, it is possible to extract common factors to the left and right. For non-commutative functions, extracting common factors to the left and right results in different algorithms, requiring different premises to ensure correctness.
[^4]: Pearls of Functional Algorithm Design, preface, pp. ix
[^5]: The Qin Jiushao algorithm is generally known as Horner's Rule in the West
