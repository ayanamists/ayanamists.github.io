---
title: "函数式 Leetcode 百题 -- 排列"
date: 2024-04-24T19:38:38+08:00
draft: false
---

算法题，是很多程序员的梦魇之一。算法本身当然是有趣的，但不会做题的抓狂、面试回答不出问题的窘迫，没有人会喜欢。

应对算法题挑战的传统方式是 **刷题** 。 刷题，也就是做大量的题目来学习算法。很多人笃信一种理论，无论是数学学习还是算法学习，都必须进行大量的练习。

我也认同这种理论。冯诺依曼的箴言还挂在本博客的首页上：

> Young man, in mathematics you don't understand things. You just get used to them.

直觉的认识，直觉的经验，是无比宝贵的财富。练习是获取这种宝贵财富最可靠的手段。简而言之，想要学习算法，你必须刷题，必须练习。

然而，我确实极度厌烦一种算法学习的常规路径。那就是看到某道题不会做，思考了 10 分钟还是不知道从何入手，那么就去寻找答案。也许你 “理解” 了答案或者记住了答案，下次看到相同的题目的时候就套进去。只要你见过的答案够多，那么你能 “套” 的题目也够多。

为什么我不喜欢这种学习方法呢？也许是我不够聪明吧，我很难 “理解” 别人给的答案。太多的人只是把最后的高效算法告诉你，而忽略了直觉建立的过程，更别提算法的证明了。从这些人的答案里，我既没有感受到他们对于问题的深刻理解，又没有感受到算法本身的美感。

难道我们只能通过给 “天才” 设计的教育来学习算法吗？

不。世界上还有一群人，在尝试着用纯函数式的语言，用等式推导的方法来设计程序。这些人同样也珍视直觉的价值。他们会从一个清晰而不高效的程序出发，一步步地给出最终的高效算法。在这个过程中，算法的美、算法的 insight 纤毫毕现。已故的 Richard Bird，台湾的穆信成老师都是这种方式的先驱。

我想，虽然等式推导很多时候非常困难，但是把算法的 non-trivial 之处是什么、算法的直觉是什么讲清楚，也许没有那么困难。

本系列文章就是我的尝试。我将在这些文章中，讲解我是怎么用纯函数式编程解决 Leetcode 的。至于为什么选择 Leetcode 而不是难度更高的 Codeforces，我认为难度太高，也许不是好事。高难度的问题很可能需要多个算法共同来处理，而我们更想在每篇文章中关注某类具体的算法。（另一个原因是，我本人对于算法仍然是某种意义的 “初学者”，难度太高我也处理不来）

选择 Leetcode 的一个后果是，我们必须用 Racket 或者 Scala 而非 Haskell 来提交代码。这有时候会造成一些困扰，不过我的经验是，大部分我给出的 Haskell 代码可以直接翻译到 Scala 上。每道题目我也会给出对应的 Scala 解。如果确实有问题，比如 Scala 没有 `Data.Graph` 这种不可变数据结构库，那么我们会视情况退回到非纯函数式编程。不过，这种情况总体应该是较少的。在每道题目的最后，我可能也会把对应的命令式版本给出，也许我们还会探讨一下函数式版本和命令式版本的区别。

这些文章不是 “函数式编程入门”。虽然我也许会提示某个库函数的用法，但是总体来说，这些文章假设你已经有了纯函数式编程的基本知识 -- 递归，列表，fold, map 等等。如果对函数式编程一无所知，那么我认为读懂这些文章是相当有挑战性的事情。

> 路漫漫其修远兮，吾将上下而求索

## 31. 下一个排列

### 题目描述

