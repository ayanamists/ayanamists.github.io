---
title: A Not Too Big, Not Too Small Trap -- GMP's C++ Binding
date: 2020-12-21 19:31:00
categories:
  - Programming Practice
tags: 
  - C++
  - GC
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

## C++ Binding of the GNU MP Library

The GNU MP library is a library for large integer and multi-precision floating-point arithmetic. It is written in C, but also provides C++ bindings. When writing programs in C++, unless you are a masochist or a fanatical manual compiler transformation enthusiast, using the C++ binding is undoubtedly a better choice.

This is because the C language version of the binding encapsulates all operations into instructions similar to those in assembly language. For example, if you want to calculate a large integer version of `1+2`, you should write it like this:

```c
mpz_t a, b, c;
mpz_init_set_ui(a, 1);
mpz_init_set_ui(b, 2);
mpz_add(c, a, b);
mpz_clear(a);
mpz_clear(b);
mpz_clear(c);
```

Honestly, this is not as concise as writing assembly directly:

```asm
mov $1, %eax
mov $2, %ebx
add %ebx, %eax
mov %ecx, %eax
```

The fundamental reason for this situation is that there is no convenient means in C to manage memory resources and construct structures quickly (of course, some have been added in the new standards), which makes C able to implement the *expression evaluation model*, but it is not convenient to implement the *custom type expression evaluation model*.

The transformation between the expression evaluation model and the register machine model is the essence of compilation. Writing code in C in this way is equivalent to performing some transformations that the compiler does (such as ANF). Therefore, people who like to write code in this way either like to write assembly, or they like to manually perform compiler transformations.

The C++ binding is a savior at this time. In situations where C/C++ must be used (such as the "Modern Cryptography Experiment" course in our school), using the C++ binding can avoid this embarrassment:

```c++
const mpz_class a {1}, b {2};
const auto c = a + b;
```

This code can perform **exactly the same** behavior as the above C language code. This is because C++ has several excellent features compared to C, the best of which is the so-called RAII, that is, **R**esource **A**cquisition **I**s **I**nitialization. Here, in conjunction with the runtime stack, in simple terms, it is the cooperation of the constructor and destructor, so that objects on the stack acquire resources when constructed (manually introducing a binding), and release resources when destructed (automatically destructed when exiting the current scope). For resources like memory, it seems that we are using a "language with GC", and we don't need to worry about any memory issues.

A natural question arises, can stack-based RAII really replace GC? Through the following explanation of GMP, readers should be able to come up with their own answers.

## A Strange Problem

As mentioned before, my main purpose of using the GNU MP library is to conduct cryptographic experiments. In our cryptographic experiments, there is a problem of calculating DLP (Discrete Logarithm Problem), which is very large in scale, and the running speed is a relatively important factor. Therefore, I must use the GNU MP library, which guarantees speed. However, in the experiment, I encountered a very strange problem, that is, sometimes some codes often produce incorrect results, and I can't find the source of the problem even after repeated checks. What's worse, these problems are like ghosts, sometimes they appear and sometimes they don't, and the results when they appear are sometimes *different*.

My first reaction to this was that there might be some memory issues. But I immediately denied this idea. The GNU MP library, which is used by many people, generally will not have such a malignant problem. But in my code, there are only simple calculations, similar to:

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

After a difficult exploration, I determined a "minimum problem structure". The "minimum problem structure" means the simplest and least line of code that triggers this problem. It is:

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

On my computer, this code will give a very surprising result:

```shell
➜  gmp_error git:(master) ✗ g++ test.cpp -o a -g -lgmp -O0 -lgmpxx
➜  gmp_error git:(master) ✗ ./a                                   
94361021124304
94361021124336%
```

$1+2=94361021124304$ or $1+2=94361021124336$?

Such simple code has produced such a weird error, it's really strange!

## The Design of the GNU MP Library

