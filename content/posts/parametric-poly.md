---
layout: post
title: C++ 模板与参数多态的根本区别
date: 2021-09-14 15:53:47
tags: 
  - C++
  - 编程语言
categories:
  - 编程语言
---

## C++ 模板与参数多态

模板(template)是一种 C++ 语言特性，类似于：

```cpp
template <class T>
T id(T x) {
    return x;
}
```

这里的 `T` 可以是任意类型。很多人都注意到了这与参数多态(parametric polymorphism)的相似性，更有甚者直接把模板机制看作是一种参数多态。比如这份[ppt](https://www.cs.bham.ac.uk/~hxt/2015/c-plus-plus/templates-slides.pdf)和这个[网页](https://catonmat.net/cpp-polymorphism)。

然而这篇博客中，我将会说明，**无论 C++ 模板到底是不是一个好的语言特性，它都和参数多态有根本性的区别**。

## 到底什么是参数多态

参数多态的历史要追溯一份讲义([Fundamental Concepts in Programming Languages](https://link.springer.com/article/10.1023%2FA%3A1010000313106))，讲义虽然主要描述的是 CPL 语言，但是实际上这一段应该描述的是 LISP 的 MAP 函数：

> Parametric polymorphism is more regular and may be illustrated by an example. Suppose f is a function whose argument is of type α and whose results is of β (so that the type of f might be written α ⇒ β), and that L is a list whose elements are all of type α (so that the type of L is α list). We can imagine a function, say Map, which applies f in turn to each member of L and makes a list of the results. Thus Map[f,L] will produce a β list. We would like Map to work on all types of list provided f was a suitable function, so that Map would have to be polymorphic. However its polymorphism is of a particularly simple parametric type which could be written

$$
(α → β , α\ list) → β \ list
$$

1975 年的 ML 语言在类型系统的层面上实现了这种多态。在 ML 类语言中，一个函数的类型中可以含有类型变量，例如如下的`id`函数：

```ocaml
let id (x:'a) = x
```

`id`函数的类型是`'a -> 'a`，其中`'a`就是一个类型变量，它可以与**任意**类型相匹配，所以得到

```ocaml
id 1
id "ssdfs"
id 1.23
```

都是合法的表达式。ML 类语言的类型系统可以看作是对 System F 进行限制和扩展后的系统。System F 可以看作对 λ 表达式的扩展，在 System F 中，λ 表达式的每个绑定变量都必须给定一个类型，而这个类型则是通过 $Λ$（大写的λ）引入的。

具体来说，最基本的λ表达式 $λx.x$ 在 System F 中应该写作：

$$
Λt.λ (x:t).x
$$

而这个表达式的类型是

$$
∀t.t → t
$$

这个大 $Λ$ 引入的类型变量在使用时需要得到显式 apply（就像对小 $λ$ 的 apply 一样）。在 System F 中，类似于 λ 表达式，没有除了 $λ$、$Λ$ 和 $∀$ 以外的东西，如果给它暂时加入整数类型（例如$1: \text{Int}$），那么就可以给出这样的求值实例:

$$
\begin{aligned}
((Λ t. λ (x:t). x) \text{Int})1 & \Rightarrow \\\\
(λ (x:\text{Int}). x) 1 & \Rightarrow 1
\end{aligned}
$$

由于Hindey-Milner类型推理算法的存在，ML类语言中无须手动给出这样的 $\text{Int}$，函数的类型也会自动推导为带有全称量词的形式。虽然为了进行推理，某些在 System F 中合法的式子不能在 ML 中被构造，但搞清楚 System F 的类型规则对接下来的讨论是大有裨益的。

System F的类型规则分为四个规则：

1.
    $$
    {\frac{Γ , (x:t_1) ⊢ y:t_2}{Γ ⊢ (λ(x:t_1).y):t_1 → t_2}}_{}
    $$

2.
    $$
    \frac{Γ , A:\bold{type} ⊢ x:t_1 }{ Γ ⊢ (Λ A.x): ∀ A. t_1}
    $$

3.
    $$
    \frac{Γ ⊢ x:(t_1 → t_2), y:t_1}{Γ ⊢ (x\ y): t_2}
    $$

4.
    $$
    \frac{Γ ⊢ x:(∀A.t_1)\  \ \ \ (B:\bold{type})}{ Γ ⊢ (x\ B):[B/A]t_1}
    $$

从规则2.中可以看到，这里的 $A$ 是**任意**的类型，引入这个 $Λ$ 的条件是无论 $A$ 到底是什么类型，它都可以推出 $x$ 是 $t₁$ 类型。这样一来，在规则4.中，$B$ 也是一个**任意**的类型。可以不加证明地得到：

> **命题1.** 如果 $(ΛA.x)$ 是一个系统F中的封闭项，那么对于任意的类型 $t$，$((ΛA.x)\ t)$ 是一个系统F中的封闭项。

命题1.是任何声称自己实现了参数多态的系统所必须满足的命题。在ML类语言中，它表现为，一个类型为`'a -> b`（这里的`b`没有加引号，表示任何可能的类型，比如`'a`，这样一来就是`'a -> 'a`）的函数`f`，对于任何类型为`t`的合法ML表达式`x`，`f x`的类型都是`[t/'a]b`.

例如之前的`id`函数，`id 10`的类型就是`[int/'a]'a`，即`int`.

事实上，参数多态满足更强的条件--parametricity，鉴于比较复杂，这里暂不解释其含义。

## C++模板满足命题1.吗？

答案是很显然的：**不满足**。这也就是为什么C++模板不是参数多态的一个原因。

例如说，`id`函数可以写成：

```cpp
template <class T>
T id(T x) {
    return 10;
}
```

如果之前没有写过，你可能会惊讶地发现，这竟然是**不会报错**的！事实上，无论怎么写，它都不会报错：

```cpp
template <class T>
int id(T x) {
    return x;
}
```

它真正报错的位置，是在对`id`函数进行使用的时候：

```cpp
auto x = id("sdfsdf");

------------------------
.\test.cpp:5:10: error: cannot initialize return object of type 'const char *'
      with an rvalue of type 'int'
  return 10;
         ^~
.\test.cpp:9:14: note: in instantiation of function template specialization       
      'id<const char *>' requested here
    auto y = id("s");
             ^
1 error generated.
```

这诉说了一个无情的真相：C++ 编译器不会对模板函数进行类型检查，只会在**特化**的时候进行类型检查。而这使得命题1.无法成立，因为我们根本没有保证在`T`是一个类型，`x`是`T`类型的情况下，可以推出`id`函数体（也就是`return x`）的类型是`T`.

相反，ocaml 会无视泛型参数：

```ocaml
# let id (x: 'a) : int = x ;;
val id : int -> int = <fun>
```

```ocaml
# let id (x: 'a) : 'a = 10 ;;
val id : int -> int = <fun>
```

Haskell 会报错：

```haskell
Prelude> :{
Prelude| id :: a -> a
Prelude| id x = 10
Prelude| :}

<interactive>:6:8: error:
    • No instance for (Num a) arising from the literal ‘10’
      Possible fix:
        add (Num a) to the context of
          the type signature for:
            id :: forall a. a -> a
    • In the expression: 10
      In an equation for ‘id’: id x = 10
```

```haskell
Prelude> :{
Prelude| id :: a -> Int
Prelude| id x = x
Prelude| :}

<interactive>:12:8: error:
    • Couldn't match expected type ‘Int’ with actual type ‘a’
      ‘a’ is a rigid type variable bound by
        the type signature for:
          id :: forall a. a -> Int
        at <interactive>:11:1-14
    • In the expression: x
      In an equation for ‘id’: id x = x
    • Relevant bindings include
        x :: a (bound at <interactive>:12:4)
        id :: a -> Int (bound at <interactive>:12:1)
```

可见，C++ 并不是参数多态，它的模板系统和 ML 类语言中的泛型系统完全不在一个层次上。

## 类型类影响问题吗？

有人可能会说了，Haskell 的类型类不是也会出现“在使用的时候才报错”这一现象吗？比如说

```haskell
max :: Ord p => p -> p -> p
max x y = if y > x then y else x

data X = I | J

max I J -- 这里报错
-- error:
--    • No instance for (Ord X) arising from a use of ‘max’
--    • In the expression: max I J
--      In an equation for ‘it’: it = max I J
```

这个错误简单来说，就是自定义类型`X`没有实现`Ord`类型类，而`max`的**类型签名**中需要类型类`Ord`，所以报错。这和 C++ 模板仍然很不一样。根本性的区别是，在对`max I J`进行类型检查的时候，只需要`max :: Ord p => p -> p -> p`这一个信息就够了，不需要知道`max`的函数体是什么，更不需要“特化”这个函数，这仍是在类型系统中的推导。

## 真假泛型？性能更好？

虽然我并不是很喜欢 Java 语言，但是 Java 语言的泛型系统（在一般情况下）满足命题1.，它是真正的参数多态。这是因为 Java 泛型的前身--Generic Java（GJ）是由 Philip Wadler 等真正的编程语言专家设计的。

![GJ作者的合照](https://homepages.inf.ed.ac.uk/wadler/gj/gj-front-full.jpg)

有人说 Java 泛型采用了类型擦除，是“假泛型”，这显然是被c\++模板的设计影响了。事实上显然参数多态对应的参数类型更有资格把自己叫做“泛型”。c\++才是“假泛型”的例子。

c\++模板的设计使得 module 系统变成了一个困难的问题，到了 2021 年，99.9% 的c\++程序仍然在使用上古时代的头文件--这一彻头彻尾的垃圾--模拟 module 系统。

至于性能，所谓的“特化”在编程语言中有两个更强的概念可以替代。

+ 部分求值(partial evaluation)
+ 内联(inlining)

特化可以看作是一个部分求值的例子。F# 则可以通过字节码反编译回 C#，在[sharplab](https://sharplab.io)中，可以发现

```fsharp
let select x y = x

let select2 (x:int) (y:int) = select x y
```

这样的例子会得到：

```csharp
using System.Reflection;
using Microsoft.FSharp.Core;

[assembly: FSharpInterfaceDataVersion(2, 0, 0)]
[assembly: AssemblyVersion("0.0.0.0")]
[CompilationMapping(SourceConstructFlags.Module)]
public static class @_
{
    [CompilationArgumentCounts(new int[] {
        1,
        1
    })]
    public static a select<a, b>(a x, b y)
    {
        return x;
    }

    [CompilationArgumentCounts(new int[] {
        1,
        1
    })]
    public static int select2(int x, int y)
    {
        return x;
    }
}
namespace <StartupCode$_>
{
    internal static class $_
    {
    }
}
```

`select2`难道不是对`select`的“特化”吗？由此看来，C++ 模板的“性能更好”也是很难说清楚的问题。

## Concept: 能解决问题吗？

是否是参数多态，似乎只是一个学术问题。不是参数多态，主要导致的是使用一个参数化类型（parametric type，也就是含有∀的类型）时的潜在错误，pararmetricity 不成立，某些定理（比如`map f . map g = map (f . g)`）也不成立。

之前已经提到了，Haskell 也有会出错的问题。其他 ML 家族的语言，如果不算上类似 F# 的 object 系统的扩展，则基本没有出错的问题，但是需要传递更多参数或显式使用模块（module）系统。F# 的 `List.sortWith` 的类型就是 `(('a -> 'a -> int) -> 'a list -> 'a list)`，对于任何 `'a` 都不会出错，但是需要显式地传递一个 comparer. 

所以，不是参数多态，也许也不是特别大的问题。

但说到 C++，不是参数多态确实带来了一个相当严重的问题，那就是模板系统给出的报错信息常常令人无从下手。

例如这样的程序：

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

class A {};

int main() {
    std::vector<A> v;
    std::sort(v.begin(), v.end());
}
```

会产生 200 行，19kb 的报错：

```text
In file included from .\test.cpp:3:
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29910\include\algorithm:7361:32: error: no matching function for call to object of type 'std::less<>'
            if (_DEBUG_LT_PRED(_Pred, _Val, *_First)) { // found new earliest element, move to front
...
```

为了解决这个问题，C++20 引入了 concept，concept 借鉴了类型类和 Java 泛型的设计，可以使得我们能写出类似于之前 Haskell 的`max`那样的程序：

```cpp
#include <iostream>
#include <algorithm>
#include <concepts>

template <class T> requires std::equality_comparable<T>
bool max(T x, T y) {
    if (x > y) {
        return x;
    } else {
        return y;
    }
}
```

如果这时写出：

```cpp
class A{};

int main() {
    max(A(), A());
}
```

编译器会报错：

```text
source>: In function 'int main()':
<source>:17:8: error: no matching function for call to 'max(A, A)'
   17 |     max(A(), A());
      |     ~~~^~~~~~~~~~
<source>:6:6: note: candidate: 'template<class T>  requires  equality_comparable<T> bool max(T, T)'
    6 | bool max(T x, T y) {
      |      ^~~
<source>:6:6: note:   template argument deduction/substitution failed:
<source>:6:6: note: constraints not satisfied
In file included from /opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/compare:39,
                 from /opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/bits/stl_pair.h:65,
                 from /opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/bits/stl_algobase.h:64,
                 from /opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/bits/char_traits.h:39,
                 from /opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/ios:40,
                 from /opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/ostream:38,
                 from /opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/iostream:39,
                 from <source>:1:
/opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/concepts: In substitution of 'template<class T>  requires  equality_comparable<T> bool max(T, T) [with T = A]':
<source>:17:8:   required from here
/opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/concepts:280:15:   required for the satisfaction of '__weakly_eq_cmp_with<_Tp, _Tp>' [with _Tp = A]
/opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/concepts:290:13:   required for the satisfaction of 'equality_comparable<T>' [with T = A]
/opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/concepts:281:4:   in requirements with 'std::remove_reference_t<_Tp>& __t', 'std::remove_reference_t<_Up>& __u' [with _Up = A; _Tp = A]
/opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/concepts:282:17: note: the required expression '(__t == __u)' is invalid
  282 |           { __t == __u } -> __boolean_testable;
      |             ~~~~^~~~~~
/opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/concepts:283:17: note: the required expression '(__t != __u)' is invalid
  283 |           { __t != __u } -> __boolean_testable;
      |             ~~~~^~~~~~
/opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/concepts:284:17: note: the required expression '(__u == __t)' is invalid
  284 |           { __u == __t } -> __boolean_testable;
      |             ~~~~^~~~~~
/opt/compiler-explorer/gcc-11.2.0/include/C++/11.2.0/concepts:285:17: note: the required expression '(__u != __t)' is invalid
  285 |           { __u != __t } -> __boolean_testable;
      |             ~~~~^~~~~~
cc1plus: note: set '-fconcepts-diagnostics-depth=' to at least 2 for more detail
Compiler returned: 1
```

这里会直接表明A不满足`std::equality_comparable`的约束。虽然如此，噪音还是非常的多。更重要的是，对于库函数--这种我们不想研究源码的函数，效果还是一般：

```text
n file included from /opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/bits/stl_algobase.h:71,
                 from /opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/bits/char_traits.h:39,
                 from /opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/ios:40,
                 from /opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/ostream:38,
                 from /opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/iostream:39,
                 from <source>:1:
/opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/bits/predefined_ops.h: In instantiation of 'constexpr bool __gnu_cxx::__ops::_Iter_less_iter::operator()(_Iterator1, _Iterator2) const [with _Iterator1 = __gnu_cxx::__normal_iterator<A*, std::vector<A> >; _Iterator2 = __gnu_cxx::__normal_iterator<A*, std::vector<A> >]':
/opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/bits/stl_algo.h:1826:14:   required from 'constexpr void std::__insertion_sort(_RandomAccessIterator, _RandomAccessIterator, _Compare) [with _RandomAccessIterator = __gnu_cxx::__normal_iterator<A*, std::vector<A> >; _Compare = __gnu_cxx::__ops::_Iter_less_iter]'
/opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/bits/stl_algo.h:1866:25:   required from 'constexpr void std::__final_insertion_sort(_RandomAccessIterator, _RandomAccessIterator, _Compare) [with _RandomAccessIterator = __gnu_cxx::__normal_iterator<A*, std::vector<A> >; _Compare = __gnu_cxx::__ops::_Iter_less_iter]'
/opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/bits/stl_algo.h:1957:31:   required from 'constexpr void std::__sort(_RandomAccessIterator, _RandomAccessIterator, _Compare) [with _RandomAccessIterator = __gnu_cxx::__normal_iterator<A*, std::vector<A> >; _Compare = __gnu_cxx::__ops::_Iter_less_iter]'
/opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/bits/stl_algo.h:4842:18:   required from 'constexpr void std::sort(_RAIter, _RAIter) [with _RAIter = __gnu_cxx::__normal_iterator<A*, std::vector<A> >]'
<source>:10:14:   required from here
/opt/compiler-explorer/gcc-11.2.0/include/c++/11.2.0/bits/predefined_ops.h:45:23: error: no match for 'operator<' (operand types are 'A' and 'A')
   45 |       { return *__it1 < *__it2; }
      |                ~~~~~~~^~~~~~~~

剩下还有80多行
```

公允地说，C++ 的模板设计也带来了一些好处。虽然从理论上来说，编译器有一些通用的方法能够得到“特化”所带来的性能提升，但实践上的各种困难[^1]使得 C++ 模板在性能上确实有一些优势（这里指的是运行时性能，在编译性能上，C++ 模板是极其糟糕的）。

相比语言设计本身，我更不喜欢的是某些 C++ 程序员的无知，以及这种无知带来的傲慢态度。某些人动辄把 Java 的泛型叫做“假泛型”，认为只有像 C++ 模板一样的泛型实现才是“真泛型”。我真诚地建议，这些人应该认真阅读一下本文。

[^1]: 知乎上有一篇[特别好的文章](https://zhuanlan.zhihu.com/p/456388736)介绍了 Haskell 等语言在这个问题上的尴尬