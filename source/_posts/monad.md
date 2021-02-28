---
title: 从实用的角度看Monad（一）
date: 2021-02-26 20:20:35
tags: 函数式编程
---

## 从错误处理说起

错误处理是编程语言中的古老话题，从CPU的某些信号，到C++、托管平台里的`exception`，都是某种形态的错误处理。

在函数式语言中，我们一般采用`Result`或者说`Maybe`一类的东西做错误处理，例如说，在Haskell中定义自然数除法，可以这样定义：

```haskell
data Nat = Zero | S Nat 
  deriving Show
  
sub n Zero = n
sub Zero n = Zero
sub (S n) (S n') = sub n n'

eqb Zero n = False
eqb (S n) (Zero) = True
eqb (S n) (S n') = eqb n n'

--除法定义
ndiv Zero n' = Zero
ndiv n n' =
  if eqb n n' then ndiv (sub n n') n' else n
```

然而，这样定义并不能覆盖一个数学上无意义、计算上不停机的情况：

```haskell
ndiv (S Zero) Zero
```

所以，我们可以将这种情况定义为所谓的“错误”，把除此之外的情况定义为“正确”：

```haskell
ndiv n Zero = Nothing
ndiv Zero n' = Just Zero
ndiv n n' =
  if eqb n n' then ndiv (sub n n') n' else Just n
```

这里的`Just`和`Nothing`都是“Constructor”，它们构造的类型是`Maybe`：

```haskell
Just :: a -> Maybe a
```

这里的Maybe是一个`(* -> *)`，是所谓的“高阶类型”，它是类型的函数，即输入一个类型，输出一个类型。

这样看起来似乎并没有什么问题，然而，当我们要对多个`Maybe a`进行模式匹配式的处理时，麻烦立刻显现了出来：

```haskell
case m1 of
  Just t1 -> case m2 of
			   Just t2 -> ...
			   Nothing -> Nothing
  Nothing -> Nothing
```

明明当`t1`或者`t2`是`Nothing`的时候，计算就可以中止了，为什么在每次模式匹配中都需要对`Nothing`进行处理？能不能找到一个更好的办法来表达这些逻辑？

这就是Monad大显身手的时刻。我们先定义一个函数`comb`，它的类型是：

```haskell
comb :: Maybe a -> (a -> Maybe b) -> Maybe b
```

它的定义是：

```haskell
comb (Just a) f = f a
comb Nothing f = Nothing
```

上面的程序立刻可以被改写为：

```haskell
comb m1 (\t1 -> comb m2 (\t2 -> ...))
```

这，就是大名鼎鼎的Monad最直接、最简练的描述。在Philip Wadler的论文*Monads for functional programming*中，第一个例子就是Maybe.

一个Monad需要具有一个`return`函数（这里的`Just`）和一个`>>=`函数（这里的`comb`），第一个用来构造，第二个用来组合。它们需要满足一些定律，详见Philip的论文。

## map? fold? Monad如何和列表结合

Monad到这里，似乎还是很简单、很直接的东西，但其实不然。一旦把Monad用在解释器中，我们立刻就会遇到需要把Monad和列表结合起来的情况，而这其实并不是非常平凡的。

考虑一个最简单的语法树：

```haskell
data L = Id Int | F [Int]
```

我们对其求值：

```haskell
eval l = 
  case l of 
    Id i -> i
    F il -> foldl (+) 0 il
```

如果把`Id i`的那个分支改成

```haskell
Id i -> Just i
```

那么下面就不能用`foldl`进行求值了，原因很简单，`foldl`不可能在遇到一个`Nothing`的时候就停下来，它永远会从左至右对每一个元素进行操作，这可以从`foldl`的定义中找到答案（只考虑对列表的`foldl`）：

```haskell
foldl :: (b -> a -> b) -> b -> [a] -> b
foldl f i [] -> i
foldl f i (hd::tl) -> foldl f (f i hd) tl
```

也就是说，`f`被应用的时候，外边总有一个类似于这样：

```haskell
\x -> fold f x tl
```

的continuation，无论在`f`里怎么写（除非用`call/cc`）都不可能在碰到一个`Nothing`的时候就停下来，而显然，在`foldl`一个Monad列表的过程中，如果碰到了一个`Nothing`，整个表达式毫无疑问地必须是`Nothing`。这是`foldl`所无法实现的。

所以，我们需要自己实现一个`foldM`，具体如何实现呢？希望读者自行思考，这里给出我实现的版本：

```haskell
foldM :: Monad m => (b -> a -> m b) -> b -> [a] -> m b
foldM f z [] = return z
foldM f z (x:xs) = 
  f z x >>= \i -> foldM f i xs
```

## 列表本身也是Monad

列表，显然也是一个`(* -> *)`，那么，它如果作为一个Monad，会有什么样的意义呢？*Structure and Interpretation of Computer Programs*的第4.3节为这样的程序写了一个解释器：

```scheme
(list (amb 1 2 3) (amb 'a 'b))
```

这个东西的值是

```scheme
'((1 a) (1 b) (2 a) (2 b) (3 a) (3 b))
```

把上面的程序修改一下，我们就会发现有趣的事情：

```scheme
(>>= (amb 1 2 3)
     (lambda (x)
       (>>= (amb 'a 'b)
            (lambda (y)
              (return `(,x ,y))))))
```

这正是Monad化的表示。而在Haskell中，这就是List Monad：

```haskell
*Main> do x <- [1, 2, 3]; y <- ["a", "b"]; return (x, y)
[(1,"a"),(1,"b"),(2,"a"),(2,"b"),(3,"a"),(3,"b")]
```

（Haskell中没有不同类型的列表）

## "Id Monad"是什么？

考虑一个`id`类型函数，它把所有的输入都映射到输入本身，也就是说：

```haskell
id Int = Int
id Char = Char
```

那么，如果这个类型函数作为Monad，`>>=`和`return`又该是什么意思呢？

答案很明显：

```haskell
>>= :: a -> (a -> b) -> b
>>= a f = f a

return :: a -> a
return x = x
```

随便找一段程序，把它全部用“Id Monad”改写，我们会得到：

```haskell
--改写前
factor 0 = 1
factro n = n * (factor n - 1)

--改写后
factor 0 = 1
factor n = n        >>= (\x -> 
           n - 1    >>= (\y ->
           factor y >>= (\z ->
           z * x    >>= (\a
           return a     ))))
```

这是什么？有人可能会说，这是CPS(*continuation passing style*). 实际上，这和CPS还有一段距离，更准确的说法是，这是ANF(*administrative normal form*)的一种同构。

## State Monad

有了上面的知识，我们还不足以完全搞明白State Monad. 因为，我们还缺少一个推广、一个直观想法和两个创造。欲知后事如何，请看下回分解。

