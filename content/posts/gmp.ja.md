---
title: 中程度の罠 -- GMPのC++バインディング
date: 2020-12-21 19:31:00
categories:
  - プログラミング実践
tags: 
  - C++
  - GC
---

{{<  admonition "warning" >}}
この記事はChatGPTによって中国語から翻訳されたもので、いくつかの誤りが含まれているかもしれません。不正確な部分があればご了承ください。
{{< /admonition >}}

## GNU MPライブラリのC++バインディング

GNU MPライブラリは、大きな整数と多精度浮動小数点数の計算ライブラリです。それ自体はC言語で書かれていますが、C++バインディングも提供しています。C++でプログラムを書くとき、あなたが自己虐待狂や手動コンパイラ変換の熱狂的な愛好家でないなら、C++バインディングを使うことは間違いなく良い選択です。

これは、C言語バージョンのバインディングがすべての操作をアセンブリ言語の命令のようにカプセル化しているからです。例えば、大きな整数バージョンの `1+2` を計算したい場合、次のように書くべきです：

```c
mpz_t a, b, c;
mpz_init_set_ui(a, 1);
mpz_init_set_ui(b, 2);
mpz_add(c, a, b);
mpz_clear(a);
mpz_clear(b);
mpz_clear(c);
```

正直に言うと、これは直接アセンブリを書くよりも簡潔ではありません：

```asm
mov $1, %eax
mov $2, %ebx
add %ebx, %eax
mov %ecx, %eax
```

この状況の根本的な原因は、C言語にはメモリリソース管理や高速構造構築の便利な手段がない（もちろん、新しい標準にはいくつかある）ため、C言語は*式評価モデル*を実装することができますが、*カスタム型の式評価モデル*を便利に実装することはできません。

式評価モデルとレジスタマシンモデルの間の変換はコンパイルの本質であり、このようにC言語でコードを書くことは、部分的にコンパイラが行う変換（例えばANF）を自分で行うことに相当します。だから、このようにコードを書くのが好きな人は、アセンブリを書くのが好きな人か、手動でコンパイラ変換を行うのが好きな人です。

C++バインディングはこの時点で救世主となり、C/C++を使用しなければならない場合（例えば私たちの学校の「現代暗号学実験」のコース）、C++バインディングを使用することでこのような困難を避けることができます：

```c++
const mpz_class a {1}, b {2};
const auto c = a + b;
```

このコードは、上記のC言語コードと**完全に同じ**動作を実行します。これは、C++がCに比べていくつかの優れた特性を持っているためで、その中でも最も優れているのはRAII、つまり**R**esource **A**cquisition **I**s **I**nitialization、リソースの取得は初期化です。ここでは、実行スタックと一緒に動作し、簡単に言えば、コンストラクタとデストラクタの組み合わせにより、スタック上のオブジェクトが構築されるとき（手動でバインディングを導入するとき）リソースを取得し、破棄されるとき（現在のスコープを退出するときに自動的に破棄される）リソースを解放します。メモリというリソースに対しては、これにより我々は『GCのある言語』を使用しているかのように、どんなメモリ問題も気にせずに済むようになります。

自然に生じる一つの問題は、スタック型RAIIが本当にGCを置き換えることができるのかということです。以下のGMPの説明を通じて、読者の皆さんが自分自身の答えを出すことができるでしょう。

## 奇妙な問題

既に述べたように、私がGNU MPライブラリを使用する主な目的は暗号学の実験を行うためです。私たちの暗号学の実験では、DLP（離散対数）の計算問題があり、その規模は非常に大きく、実行速度は重要な要素です。そのため、私は速度が保証されているGNU MPのようなライブラリを使用しなければなりません。しかし、実験中に、私は非常に奇妙な問題に遭遇しました。それは、時々一部のコードが頻繁に不正確な結果を出すというもので、私が何度もコードをチェックしても問題の原因を見つけることができませんでした。さらに深刻なのは、これらの問題が幽霊のように、時々現れたり消えたりし、現れたときの結果が時々*異なる*ということです。

これに対して、私の最初の反応は何かメモリの問題があるのではないかということでした。しかし、私はすぐにこの考えを否定しました。GNU MPのような多くの人々に使用されているライブラリは、一般的にはこのような悪性の問題を起こすことはありません。しかし、私のコードには単純な計算しか含まれていませんでした。例えば：

