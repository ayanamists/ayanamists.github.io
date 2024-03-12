---
title: 一个不大不小的陷阱 -- GMP 的 C++ 绑定
date: 2020-12-21 19:31:00
categories:
  - 编程实践
tags: 
  - C++
  - GC
---

## GNU MP库的C++绑定

GNU MP库是一个大整数和多精度浮点数的运算库。它本身是用C语言写成的，但也提供了C\++绑定。当用C\++写程序时，如果你不是自虐狂或者狂热的手动编译器变换爱好者，那么用C++绑定毫无疑问是更好的选择。

这是因为，C语言版本的绑定把所有操作都封装成了类似于汇编语言中的指令。比如说，如果要算一个大整数版本的`1+2`，那么应该这么写：

```c
mpz_t a, b, c;
mpz_init_set_ui(a, 1);
mpz_init_set_ui(b, 2);
mpz_add(c, a, b);
mpz_clear(a);
mpz_clear(b);
mpz_clear(c);
```

说实话，这还不如直接写汇编来的简洁：

```asm
mov $1, %eax
mov $2, %ebx
add %ebx, %eax
mov %ecx, %eax
```

这种情况的根本原因是，C语言中没有方便的手段来进行内存资源管理和快速结构构造（当然，新的标准也有了一些），导致C语言虽然可以实现*表达式求值模型*，但无法方便地实现*自定义类型的表达式求值模型*。

而表达式求值模型和寄存器机模型之间的变换就是编译的本质，用C语言这样写代码，相当于是在自己进行部分编译器进行的变换（比如ANF）。所以，喜欢这样写代码的人要么是喜欢写汇编，要么是喜欢自己进行手动编译器变换。

C\++绑定这时堪称救世主，在不得不使用C/C\++的场合（比如我校的《现代密码学实验》课程），用C++绑定可以避免这种尴尬：

```c++
const mpz_class a {1}, b {2};
const auto c = a + b;
```

这段代码可以执行和上面C语言代码**完全相同**的行为。这是因为C++相比C有了几个优良特性，其中最优秀的的当属所谓的RAII，也就是**R**esource **A**cquisition **I**s **I**nitialization，获取资源即初始化。在这里和运行栈一起配合，简单来说说就是构造函数和析构函数的配合，使得栈上的对象在构造时（手动引入一个绑定时）获取资源，在析构时（退出当前作用域时自动析构）释放资源。对内存这种资源来说，这样一来我们好似在使用一种『有GC的语言』，无须关心任何内存问题一样。

自然而然的一个问题是，栈式RAII真的能够代替GC吗？通过下文对GMP的解说，想必读者能够给出自己的答案。

## 一个奇怪的问题

已经说过，我使用GNU MP库的主要目的是为了进行密码学实验。我们的密码学实验中有一个计算DLP（离散对数）的问题，规模非常大，运行速度是比较重要的因素。由此我必须使用GNU MP这种速度有保证的库。然而，在实验中，我遇到了一个非常奇怪的问题，那就是有时候某些代码常常出现不正确的结果，而我反复检查代码也不能发现问题的由来。更严重的是，这些问题像幽灵一样，有时出现有时不出现，出现时的结果有时是*不相同*的。

对此，我第一反应是出了一些内存问题。可我立刻就否定了这种想法。GNU MP这种被很多人使用的库，一般来说不会出现这种恶性问题。可是我的代码中只有简单的计算，类似于：

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

在经过了一番艰险的探索后，我确定了一个『最小问题结构』。『最小问题结构』是说，触发这个问题的最简单、行数最少的代码。它是：

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

在我的计算机上，这段代码会给出非常惊人的结果：

```shell
➜  gmp_error git:(master) ✗ g++ test.cpp -o a -g -lgmp -O0 -lgmpxx
➜  gmp_error git:(master) ✗ ./a                                   
94361021124304
94361021124336%
```

$1+2=94361021124304$，还是 $1+2=94361021124336$？

如此简单的代码却产生了如此诡异的错误，真是怪哉！

## GNU MP库的设计