To crack this mystery, we should start from another strange phenomenon, that is:

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

This code has no problem at all? Readers may find it hard to believe this fact, but it is the reality:

```shell
➜  gmp_error git:(master) ✗ g++ test.cpp -o a -g -lgmp -O0 -lgmpxx
➜  gmp_error git:(master) ✗ ./a                                   
3
3%
```

This makes the problem very clear. What type will the keyword `auto` deduce `a` to be? Using IDE or c++filt to check, the answer is even more confusing:

```c++
const __gmp_expr<mpz_t, __gmp_binary_expr<mpz_class, mpz_class, __gmp_binary_plus>> a
```

What type is this? It seems that we must find the answer in the `gmpxx.h` file.

In `gmpxx.h`, we will see that `mpz_class` is actually `mpz_expr<mpz_t, mpz_t>`:

```c++
/**************** mpz_class -- wrapper for mpz_t ****************/

template <> // line 1572
class __gmp_expr<mpz_t, mpz_t>{ ... }; 

typedef __gmp_expr<mpz_t, mpz_t> mpz_class; // line 1756
```

So, this `__gmp_expr` high-order type (theoretically, this is indeed equivalent to a high-order type) may have some other specializations. As expected, this file also defines many specializations of `__gmp_expr`. For example, the `a` we saw earlier, the actual type is:

```c++
template <class T, class Op>
class __gmp_expr
<T, __gmp_binary_expr<__gmp_expr<T, T>, __gmp_expr<T, T>, Op> >
```

Let's observe the constructor of this class:

```c++
__gmp_expr(const val1_type &val1, const val2_type &val2)
    : expr(val1, val2) { }
```

`expr` is a member variable of the class, it is declared as:

```c++
__gmp_binary_expr<val1_type, val2_type, Op> expr;
```

What is this `__gmp_binary_expr`? It is defined as follows:

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

This is a bit confusing, defining such a class with only a constructor seems to have no special meaning, we need to find the function that uses it. As mentioned earlier, if the type of the right value is `mpz_class`, there will be no problem. From `mpz_expr<..>` to `mpz_class`, a type conversion must have occurred. Where is this type conversion function? Let's go back to the definition of `mpz_class`:

```c++
template <class T>
__gmp_expr(const __gmp_expr<mpz_t, T> &expr)
{ mpz_init(mp); __gmp_set_expr(mp, expr); }
template <class T, class U>
explicit __gmp_expr(const __gmp_expr<T, U> &expr)
{ mpz_init(mp); __gmp_set_expr(mp, expr); }
```

This function is undoubtedly converting `__gmp_expr<...>` to `mpz_class`. So what is `__gmp_set_expr` doing?

Let's look at its definition:

```c++
template <class T>
inline void __gmp_set_expr(mpz_ptr z, const __gmp_expr<mpz_t, T> &expr)
{
  expr.eval(z);
}
```

Hmm? This `eval` function seems to be defined in `__gmp_expr<T ...>`, let's look at the definition we just saw:

```c++
void eval(typename __gmp_resolve_expr<T>::ptr_type p) const
{ Op::eval(p, expr.val1.__get_mp(), expr.val2.__get_mp()); }
```

It is forwarded to the `Op::eval` function. The previous type of `Op` was `__gmp_binary_plus`, how is its `eval` function defined?

```c++
struct __gmp_binary_plus
{
  static void eval(mpz_ptr z, mpz_srcptr w, mpz_srcptr v)
  { mpz_add(z, w, v); }
```

This is really familiar, we finally figured out what this combination of punches is doing.

First of all, `__gmp_expr< ... >` is like a syntax tree, it records all operation information. When the value of this type is converted to `mpz_class`, it is evaluated, and the evaluated value is put into the binding after conversion.

But what is the point of this? In my opinion, such code does not simplify any logic. The C++ compiler can fully guarantee that no extra copies are generated. In fact, such a complex construction and simply writing a class and overloading the operator have almost the same effect.