```c++
/*
 * Pohlig-Hellman algorithm for Group of prime power order
 */
mpz_class
pohligHellmanP(const mpz_class& g, const mpz_class& h,
               const mpz_class& pn, const mpz_class& en,
               const mpz_class& p) {
    const auto y = fastPow(g, Pow(pn, en - 1), p);
    assert (fastPow(y, pn, p) == 1);
    mpz_class x{0};
    for (auto i = 0; i < en; ++i) {
        auto hi = fastPow(Inverse(fastPow(g, x, p), p) * h,
                          Pow(pn, en - 1 - i), p);
        auto di = pDlp(y, hi, pn, p);
        x = x + Pow(pn, i) * di;
    }
    return x;
}
```

一連の困難な探求の後、私は『最小問題構造』を確定しました。『最小問題構造』とは、この問題を引き起こす最も単純で、行数が最も少ないコードを指します。それは次のようなものです：

```c++
mpz_class nothing() {
  const auto a = mpz_class { 1 } + mpz_class { 2 };
  std::cout << a << std::endl;
  return a;
}

int main() {
  std::cout << nothing();
}
```

私のコンピュータ上では、このコードは非常に驚くべき結果を出します：

```shell
➜  gmp_error git:(master) ✗ g++ test.cpp -o a -g -lgmp -O0 -lgmpxx
➜  gmp_error git:(master) ✗ ./a                                   
94361021124304
94361021124336%
```

$1+2=94361021124304$、それとも $1+2=94361021124336$？

これほど単純なコードがこれほど奇妙なエラーを生じさせるなんて、本当に奇妙です！

## GNU MPライブラリの設計

この謎を解くためには、別の奇妙な現象から手をつけるべきです。それは次のようなものです：

```c++
mpz_class nothing() {
  const mpz_class a = mpz_class { 1 } + mpz_class { 2 };
  std::cout << a << std::endl;
  return a;
}

int main() {
  std::cout << nothing();
}
```

このコードは全く問題がない？読者の皆さんはこの事実を信じるのが難しいかもしれませんが、それは現実です：

```shell
➜  gmp_error git:(master) ✗ g++ test.cpp -o a -g -lgmp -O0 -lgmpxx
➜  gmp_error git:(master) ✗ ./a                                   
3
3%
```

それにより問題は明確になります。`auto`というキーワードは`a`を何の型に推論するのでしょうか？IDEやc++filtで確認すると、答えはますます混乱します：

```c++
const __gmp_expr<mpz_t, __gmp_binary_expr<mpz_class, mpz_class, __gmp_binary_plus>> a
```

この型は何ですか？答えを見つけるためには`gmpxx.h`というファイルを見る必要があるようです。

`gmpxx.h`を見ると、`mpz_class`は実際には`mpz_expr<mpz_t, mpz_t>`であることがわかります：

```c++
/**************** mpz_class -- wrapper for mpz_t ****************/

template <> // line 1572
class __gmp_expr<mpz_t, mpz_t>{ ... }; 

typedef __gmp_expr<mpz_t, mpz_t> mpz_class; // line 1756
```

では、この`__gmp_expr`という高階型（理論的には確かに高階型に相当します）は他にも特化があるのでしょうか？確かに、このファイルには`__gmp_expr`の多くの特化が定義されています。例えば、先ほど見た`a`の実際の型は以下のようになります：

```c++
template <class T, class Op>
class __gmp_expr
<T, __gmp_binary_expr<__gmp_expr<T, T>, __gmp_expr<T, T>, Op> >
```

このクラスのコンストラクタを見てみましょう：

```c++
__gmp_expr(const val1_type &val1, const val2_type &val2)
    : expr(val1, val2) { }
```

`expr`はクラスのメンバ変数で、以下のように宣言されています：

```c++
__gmp_binary_expr<val1_type, val2_type, Op> expr;
```

この`__gmp_binary_expr`とは何者なのでしょうか？その定義は以下の通りです：

```c++
template <class T, class U, class Op>
struct __gmp_binary_expr
{
  typename __gmp_resolve_ref<T>::ref_type val1;
  typename __gmp_resolve_ref<U>::ref_type val2;

  __gmp_binary_expr(const T &v1, const U &v2) : val1(v1), val2(v2) { }
private:
  __gmp_binary_expr();
};
```

これは少し混乱を招くかもしれません。このようなコンストラクタしか持たないクラスを定義する特別な意味は何でしょうか？それを使用する関数を見つける必要があります。以前に述べたように、右辺の型が`mpz_class`であれば問題は発生しません。`mpz_expr<..>`から`mpz_class`になるとき、型変換が行われています。この型変換関数はどこにあるのでしょうか？再び`mpz_class`の定義に戻ります：

