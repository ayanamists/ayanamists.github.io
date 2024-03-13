---
layout: post
title: The Fundamental Difference Between C++ Templates and Parametric Polymorphism
date: 2021-09-14 15:53:47
tags: 
  - C++
  - Programming Language
categories:
  - Programming Language
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

## C++ Templates and Parametric Polymorphism

Templates are a feature of the C++ language, similar to:

```cpp
template <class T>
T id(T x) {
    return x;
}
```

Here, `T` can be any type. Many people have noticed its similarity to parametric polymorphism, and some even directly regard the template mechanism as a kind of parametric polymorphism. Such as this [ppt](https://www.cs.bham.ac.uk/~hxt/2015/c-plus-plus/templates-slides.pdf) and this [webpage](https://catonmat.net/cpp-polymorphism).

However, in this blog post, I will explain that **regardless of whether C++ templates are a good language feature, they are fundamentally different from parametric polymorphism**.

## What is Parametric Polymorphism

The history of parametric polymorphism can be traced back to a lecture note ([Fundamental Concepts in Programming Languages](https://link.springer.com/article/10.1023%2FA%3A1010000313106)). Although the lecture note mainly describes the CPL language, in fact, this section should describe the MAP function of LISP:

> Parametric polymorphism is more regular and may be illustrated by an example. Suppose f is a function whose argument is of type α and whose results is of β (so that the type of f might be written α ⇒ β), and that L is a list whose elements are all of type α (so that the type of L is α list). We can imagine a function, say Map, which applies f in turn to each member of L and makes a list of the results. Thus Map[f,L] will produce a β list. We would like Map to work on all types of list provided f was a suitable function, so that Map would have to be polymorphic. However its polymorphism is of a particularly simple parametric type which could be written

$$
(α → β , α\ \text{list}) → β \ \text{list}
$$

The ML language in 1975 implemented this kind of polymorphism at the level of the type system. In ML-like languages, a function's type can contain type variables, such as the `id` function below:

```ocaml
let id (x:'a) = x
```

The type of the `id` function is `'a -> 'a`, where `'a` is a type variable that can match **any** type, so we get

```ocaml
id 1
id "ssdfs"
id 1.23
```

All are legal expressions. The type system of ML-like languages can be seen as a system that restricts and extends System F. System F can be seen as an extension of λ expressions. In System F, each bound variable of a λ expression must be given a type, and this type is introduced by $Λ$ (uppercase λ).

Specifically, the most basic λ expression $λx.x$ in System F should be written as:

$$
Λt.λ (x:t).x
$$

And the type of this expression is

$$
∀t.t → t
$$

This large $Λ$ introduces a type variable that needs to be explicitly applied when used (just like the apply for small $λ$). In System F, similar to λ expressions, there is nothing other than $λ$, $Λ$ and $∀$. If you temporarily add an integer type (for example, $1: \text{Int}$), then you can give such an evaluation instance:

$$
\begin{aligned}
((Λ t. λ (x:t). x) \text{Int})1 & \Rightarrow \\\\
(λ (x:\text{Int}). x) 1 & \Rightarrow 1
\end{aligned}
$$

Due to the existence of the Hindley-Milner type inference algorithm, there is no need to manually give such $\text{Int}$ in ML-like languages, and the type of the function will also be automatically inferred to the form with a universal quantifier. Although in order to make inferences, some legal expressions in System F cannot be constructed in ML, but it is very beneficial to understand the type rules of System F for the following discussion.

The type rules of System F are divided into four rules:

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

From Rule 2, we can see that $A$ here is an **arbitrary** type, and the condition for introducing this $Λ$ is that no matter what type $A$ is, it can be inferred that $x$ is of type $t₁$. In this way, in Rule 4, $B$ is also an **arbitrary** type. It can be obtained without proof:

> **Proposition 1.** If $(ΛA.x)$ is a closed term in System F, then for any type $t$, $((ΛA.x)\ t)$ is a closed term in System F.

Proposition 1 is a proposition that any system claiming to have implemented parametric polymorphism must satisfy. In ML-like languages, it is manifested as, a function `f` of type `'a -> b` (here `b` is not quoted, representing any possible type, such as `'a`, so it is `'a -> 'a`), for any legal ML expression `x` of type `t`, the type of `f x` is `[t/'a]b`.

For example, the previous `id` function, the type of `id 10` is `[int/'a]'a`, that is, `int`.

In fact, parametric polymorphism satisfies a stronger condition--parametricity, which is quite complicated, so its meaning is not explained here.

## Does C++ template satisfy Proposition 1?

The answer is obviously: **No**. This is also a reason why C++ templates are not parametric polymorphism.

For example, the `id` function can be written as:

```cpp
template <class T>
T id(T x) {
    return 10;
}
```

If you haven't written it before, you may be surprised to find that this is **not an error**! In fact, no matter how you write it, it won't be an error:

```cpp
template <class T>
int id(T x) {
    return x;
}
```

The real error occurs when using the `id` function:

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

This tells a ruthless truth: The C++ compiler does not perform type checking on template functions, but only performs type checking when **specializing**. This makes Proposition 1 impossible, because we have no guarantee that when `T` is a type and `x` is of type `T`, the type of the `id` function body (that is, `return x`) can be inferred to be `T`.

On the contrary, ocaml will ignore generic parameters:

```ocaml
# let id (x: 'a) : int = x ;;
val id : int -> int = <fun>
```

```ocaml
# let id (x: 'a) : 'a = 10 ;;
val id : int -> int = <fun>
```

Haskell will throw an error:

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

As you can see, C++ is not parameter polymorphic, its template system and the generic system in ML-like languages are not on the same level.

## Does the type class affect the problem?

Some people might say, doesn't Haskell's type class also have the phenomenon of "only reporting errors when used"? For example

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

This error, simply put, is that the custom type `X` does not implement the `Ord` type class, and the **type signature** of `max` requires the `Ord` type class, so it throws an error. This is still very different from C++ templates. The fundamental difference is that when type checking `max I J`, only `max :: Ord p => p -> p -> p` this information is enough, there is no need to know what the function body of `max` is, let alone "specialize" this function, this is still a deduction in the type system.

## True or false generics? Better performance?

Although I don't particularly like the Java language, the generic system of the Java language (in general cases) satisfies Proposition 1., it is true parameter polymorphism. This is because the predecessor of Java generics--Generic Java (GJ) was designed by real programming language experts such as Philip Wadler.

![Photo of GJ authors](https://homepages.inf.ed.ac.uk/wadler/gj/gj-front-full.jpg)

Some people say that Java generics use type erasure, which is "fake generics", which is obviously influenced by the design of c++ templates. In fact, it is clear that the parameter type corresponding to parameter polymorphism is more qualified to call itself "generic". c++ is an example of "fake generics".

The design of c++ templates makes the module system a difficult problem. By 2021, 99.9% of c++ programs are still using ancient header files--this complete garbage--to simulate the module system.

As for performance, the so-called "specialization" has two stronger concepts in programming languages that can replace it.

+ Partial Evaluation
+ Inlining

Specialization can be seen as an example of partial evaluation. F# can be decompiled back to C# through bytecode, in [sharplab](https://sharplab.io), you can find

```fsharp
let select x y = x

let select2 (x:int) (y:int) = select x y
```

Such an example will yield:

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

Isn't `select2` a "specialization" of `select`? From this point of view, it's hard to say whether C++ templates are "better performing".

## Concept: Can it solve the problem?

Whether it is parametric polymorphism seems to be an academic question. Non-parametric polymorphism mainly leads to potential errors when using a parametric type (i.e., a type containing ∀). If parametricity does not hold, some theorems (such as `map f . map g = map (f . g)`) also do not hold.

As mentioned before, Haskell also has problems. Other ML family languages, if not counting extensions like F#'s object system, basically have no problems, but they need to pass more parameters or explicitly use the module system. The type of F#'s `List.sortWith` is `(('a -> 'a -> int) -> 'a list -> 'a list)`, it won't go wrong for any `'a`, but it needs to explicitly pass a comparer.

So, non-parametric polymorphism may not be a big problem.

But when it comes to C++, non-parametric polymorphism does bring a serious problem, that is, the error messages given by the template system are often baffling.

For example, a program like this:

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

Will produce 200 lines, 19kb of errors:

```text
In file included from .\test.cpp:3:
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29910\include\algorithm:7361:32: error: no matching function for call to object of type 'std::less<>'
            if (_DEBUG_LT_PRED(_Pred, _Val, *_First)) { // found new earliest element, move to front
...
```

To solve this problem, C++20 introduced the concept, which draws on the design of type classes and Java generics, allowing us to write programs similar to the previous Haskell `max`:

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

If you write at this time:

```cpp
class A{};

int main() {
    max(A(), A());
}
```

The compiler will report an error:

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

This will directly indicate that A does not satisfy the `std::equality_comparable` constraint. Even so, the noise is still very much. More importantly, for library functions--functions we don't want to study the source code, the effect is still general:

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

To be fair, the design of C++ templates also brings some benefits. Although theoretically, the compiler has some general methods to get the performance improvement brought by "specialization", various practical difficulties[^1] make C++ templates indeed have some advantages in performance (here refers to runtime performance, in terms of compilation performance, C++ templates are extremely bad).

Compared with language design itself, what I dislike more is the ignorance of some C++ programmers and the arrogant attitude brought by this ignorance. Some people often call Java's generics "fake generics", thinking that only generics implementations like C++ templates are "real generics". I sincerely suggest that these people should read this article carefully.

[^1]: There is a [particularly good article](https://zhuanlan.zhihu.com/p/456388736) on Zhihu that introduces the embarrassment of languages like Haskell on this issue.