The only advantage is that when the variable uses `auto` instead of `mpz_class`, the variable itself is a syntax tree rather than a value, and it is only evaluated when the value of this expression is needed (that is, during type conversion). This is the so-called "lazy evaluation".

I find it hard to understand what the benefits of lazy evaluation are in numerical computation tasks. The biggest advantage of lazy evaluation is that it does not calculate unnecessary values, such as:

```scheme
(define (f) (f))
(define (g t1 t2) (t2))

(g (f) 1) ;In scheme, infinite loop
```

```haskell
f = f
g t1 t2 = t2
g f 1 --In Haskell, this will get 1
```

However, in such numerical computation tasks, we generally do not perform any redundant calculations. Lazy evaluation itself cannot simplify necessary calculations, and there is no advantage in terms of performance.

Moreover, this design will cause serious errors as just mentioned. This is because each `__gmp_binary_expr` actually saves the `const` reference of two variables, and fundamentally, `const` references cannot capture a right value. Calling

```c++
__gmp_binary_expr(const T &v1, const U &v2) : val1(v1), val2(v2) { }
```

will only assign the pointer pointing to `v1` to `val1`, and the pointer pointing to `v2` to `val2`.

Looking back at this line of code:

```c++
const auto a = mpz_class { 1 } + mpz_class { 2 };
...
```

It will actually become:

```c++
mpz_class temp1 {1}, temp2 {2};
a = temp1 + temp2;
~temp1(); ~temp2();
...
```

After the destructor is executed, the targets pointed to by the nodes in the syntax tree `a` have been completely destructed, and all code accessing these objects is wrong. In other words, the legal time of `a` only exists in the moment after the current statement is executed and before the next statement is executed.

## Solving the problem

There are two ways to solve the problem:

+ Modify `gmpxx.h`. 
+ All declarations use `mpz_class` instead of `auto`.

However, even if this file is modified, the problem that `const&` cannot capture right values is still unsolvable.

How about changing `__gmp_binary_expr` to value semantics? In other words, we let `val1` and `val2` be not `const &T` and `const &U` but real `T` and `U`.

This can painlessly solve the problem of `const auto a = mpz_class { 1 } + mpz_class { 2 };`. Because `mpz_class{1}` and `mpz_class{2}` are both "rvalues", or "X values", the method of "rvalue reference" can painlessly transfer resources. In fact, if we just want to solve the addition problem, we only need to modify a few places:

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

Customize a specialization of `__gmp_binary_expr` to handle the situation where both are `mpz_class`.

This way, the above code can get the correct `3`.

However, let's not talk about the workload of all corrections, such a correction will inevitably encounter a problem: *If the input is a lvalue, it cannot be "painlessly" moved, and copying is required, which is not conducive to performance*.

How to solve this problem? The answer is that (at least I) can't solve it.

## GC and RAII

The above problem is not a problem in languages with GC. Even in a language like python, constantly getting a binding of a resource will not produce any copy:

```python
a = [1, 2, 3, 4]
b = a
c = b
```

Of course, this is because `a`, `b`, and `c` are actually pointing to the same object, similar to `const &`.

However, `const &` in languages with GC can perfectly solve the problem of "cannot capture rvalues":

```python
class A:
    def __init__(self, arr):
        self.arr = arr
        
a = A([1,2,3,4])
```

Fundamentally speaking, RAII cannot allow the same object on the stack to be "owned" by two bindings at the same time. The rule of eliminating objects on the stack is a strict scope rule, and there can be no situation of "borrowing objects from the stack". In languages with GC, because both "objects" and "resources owned by objects" are on the heap, or even integrated, this problem will not occur.

In this way, RAII cannot replace GC. Of course, languages like Rust may be able to solve this problem through some other methods. However, we can draw this conclusion: In C++, the ability of RAII is ultimately limited.