```c++
template <class T>
__gmp_expr(const __gmp_expr<mpz_t, T> &expr)
{ mpz_init(mp); __gmp_set_expr(mp, expr); }
template <class T, class U>
explicit __gmp_expr(const __gmp_expr<T, U> &expr)
{ mpz_init(mp); __gmp_set_expr(mp, expr); }
```

この関数は間違いなく`__gmp_expr<...>`を`mpz_class`に変換しています。では、`__gmp_set_expr`は何をしているのでしょうか？

その定義を見てみましょう：

```c++
template <class T>
inline void __gmp_set_expr(mpz_ptr z, const __gmp_expr<mpz_t, T> &expr)
{
  expr.eval(z);
}
```

え？この`eval`関数は`__gmp_expr<T ...>`で定義されているようです、先ほどの定義を再度確認してみましょう：

```c++
void eval(typename __gmp_resolve_expr<T>::ptr_type p) const
{ Op::eval(p, expr.val1.__get_mp(), expr.val2.__get_mp()); }
```

これは`Op::eval`関数に転送されています。以前の型の`Op`は`__gmp_binary_plus`で、その`eval`関数はどのように定義されているのでしょうか？

```c++
struct __gmp_binary_plus
{
  static void eval(mpz_ptr z, mpz_srcptr w, mpz_srcptr v)
  { mpz_add(z, w, v); }
```

これは非常に親切で、ついにこの一連のコンボが何をしているのかを理解しました。

まず、`__gmp_expr< ... >`は構文木のようなもので、すべての操作情報を記録しています。この型の値が`mpz_class`に変換されるとき、評価が行われ、評価後の値が変換後のバインドに格納されます。

しかし、これが何を意味するのでしょうか？私の見解では、このようなコードはロジックを何も簡略化していません。C++コンパイラは余分なコピーを生成しないことを完全に保証できます。実際、このように複雑な構造と直接クラスを書いて演算子をオーバーロードする効果はほぼ同じです。

唯一の利点は、変数が`auto`を使用している場合、変数自体が値ではなく構文木であり、その式の値が必要になる（つまり型変換が行われる）まで評価されないことです。これはいわゆる「遅延評価」です。

数値計算タスクで遅延評価を行う利点が何なのか、私には理解できません。遅延評価の最大の利点は、不必要な値を計算しないことです。例えば：

```scheme
(define (f) (f))
(define (g t1 t2) (t2))

(g (f) 1) ;schemeでは、無限ループ
```

```haskell
f = f
g t1 t2 = t2
g f 1 --haskellでは、これは 1 を返します
```

しかし、このような数値計算タスクでは、通常は余計な計算は行わない。遅延評価自体は必要な計算を簡略化できず、パフォーマンス上の利点はありません。

さらに、この設計は先ほどの深刻なエラーを引き起こします。これは、各`__gmp_binary_expr`が実際に保存しているのは2つの変数の`const`参照であり、基本的に`const`参照は右辺値をキャプチャできないからです。呼び出し

```c++
__gmp_binary_expr(const T &v1, const U &v2) : val1(v1), val2(v2) { }
```

は、`v1`へのポインタを`val1`に、`v2`へのポインタを`val2`に割り当てるだけです。

このコードを再度見てみましょう：

```c++
const auto a = mpz_class { 1 } + mpz_class { 2 };
...
```

実際には次のようになります：

```c++
mpz_class temp1 {1}, temp2 {2};
a = temp1 + temp2;
~temp1(); ~temp2();
...
```

デストラクタが実行された後、`a`の構文木のノードが指す対象は完全にデストラクトされ、これらのオブジェクトにアクセスするコードはすべてエラーです。言い換えれば、`a`が有効なのは現在の文が完了し、次の文がまだ実行されていない瞬間だけです。

## 問題の解決

問題を解決するには2つの方法があります：

+ `gmpxx.h`を修正する。 
+ すべての宣言を`mpz_class`で行い、`auto`を使用しない。

しかし、このファイルを修正しても、`const&`が右辺値をキャプチャできない問題は依然として解決できません。

`__gmp_binary_expr`を値セマンティクスに変更するのはどうでしょうか？つまり、`val1`と`val2`を`const &T`と`const &U`ではなく、実際の`T`と`U`にするということです。

これは `const auto a = mpz_class { 1 } + mpz_class { 2 };` の問題を痛みなく解決できます。なぜなら、`mpz_class{1}`と`mpz_class{2}`はどちらも「右値」、つまり「X値」で、「右値参照」によってリソースを痛みなく引き継ぐことができるからです。実際、加法の問題を解決するだけなら、いくつかの箇所を修正するだけで済みます：

