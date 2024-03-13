---
layout: post
title: C++ テンプレートとパラメータ多態性の根本的な違い
date: 2021-09-14 15:53:47
tags: 
  - C++
  - プログラミング言語
categories:
  - プログラミング言語
---

{{<  admonition "warning" >}}
この記事はChatGPTによって中国語から翻訳されたもので、いくつかの誤りが含まれているかもしれません。不正確な部分があればご了承ください。
{{< /admonition >}}

## C++ テンプレートとパラメータ多態性

テンプレート(template)は C++ 言語の特性で、以下のようなものです：

```cpp
template <class T>
T id(T x) {
    return x;
}
```

ここでの `T` は任意の型を指すことができます。多くの人々はこれがパラメータ多態性(parametric polymorphism)に似ていることに気づき、さらにはテンプレートメカニズムをパラメータ多態性の一種と直接見なす人もいます。例えば、この[ppt](https://www.cs.bham.ac.uk/~hxt/2015/c-plus-plus/templates-slides.pdf)やこの[webページ](https://catonmat.net/cpp-polymorphism)。

しかし、このブログ記事では、**C++ テンプレートが良い言語特性であるかどうかは別として、それがパラメータ多態性とは根本的に異なること**を説明します。

## パラメータ多態性とは何か

パラメータ多態性の歴史は、講義ノート([Fundamental Concepts in Programming Languages](https://link.springer.com/article/10.1023%2FA%3A1010000313106))に遡る必要があります。この講義ノートは主に CPL 言語を説明していますが、実際には以下の部分は LISP の MAP 関数を説明しているはずです：

> パラメータ多態性はより規則的で、例を挙げて説明することができます。f が型 α の引数を持ち、結果が β の関数（つまり、f の型は α ⇒ β と書くことができます）であり、L がすべての要素が型 α のリスト（つまり、L の型は α list）であるとします。f を順番に L の各メンバーに適用し、結果のリストを作る関数、つまり Map を想像することができます。したがって、Map[f,L] は β list を生成します。私たちは Map が f が適切な関数である限り、すべての型のリストで動作することを望んでいます。したがって、Map は多態性を持つ必要があります。しかし、その多態性は特に単純なパラメータ型であり、書き方は

$$
(α → β , α\ \text{list}) → β \ \text{list}
$$

1975年のML言語は、このような多態性を型システムのレベルで実現しました。ML系の言語では、関数の型に型変数を含めることができます。例えば、以下の`id`関数：

```ocaml
let id (x:'a) = x
```

`id`関数の型は`'a -> 'a`で、ここでの`'a`は型変数で、**任意**の型にマッチすることができます。そのため、

```ocaml
id 1
id "ssdfs"
id 1.23
```

はすべて合法な式です。ML系の言語の型システムは、System Fに制約と拡張を加えたものと見ることができます。System Fはλ式の拡張と見ることができ、System Fでは、λ式の各束縛変数には型を指定する必要があり、この型は$Λ$（大文字のλ）で導入されます。

具体的には、最も基本的なλ式 $λx.x$ はSystem Fでは以下のように書くべきです：

$$
Λt.λ (x:t).x
$$

そして、この式の型は

$$
∀t.t → t
$$

となります。この大きな $Λ$ で導入された型変数は、使用時に明示的なapply（小さな $λ$ のapplyと同様）が必要です。System Fでは、λ式のように、$λ$、$Λ$、$∀$以外のものは存在せず、一時的に整数型（例えば$1: \text{Int}$）を追加すると、以下のような評価例を提供できます：

$$
\begin{aligned}
((Λ t. λ (x:t). x) \text{Int})1 & \Rightarrow \\\\
(λ (x:\text{Int}). x) 1 & \Rightarrow 1
\end{aligned}
$$

Hindey-Milner型推論アルゴリズムの存在により、ML系の言語では、このような $\text{Int}$ を手動で提供する必要はありません。関数の型は全称量子を含む形式で自動的に推論されます。推論を行うために、System Fで合法な式がMLで構築できない場合がありますが、System Fの型規則を理解することは、今後の議論に大いに役立ちます。

System Fの型規則は以下の4つの規則に分けられます：

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

ルール2.からわかるように、ここでの $A$ は**任意**の型で、この $Λ$ の条件は $A$ がどのような型であっても $x$ が $t₁$ 型であると推測できることです。これにより、ルール4.では、$B$ も**任意**の型です。証明なしで次のことが得られます：

> **命題1.** もし $(ΛA.x)$ がシステムFの閉じた項であるなら、任意の型 $t$ に対して、$((ΛA.x)\ t)$ はシステムFの閉じた項である。

命題1.は、パラメータ多態性を実装したと主張する任意のシステムが満たすべき命題です。ML系の言語では、これは型が`'a -> b`（ここでの`b`は引用符がなく、任意の可能な型、例えば`'a`を表し、結果として`'a -> 'a`になる）の関数`f`に対して、型が`t`の任意の合法なML式`x`に対して、`f x`の型は`[t/'a]b`であるという形で表現されます。

例えば、以前の`id`関数では、`id 10`の型は`[int/'a]'a`、つまり`int`です。

実際、パラメータ多態性はより強い条件、つまりパラメトリシティを満たしますが、これは比較的複雑なので、ここではその意味を説明しません。

## C++テンプレートは命題1.を満たしますか？

答えは明らかに：**満たしません**。これがC++テンプレートがパラメータ多態性でない一つの理由です。

例えば、`id`関数は次のように書くことができます：

```cpp
template <class T>
T id(T x) {
    return 10;
}
```

もし前に書いていなければ、これが**エラーにならない**ことに驚くかもしれません！実際、どのように書いてもエラーになりません：

```cpp
template <class T>
int id(T x) {
    return x;
}
```

実際にエラーになるのは、`id`関数を使用するときです：

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

これは無情な真実を告げています：C++ コンパイラはテンプレート関数の型チェックを行わず、**特殊化**の時にのみ型チェックを行います。これにより、命題1.は成り立たず、`T`が型で、`x`が`T`型である場合に、`id`関数の本体（つまり`return x`）の型が`T`であると推測できる保証がないからです。

逆に、ocamlはジェネリックパラメータを無視します：

```ocaml
# let id (x: 'a) : int = x ;;
val id : int -> int = <fun>
```

```ocaml
# let id (x: 'a) : 'a = 10 ;;
val id : int -> int = <fun>
```

Haskellはエラーを報告します：

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

見ての通り、C++はパラメータ多相性ではなく、そのテンプレートシステムはML系言語のジェネリックシステムとは全く異なるレベルにあります。

## 型クラスは問題に影響しますか？

Haskellの型クラスも「使用時にエラーが報告される」という現象が起こるのではないかと言う人もいるかもしれません。例えば、

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

このエラーは、簡単に言えば、カスタム型`X`が`Ord`型クラスを実装しておらず、`max`の**型シグネチャ**が型クラス`Ord`を必要とするため、エラーが発生します。これはC++のテンプレートとは全く異なります。根本的な違いは、`max I J`の型チェックを行う際には、`max :: Ord p => p -> p -> p`という情報だけで十分であり、`max`の関数本体が何であるかを知る必要はなく、ましてやこの関数を「特化」する必要はありません。これは型システム内での推論です。

## 真偽のジェネリック？パフォーマンスは良い？

私はJava言語をあまり好んでいませんが、Java言語のジェネリックシステム（一般的な状況下で）は命題1を満たしており、それは真のパラメータ多相性です。これは、Javaのジェネリックの前身であるGeneric Java（GJ）が、Philip Wadlerなどの真のプログラミング言語の専門家によって設計されたからです。

![GJの作者たちの集合写真](https://homepages.inf.ed.ac.uk/wadler/gj/gj-front-full.jpg)

Javaのジェネリックは型消去を採用しており、「偽のジェネリック」であると言う人もいますが、これは明らかにC++のテンプレートの設計に影響を受けています。実際には、パラメータ多相性に対応するパラメータ型が「ジェネリック」と呼ぶ資格があるのは明らかです。C++は「偽のジェネリック」の例です。

C++のテンプレートの設計は、モジュールシステムを困難な問題にしてしまいました。2021年現在、99.9%のC++プログラムは依然として古代のヘッダーファイル--これは完全なゴミ--を使ってモジュールシステムを模倣しています。

パフォーマンスについては、「特化」という概念は、プログラミング言語において2つのより強力な概念に置き換えることができます。

+ 部分評価（partial evaluation）
+ インライン化（inlining）

特化は部分評価の一例と見なすことができます。F#はバイトコードを逆コンパイルしてC#に戻すことができ、[sharplab](https://sharplab.io)では、

```fsharp
let select x y = x

let select2 (x:int) (y:int) = select x y
```

このような例が次のようになります：

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

`select2`は`select`の「特化」ではないのでしょうか？したがって、C++テンプレートの「パフォーマンスが良い」という主張は明確ではありません。

## コンセプト：問題を解決できますか？

パラメータ多態性があるかどうかは、学術的な問題のように思えます。パラメータ多態性がないと、主にパラメトリック型（parametric type、つまり∀を含む型）を使用する際の潜在的なエラーが発生し、パラメトリシティが成り立たないと、一部の定理（例えば`map f . map g = map (f . g)`）も成り立ちません。

以前にも触れたように、Haskellにもエラーが発生する問題があります。他のML系の言語では、F#のようなオブジェクトシステムの拡張を除けば、基本的にエラーは発生しませんが、より多くのパラメータを渡す必要があったり、モジュール（module）システムを明示的に使用する必要があります。F#の`List.sortWith`の型は`(('a -> 'a -> int) -> 'a list -> 'a list)`で、任意の`'a`に対してエラーは発生しませんが、comparerを明示的に渡す必要があります。

したがって、パラメータ多態性がないということは、それほど大きな問題ではないかもしれません。

しかし、C++について言えば、パラメータ多態性がないことは、テンプレートシステムが出すエラーメッセージが非常に扱いにくいという、かなり深刻な問題を引き起こします。

例えば、次のようなプログラム：

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

は200行、19kbのエラーを生成します：

```text
In file included from .\test.cpp:3:
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29910\include\algorithm:7361:32: error: no matching function for call to object of type 'std::less<>'
            if (_DEBUG_LT_PRED(_Pred, _Val, *_First)) { // found new earliest element, move to front
...
```

この問題を解決するために、C++20はconceptを導入しました。conceptは型クラスとJavaのジェネリクスの設計を取り入れており、以前のHaskellの`max`のようなプログラムを書くことができます：

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

この時に以下のように書くと：

```cpp
class A{};

int main() {
    max(A(), A());
}
```

コンパイラはエラーを報告します：

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

ここでは、Aが`std::equality_comparable`の制約を満たしていないことが直接示されています。それにもかかわらず、ノイズは非常に多いです。さらに重要なのは、ソースコードを調査したくないライブラリ関数に対しては、効果が一般的であることです：

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

公平に言えば、C++のテンプレート設計もいくつかの利点をもたらしています。理論的には、コンパイラには"特化"がもたらすパフォーマンス向上を得るための一般的な方法がありますが、実際の困難[^1]により、C++のテンプレートはパフォーマンス面で確かにいくつかの優位性を持っています（ここで言うパフォーマンスとは実行時のパフォーマンスであり、コンパイルパフォーマンスについては、C++のテンプレートは非常に悪いです）。

言語設計自体よりも、私がより好きでないのは、一部のC++プログラマーの無知と、その無知がもたらす傲慢な態度です。一部の人々は、Javaのジェネリクスを"偽ジェネリクス"と呼び、C++のテンプレートのようなジェネリクスの実装だけが"真のジェネリクス"であると考えています。私は心から、これらの人々にこの記事を真剣に読むことをお勧めします。

[^1]: 知乎には、Haskellなどの言語がこの問題に直面している困難を紹介する[非常に良い記事](https://zhuanlan.zhihu.com/p/456388736)があります。