要破解这个谜团，我们应该从另一个怪现象下手，那即是：

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

这段代码竟然毫无问题？恐怕读者难以相信这个事实，然而它就是真正的现实：

```shell
➜  gmp_error git:(master) ✗ g++ test.cpp -o a -g -lgmp -O0 -lgmpxx
➜  gmp_error git:(master) ✗ ./a                                   
3
3%
```

这样一来问题就很明朗了。`auto`这个关键字究竟会将`a`推导为什么类型？用IDE或者c++filt查看，答案更是让人一头雾水：

```c++
const __gmp_expr<mpz_t, __gmp_binary_expr<mpz_class, mpz_class, __gmp_binary_plus>> a
```

这类型是什么？看来必须到`gmpxx.h`这个文件里寻找答案了。

在`gmpxx.h`中，我们会看到，`mpz_class`实际上是`mpz_expr<mpz_t, mpz_t>`:

```c++
/**************** mpz_class -- wrapper for mpz_t ****************/

template <> // line 1572
class __gmp_expr<mpz_t, mpz_t>{ ... }; 

typedef __gmp_expr<mpz_t, mpz_t> mpz_class; // line 1756
```

那么，这个`__gmp_expr`高阶类型（理论上来说这确实相当于高阶类型）恐怕还有一些其他的特化，果不其然，这个文件中还定义了很多`__gmp_expr`的特化，比如说，我们前面看到的`a`，实际上的类型是：

```c++
template <class T, class Op>
class __gmp_expr
<T, __gmp_binary_expr<__gmp_expr<T, T>, __gmp_expr<T, T>, Op> >
```

我们观察一下这个类的构造函数：

```c++
__gmp_expr(const val1_type &val1, const val2_type &val2)
    : expr(val1, val2) { }
```

`expr`是类的成员变量，它声明为：

```c++
__gmp_binary_expr<val1_type, val2_type, Op> expr;
```

这个`__gmp_binary_expr`又是何方神圣呢？它定义如下：

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

这有点令人一头雾水，定义这样的一个只有构造函数的类似乎没有什么特别的意义，我们需要寻找到用到它的函数。之前提到过，如果右值的类型是`mpz_class`，那么就不会产生问题。从，`mpz_expr<..>`变成`mpz_class`，一定发生了一个类型转换。这个类型转换的函数在哪里呢？再回到`mpz_class`的定义当中：

```c++
template <class T>
__gmp_expr(const __gmp_expr<mpz_t, T> &expr)
{ mpz_init(mp); __gmp_set_expr(mp, expr); }
template <class T, class U>
explicit __gmp_expr(const __gmp_expr<T, U> &expr)
{ mpz_init(mp); __gmp_set_expr(mp, expr); }
```

这个函数毫无疑问是在将`__gmp_expr<...>`转换为`mpz_class`. 那么`__gmp_set_expr`又是在做什么呢？

查看其定义：

```c++
template <class T>
inline void __gmp_set_expr(mpz_ptr z, const __gmp_expr<mpz_t, T> &expr)
{
  expr.eval(z);
}
```

嗯？这个`eval`函数看起来是`__gmp_expr<T ...>`中定义的，我们再查看一下刚才的定义：

```c++
void eval(typename __gmp_resolve_expr<T>::ptr_type p) const
{ Op::eval(p, expr.val1.__get_mp(), expr.val2.__get_mp()); }
```

转发到了`Op::eval`这个函数上。之前类型的`Op`是`__gmp_binary_plus`，它的`eval`函数是如何定义的呢？

```c++
struct __gmp_binary_plus
{
  static void eval(mpz_ptr z, mpz_srcptr w, mpz_srcptr v)
  { mpz_add(z, w, v); }
```

这实在是十分亲切，我们终于搞明白了这一套组合拳到底是在做些什么事情。

首先，`__gmp_expr< ... >`相当于一个语法树，它记录了所有的操作信息。当这个类型的值被转换为`mpz_class`时，进行求值，求完的值被放到了转换后的绑定中。