```c++
/* 修改这个宏，使得加法有右值引用的版本 */
#define __GMPP_DEFINE_BINARY_FUNCTION(fun, eval_fun)                   \
                                                                       \
template <class T, class U, class V, class W>                          \
inline __gmp_expr<typename __gmp_resolve_expr<T, V>::value_type,       \
__gmp_binary_expr<__gmp_expr<T, U>, __gmp_expr<V, W>, eval_fun> >      \
fun(const __gmp_expr<T, U> &expr1, const __gmp_expr<V, W> &expr2)      \
{                                                                      \
  return __gmp_expr<typename __gmp_resolve_expr<T, V>::value_type,     \
     __gmp_binary_expr<__gmp_expr<T, U>, __gmp_expr<V, W>, eval_fun> > \
    (expr1, expr2);                                                    \
}                                                                      \
template <class T, class U, class V, class W>                          \
inline __gmp_expr<typename __gmp_resolve_expr<T, V>::value_type,       \
__gmp_binary_expr<__gmp_expr<T, U>, __gmp_expr<V, W>, eval_fun> >      \
fun(__gmp_expr<T, U> &&expr1, __gmp_expr<V, W> &&expr2)                \
{                                                                      \
  return __gmp_expr<typename __gmp_resolve_expr<T, V>::value_type,     \
     __gmp_binary_expr<__gmp_expr<T, U>, __gmp_expr<V, W>, eval_fun> > \
    (std::move(expr1), std::move(expr2));                              \
}
```

```c++
/* 修改这个类，使得构造函数有右值引用的版本 */
template <class T, class Op>
class __gmp_expr
<T, __gmp_binary_expr<__gmp_expr<T, T>, __gmp_expr<T, T>, Op> >
{
private:
  typedef __gmp_expr<T, T> val1_type;
  typedef __gmp_expr<T, T> val2_type;

  __gmp_binary_expr<val1_type, val2_type, Op> expr;
public:
  __gmp_expr(const val1_type &val1, const val2_type &val2)
    : expr(val1, val2) { } 
  __gmp_expr(val1_type &&val1, val2_type &&val2) // 新加入的构造函数
    : expr(std::move(val1), std::move(val2)) { }
```

```c++
template <class Op>
struct __gmp_binary_expr<mpz_class, mpz_class, Op>
{
  mpz_class val1;
  mpz_class val2;
  __gmp_binary_expr(const mpz_class &v1, const mpz_class &v2) 
    : val1(v1), val2(v2) { }
  __gmp_binary_expr(mpz_class &&v1, mpz_class &&v2) 
    : val1(std::move(v1)), val2(std::move(v2)) { }
private:
  __gmp_binary_expr();
};
```

`__gmp_binary_expr`の特化を自分で定義し、両方が`mpz_class`の場合を処理します。

これにより、上記のコードは正しく`3`を得ることができます。

しかし、修正が完了するまでの作業量はさておき、このように修正すると必ず一つの問題に直面します：*左値が渡された場合、痛みなく移動することはできず、コピーを行う必要があり、これはパフォーマンスにとって不利です*。

この問題をどのように解決するのでしょうか？答えは（少なくとも私には）解決できません。

## GCとRAII

上記の問題は、GCがある言語では、簡単に言えば問題ではありません。たとえばPythonのような言語では、リソースのバインドを繰り返しても、コピーは発生しません：

```python
a = [1, 2, 3, 4]
b = a
c = b
```

もちろん、これは`a`、`b`、`c`が実際には同じオブジェクトを指しているためで、`const &`のようなものです。

しかし、GCがある言語では、`const &`は「右値をキャプチャできない」という問題を完全に解決できます：

```python
class A:
    def __init__(self, arr):
        self.arr = arr
        
a = A([1,2,3,4])
```

根本的には、RAIIでは同じスタック上のオブジェクトを2つのバインドが同時に「所有」することはできず、スタック上のオブジェクトが消去されるルールは厳格なスコープルールであり、「スタックからオブジェクトを借りる」という状況は発生しません。一方、GCがある言語では、「オブジェクト」と「オブジェクトが所有するリソース」が堆上にあり、さらに一体化しているため、この問題は発生しません。

このように見ると、RAIIはGCを置き換えることはできません。もちろん、Rustなどの言語では、他の方法でこの問題を解決できるかもしれません。しかし、結論として、C++では、RAIIの能力は結局のところ限定的です。