以下是 [Leetcode](https://leetcode.cn/problems/next-permutation/) 的原始描述：

> 整数数组的一个 **排列**  就是将其所有成员以序列或线性顺序排列。
>
> 例如，`arr = [1,2,3]` ，以下这些都可以视作 `arr` 的排列：`[1,2,3]`、`[1,3,2]`、``[3,1,2]`、`[2,3,1]` 。
> 整数数组的 下一个排列 是指其整数的下一个字典序更大的排列。更正式地，如果数组的所有排列根据其字典顺序从小到大排列在一个容器中，那么数组的 下一个排列 就是在这个有序容器中排在它后面的那个排列。如果不存在下一个更大的排列，那么这个数组必须重排为字典序最小的排列（即，其元素按升序排列）。
>
> 例如，`arr = [1,2,3]` 的下一个排列是 `[1,3,2]` 。
> 类似地，`arr = [2,3,1]` 的下一个排列是 `[3,1,2]` 。
> 而 `arr = [3,2,1]` 的下一个排列是 `[1,2,3]` ，因为 `[3,2,1]` 不存在一个字典序更大的排列。
> 给你一个整数数组 `nums` ，找出 `nums` 的下一个排列。
>
> 必须 原地 修改，只允许使用额外常数空间。

我们是在纯函数式编程。纯函数式编程没有所谓的 “修改”，所以最后一句话我们无视即可。

### 第一想法

一般来说，很多算法问题能够给一个极为简洁（但低效）的纯函数式解。我们把这个解称之为程序的标准参照（Spec），我们的思考也从这里出发。然而，这个问题却似乎不是这种情况。考虑这个问题的数学表达：

$$
\text{next}(a) = \begin{cases}
  \min(\mathbb{P}) & a = \max(\mathbb{P}) \\\\
  \min(\\{ m | m \in \mathbb{P}, m > a \\}) & \text{otherwise}
\end{cases}
$$

这没有给我们多少灵感。事实上，在很多编程语言里，把 $\mathbb{P}$，也就是某个列表的所有排列求出来，比这个问题还要麻烦。

这种情况下，一个直觉的尝试是，**简单的递归** 能不能解决问题呢？

### 递归

十进制整数可以被表示为一个列表。某些人把它叫做 “高精度” 算法。例如，你可以用 `[1, 0, 0]` 表示 $100$ . 这种表示的好处是，整数 $a$, $b$， $a \ge b$ 当且仅当 `a >= b`，其中 `>=` 恰恰就是字典序比较。

显然地，十进制整数也可以定义 `next` 函数。这就是大名鼎鼎的 “后继” 函数，即 $f(x) = x + 1$ , 我们把这个函数记作 `next10`.

```Haskell
-- next10 [1, 1, 0] = [1, 1, 1]
next10 ...
```

这确实给了我们一些灵感。例如，`next10 [1, 1, 0] = [1, 1, 1]`，这是因为 `next [1, 0] = [1, 1]`. 可以很明显地看到这里的递归关系。具体来说，对于 `x:xs` 表示的整数，

```Haskell
next10 (x:xs) = x:(next10 xs)
```

上面的等式在大部分时候都成立。除非，`next10 xs` 已经 “求不出来了”，比如 `[1, 9, 9]` 的下一个数字（后继）是 `[2, 0, 0]`，这时就必须要 “进位”。更严格地说，`99` 是 **最大** 的两位十进制整数，所以上面的等式不成立。只要 `xs` 还不是最大值，那么上面的等式就成立。

无独有偶，题目要求的 `next` 函数具有同样的性质。根据这个想法，构造递归函数：

```Haskell
next (x:xs)
    | isMax xs  = tick x xs
    | otherwise = x:next xs
```

那么，问题就变成了

1. 如何定义 `isMax`
2. 如果 `xs` 已经是最大值了，那么 `tick` 函数该怎么定义呢？

### `isMax` 的定义

对于一个排列 `xs` 来说，“最大” 就是说，无论怎么重新排列，新的 `xs'` 一定有 `xs >= xs'`.

什么样的排列具有这种性质呢？一个很自然的想法是，因为这是字典序，所以如果想要排列尽可能大，那就一定要把大的数放在前面。换句话说，“降序” 的排列一定是最大的。

```Haskell
isMax :: Ord a => [a] -> Bool
isMax = down
```

在 Haskell 里，判断一个列表的升降，我们可以使用 "zip with tail" 的方法。具体来说，就是首先把列表和它的 `tail` 进行 `zip`，然后再使用 `map`, `all` 之类的函数来判断。

```Haskell
dup :: [a] -> [(a, a)]
dup xs = zip xs (tail xs)

down :: Ord a => [a] -> Bool
down = all (uncurry (>=)) . dup
```

### `tick` 的定义

如果 `xs` 已经满足了 `down`，换言之，它已经是最大的了，那么 `x:xs` 必须要进行比较复杂的操作才能得到下一个枚举。这个复杂的操作，被我称为 `tick`.

例如，`[2, 4, 3, 1]` 的下一个枚举是 `[3, 1, 2, 4]`. 这时 `x` 是 `2`, `xs` 是 `[4, 3, 1]`，`xs` 满足 `down`，所以，这时不能再进行简单的的递归，而要用 `tick` 进行计算。

继续以 `[2, 4, 3, 1]` 为例。直觉上来讲，要求 `[2, 4, 3, 1]` 的下一个枚举 `ys@(y:ys')`, 可以分为两步：

- 确定 `y` 是什么
- 确定 `ys'` 是什么

这时的 `y` 是 `3`. 很容易想到，`3` 是 `xs = [4, 3, 1]` 中，**最小的** 大于 `x = 2` 的数。我们把这种数叫做 `pivot`. 可以写出定义代码：

```Haskell
pivot x xs = minumum [ x' | x' <- xs, x' > x  ]
```

注意到 `xs` 是有序的，以上代码可以改写为

```Haskell
pivot x = last . takeWhile (> x)
```

需要指出的是，`pivot` 函数是一个偏函数，我们用了不安全的 `last` 和 `minimum` 操作，这意味着在调用时，必须保证至少存在一个 `x' > x`.

如果能够找到这样的 `y > x`，那么我们就可以保证，`y:ys'` 绝对比 `x:xs` 大。所以，`ys'` 的构造要使得 `y:ys'` 尽可能小。也就是说，`ys'` 应该是最小的一个排列。和我们上面对于最大值的讨论一样，最小的排列就是单调递增的排列。这只需要排序即可。

`ys'` 中的元素自然是 `xs` 除掉 `y` 之后，再加上 `x`. 例如，上面的例子 `[2, 4, 3, 1]`，可以给出

```Haskell
x:xs = [2, 4, 3, 1]

y = last . takeWhile (> x) $ xs
  = last [4, 3]
  = 3

ys'' = delete 3 xs
     = [4, 1]

ys' = sort (x:ys'')
    = sort [2, 4, 1]
    = [1, 2, 4]

ys = y:ys'
   = [3, 1, 2, 4]
```

最后，还需要讨论一种情况，那就是 `pivot x xs = ⊥`. 这种情况必须先行判断，以免出现错误。什么时候 `pivot` 不存在呢？那就是 `x:xs` 已经满足 `down` 的时候，例如 `[4, 3, 2, 1]`，这种时候，只需要把输入反转即可。

根据以上讨论，我们给出 `tick` 函数的定义：

```Haskell
import Data.List (delete)

tick :: Ord a -> a -> [a] -> [a]
tick x xs
    | null l    = reverse (x:xs)
    | otherwise = y:(sort (x:ys'))
    where l = takeWhile (> x) xs
          y = last l
          ys' = delete y xs
```

```Haskell
> tick 4 [3, 2, 1]
[1, 2, 3, 4]

> tick 2 [4, 3, 1]
[3, 1, 2, 4]
```

毫无疑问，目前的 `next` 函数已经能够正确地解决这个问题了。读者不妨把以上的 Haskell 代码翻译为 Scala，在 Leetcode 里试一试。从递归出发，我们发现这个问题的解非常直觉，可以简单而轻松地给出。

### 更高效的代码

上面定义的 `next` 函数虽然优雅直观，但是不够高效。主要的问题在于，

1. `tick` 中，不需要用到 `sort`. `xs` 是有序的，可以利用这点在 $O(n)$ 内把新的有序序列组合起来
2. `down` 被 `next` 调用了好多次，每次调用都是 $O(n)$ 的，导致总体复杂度出现了 $O(n^2)$.

问题 1. 很简单，只需要重新写一下 `tick` 即可。

问题 2. 比较麻烦。我们需要仔细观查一下计算的过程，例如 `next [3, 5, 1, 4, 2]`：

```Haskell
next [3, 5, 1, 4, 2]
= 3:(next [5, 1, 4, 2])
= 3:(5:(next [1, 4, 2]))
= 3:(5:(tick 1 [4, 2]))
```

可以看到，这个计算总是会具有一种 “形状”，那就是输入的前一部分保持不变，后一部分被 `tick`.

```Haskell
[3, 5,           1, 4, 2]
^^^^^^           #######
保持不变的部分     tick 的部分
```

而它们的分界线，直觉上来讲，就是 `down` 由 `False` 变为 `True` 的时刻。我们再回到 `next` 函数：

```Haskell
next (x:xs)
    | down xs  = tick x xs
    | otherwise = x:next xs
```

在每次递归的时候，`next (x:xs)` 都会计算 `down xs`，再用刚才的例子，

```Haskell
-- 第一次递归
down [5, 1, 4, 2] = False
-- 第二次递归
down [1, 4, 2]    = False
-- 第三次递归
down [4, 2]       = True
```

事实上，`down` 每次都顺序地计算 `tails` 里的下一个元素：

```Haskell
> tails [3, 5, 1, 4, 2]
[[3,5,1,4,2],[5,1,4,2],[1,4,2],[4,2],[2],[]]
```

不妨把 `next` 每次用到的 `down xs` 提前[^1]算出来，并放到一个列表里

```Haskell
import Data.List (tails)

nextDown (x:xs) (d:ds)
    | d = tick x xs
    | otherwise = x:nextDown xs ds

next xs = nextDown xs (downs xs)
downs = drop 1 . map down . tails
```

如果我们可以在 $O(n)$ 时间内算出 `downs`，问题 2. 就迎刃而解了。直觉上来说，这是很容易的，因为 `down [4, 3, 2, 1]` 本来就是要保证：

- `4 >= 3`
- `down [3, 2, 1]`

换句话说，当我们要算 `down [4, 3, 2, 1]` 的时候，`down [3, 2, 1]` 已经被计算过了，只是需要一种方式把这种 “上一次的计算” 存起来以便后续使用。

函数式编程社区早就有了对于这种问题的解决方法： scan theorem.

Scan theorem 指的是

```Haskell
map (foldl op a) . inits = scanl op a
```

类似地

```Haskell
map (foldr op a) . tails = scanr op a
```

利用这个定理，我们给出以下程序计算过程. 注意，为了避免问题，我们定义了

```Haskell
tails' = filter (not . null) . tails
```

```Haskell
downs = { definition }
        drop 1 . map down . tails'
      = { definition }
        drop 1 . map (all (uncurry (>=)) . dup) . tails'
      = { map distributivity }
        drop 1 . map (all (uncurry (>=))) . map dup . tails'
      = { need another proof (1) }
        drop 1 . map (all (uncurry (>=))) . tails . dup
      = { definition }
        drop 1 . map (foldr (&&) True . map (uncurry (>=))) . tails . dup
      = { fold-map fusion }
        drop 1 . map (foldr join True) . tails . dup
      = { scan theorem }
        drop 1 . scanr join True . dup
          where join (x, x1) r = x >= x1 && r
```

(1) 需要一个新的证明，它指的是下面的代码是等价的：

```Haskell
> map dup $ tails' [4, 2, 3, 1]
[[(4,2),(2,3),(3,1)],[(2,3),(3,1)],[(3,1)],[]]

> tails $ dup [4, 2, 3, 1]
[[(4,2),(2,3),(3,1)],[(2,3),(3,1)],[(3,1)],[]]
```

证明略去。

### 总结

结合上面的讨论，我们可以给出高效的函数式实现：

```Haskell
tick x xs = build l r x
    where (l, r) = span (> x) xs
          build [] r x = reverse (x:r)
          build l  r x = last l : reverse (init l ++ [x] ++ r)

dup xs = zip xs (tail xs)
downs = scanr (\(x, x1) r -> (x >= x1) && r) True . dup

next xs = l ++ tick (head r) (tail r)
    where (l, r) = splitAt (n - 1) xs
          n = length $ takeWhile not $ downs xs
```

这是 $O(n)$ 时间复杂度的。不过，我们的常数确实会比命令式程序大些。理论上来说，上一节定义的 `nextDown` 函数的实现要更高效一些（避免了 `length` 和 `splitAt`），但是我仍然觉得本节给出的实现更清楚。

我把等价的 Scala 代码提交到了 leetcode:

```Scala
object Solution {
    def nextPermutation(nums: Array[Int]): Unit = {
        val nextNums = next(nums.toList).toArray
        for (i <- nums.indices) {
            nums(i) = nextNums(i)
        }
    }

    def dup(xs: List[Int]): List[(Int, Int)] = xs zip xs.drop(1)

    def tick(x: Int, xs: List[Int]): List[Int] = {
        val (l, r) = xs.span(_ > x)
        if (l.isEmpty) r.reverse ++ List(x)
        else l.last :: (l.init ++ List(x) ++ r).reverse
    }

    def tailsDown(xs: List[(Int, Int)]): List[Boolean] = 
        xs.scanRight(true) { case ((x, x1), r) => (x >= x1) && r }

    def next(xs: List[Int]): List[Int] = {
        val n = tailsDown(dup(xs)).takeWhile(!_).length
        val (l, r) = xs.splitAt(n - 1)
        l ++ tick(r.head, r.tail)
    }
}
```

“下一个排列” 问题是少有的命令式程序可以比函数式程序简洁的问题。甚至，Haskell 的 [Data.Permute](https://hackage.haskell.org/package/permutation-0.5.0.5/docs/Data-Permute.html) 库，也用了命令式的方法。

在命令式程序中，上面的算法可以被描述为两个过程：

1. 从右向左遍历，找到最长的下降序列 $a[i \dots (\text{len - 1})]$
2. 进行计算，
   - 如果 $i = 0$，那么反转数组
   - 其他情况，找到 $j \in [i, \text{(len - 1)}]$，使得

     - $a[j] > a[i - 1]$
     - $\forall k \ge i, a[k] > a[i - 1] \to a[k] \ge a[j]$

     交换 $i - 1, j$，反转 $a[i \dots (\text{len - 1})]$

如果读者完全理解了上面的函数式算法，这个命令式算法想必是非常直白的。但如果把这个算法直接端到你的面前喂给你，你真的能明白它为什么是正确的吗？

让我们用一个 Python 程序结束今天的故事吧！

```Python
def swap(a, i, j):
    tmp = a[i]
    a[i] = a[j]
    a[j] = tmp

def rev(a, start, end):
    l = end - start
    for i in range(start, start + l // 2):
        swap(a, i, end - (i - start) - 1)

def solve(a):
    for i in range(len(a) - 1, -1, -1):
        if i >= 1 and a[i - 1] < a[i]:
            break
    if i == 0:
        rev(a, 0, len(a))
    else:
        j = len(a) - 1
        while j >= i and a[j] <= a[i - 1]:
            j -= 1
        swap(a, i - 1, j)
        rev(a, i, len(a))
```

[^1]: 注意这不是精确的描述，Haskell 是惰性的，求值过程需要仔细分析