可是，这究竟有何意义？在我看来，这样的代码没有简化任何逻辑。C++编译器完全可以保证不产生多余的复制，实际上，如此复杂的构造和直白地写一个类并重载运算符的效果几乎是完全一致的。

唯一的好处，就是当变量使用`auto`而不是`mpz_class`时，变量本身是语法树而不是值，只有当需要这个表达式的值时（也就是进行类型转换时）才进行求值。这就是所谓的『惰性求值』。

我很难理解在数值计算任务上进行惰性求值究竟有什么好处。惰性求值最大的好处就是不会算出不必要的值，比如说：

```scheme
(define (f) (f))
(define (g t1 t2) (t2))

(g (f) 1) ;在scheme中，无限循环
```

```haskell
f = f
g t1 t2 = t2
g f 1 --在haskell中，这会得到 1
```

可是，在这样的数值计算任务中，我们一般不会进行任何多余的计算。惰性求值本身不能简化必要计算，从性能上来说，这毫无优势。

而且，这设计会产生刚才的严重错误。这是因为，每个`__gmp_binary_expr`保存的实际上是两个变量的`const`引用，而从根本上来说，`const`引用是无法捕获一个右值的。调用

```c++
__gmp_binary_expr(const T &v1, const U &v2) : val1(v1), val2(v2) { }
```

只会把指向`v1`的指针赋值给`val1`，把指向`v2`的指针赋值给`val2`。

回过头来再看这句代码：

```c++
const auto a = mpz_class { 1 } + mpz_class { 2 };
...
```

它实际上会变成：

```c++
mpz_class temp1 {1}, temp2 {2};
a = temp1 + temp2;
~temp1(); ~temp2();
...
```

当析构函数执行之后，`a`这棵语法树中节点所指向的目标已经被完全析构，访问这些对象的代码全部是错误的。换句话说，`a`合法的时光仅存在于当前语句执行完、下一条语句还未执行的那一瞬间而已。

## 解决问题

要解决问题，有两个方法：

+ 修改`gmpxx.h`. 
+ 所有的声明全部使用`mpz_class`而不是`auto`.

不过，即使修改这个文件，`const&`不能捕获右值的问题仍然是无法解决的。

把`__gmp_binary_expr`改为值语义怎么样？换句话说，我们让`val1`，`val2`不再是`const &T`和`const &U`而是真正的`T`和`U`. 

这可以无痛地解决`const auto a = mpz_class { 1 } + mpz_class { 2 };`的问题。因为`mpz_class{1}`和`mpz_class{2}`都是『右值』，或者说是『X值』，有『右值引用』这个方法可以无痛地交接资源。实际上，如果只是解决加法的问题，我们只需要修改几个地方即可：

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

自定义一个`__gmp_binary_expr`的特化，处理两个都是`mpz_class`的场合。

这样就可以使得上述代码得到正确的`3`.

然而，先不说都修正完的工作量，如此修正我们必然会遇到一个问题：*如果传入的是左值，则无法进行『无痛』的移动，要进行复制，这是不利于性能的*。

怎样解决这个问题呢？答案是（至少我）解决不了。

## GC与RAII

上面的问题在有GC的语言中，简单来说不算问题。即使是在python这样的语言中，不断地得到一个资源的绑定并不会产生任何复制：

```python
a = [1, 2, 3, 4]
b = a
c = b
```

当然，这是因为`a`、`b`、`c`实际上都是指向同一个对象的，类似于`const &`.

可是，有GC语言里的`const &`可以完美地解决『不能捕获右值』的问题：

```python
class A:
    def __init__(self, arr):
        self.arr = arr
        
a = A([1,2,3,4])
```

从根本上来说，RAII不能使得同一个栈上的对象被两个绑定所同时『拥有』，栈上的对象被消除的规则是严格的作用域规则，不能出现『从栈上借对象』这样的情况。而有GC语言由于『对象』和『对象所拥有的资源』都在堆上，甚至是一体的，所以不会出现这个问题。

这样来看，RAII是不能替代GC的。当然，Rust等语言可能可以通过一些别的办法来解决这个问题。不过，我们可以下这个结论：在C++中，RAII的能力终究是有限的。