---
title: Understand Hindley-Milner Type System in Ten Minutes
author: ayanamists
tags: 
  - Type Inference
  - Programming Language
date: 2021-11-25
categories:
  - Academics
math: true
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}
## Type Inference

### What is Type Inference

Programmers are lazy creatures. Often, we don't want to verbosely specify types:

```fsharp
let t = 1 + 1
let f x = x + 1
```

The types of the three variables `t`, `f`, and `x` here are very clear. Of course, more often, it is better to explicitly specify the type.

There are also times when it is not easy to specify the type, for example:

```fsharp
let s f g x = f x (g x)
let k x y = x
let i = s k k
let ϕ = s (k (s i)) k
```

What is the type of ϕ?

The type of ϕ can be calculated, but it is very troublesome. The natural idea is to find an algorithm to calculate it instead of the human brain.

In fact, if you send this code to the haskell or f# repl, they will tell you that the type of ϕ is `p -> (p -> q) -> q`. The algorithm they use is the Hindley-Milner type inference algorithm.

### Type Inference Algorithm

In a nutshell, the type inference algorithm is to solve equations. The process of establishing equations is as follows:

1. Each function parameter and each function call return value are represented by a type variable `xᵢ`.
2. Establish equations when calling functions. The function call `f x` can establish equations with the following rules:

$$
\frac{f:a \rightarrow b\quad x:t_1 \quad (f\ x):t_2}{t_1 \rightarrow t_2 = a \rightarrow b}
$$

For example, such a function:

```fsharp
let f a b = a + b
```

Can establish such equations:

$$
\begin{aligned}
a:\\ &t_1 \\\\
b:\\ &t_2 \\\\
f:\\ &t_3 \\\\
a + b:\\ &t_4 \\\\
(+):\\ & \mathsf{int} \rightarrow \mathsf{int} \rightarrow \mathsf{int} \\\\
t_1 \rightarrow t_2 \rightarrow t₄ =&\ \mathsf{int} \rightarrow \mathsf{int} \rightarrow \mathsf{int} \\\\
t_3 =&\\ t_1 \rightarrow t_2 \rightarrow t_4
\end{aligned}
$$

By solving the two equations below, we can determine the type of each variable. The question now is, how do we solve such equations?

### Unification

Unification is an algorithm from automated theorem proving that can be used to solve the above equations. It is backed by a series of theories about substitution. Due to space limitations, we will not discuss these theories here, but will introduce the most natural algorithm[^1] in a very short space.

First of all, the above equations will involve at most two forms of terms. One is the type variable `xᵢ`, and the other is the arrow `xᵢ → xⱼ`. In fact, the arrow `xᵢ→ xⱼ` can be regarded as `f(xᵢ, xⱼ)`, so we have two forms:

1. Type variable.
2. The form of `f(v₀, v₁, v₂, ...)`. The above `int` can be regarded as `int()`

The algorithm is as follows:

Traverse each equation:

1. Perform a self-loop check. If both sides appear
    $$
    x_i = \cdots x_i \cdots
    $$
  (For example $x_i = f(x_i)$ ), then the algorithm fails.

2. If the form of the equation is:
$$
\begin{aligned}
  x_i &= \star \\\\
  \star &= x_i
\end{aligned}
$$  
   Then no matter what $\star$ is, replace $x_i$ in *equations that have not been traversed yet* with $\star$.

3. If it is not case 1., that is
   $$
   f(x_0, x_1, x_2, \cdots) = g(y_0, y_1, y_2, \cdots)
   $$
   Then,
   * If $f ≠ g$, the algorithm fails.
   * If $f = g$, add $n$ ( $n$ is the number of parameters of f ) new equations to be traversed $x_i = y_i$.

[^1]: The implementation given at the end of the text uses an algorithm based on the union-find set

## Let Polymorphism

Now we have a type inference algorithm. Consider such a function:

```fsharp
let f x = x
```

The above inference algorithm will infer `f:x₀ → x₀`, where there is a free variable `x₀`. Using the type notation of System F, "generalize" such functions:

$$
\frac{f:\psi(x_0)}{f:\forall x_0.\psi(x₀)}
$$

In this way, the type of `f` becomes a "polymorphic type", which can be "instantiated":

$$
\frac{f:\forall x.\psi(x)}{f:\psi(a)}
$$

A troublesome question is: when should generalization and instantiation be performed?

The natural idea is to generalize outside of lambda expressions like `fun x -> ...`, and instantiate when referencing a generalized variable. E.g.

```fsharp
// The following are all type-correct items
(λ x. x) 10
(λ x. x) true
```

The problem with this is that only λ expressions can be of "polymorphic type", and variables inside lambda can still only be of a single type.

```fsharp
(λf.(λ t. f true)(f 0))(λx.x)
```

Taking the above expression as an example, even if `λx.x` is of polymorphic type, `f` cannot be inferred as polymorphic type. In fact, `f` is inferred as `int → x₀` in `f 0` and `bool → x₀` in `f true`. This will cause a type error.

Recall that in F# or ocaml, the method for `f` to be truly assigned to generics is:

```fsharp
let f x = x in
    let t = f 0 in f true
```

In untyped or simple type λ expressions, the above two pieces of code are completely equivalent, `let` is just syntactic sugar. But in ML languages, `let` is not syntactic sugar. In fact, in ML languages, the time to "generalize" is exactly at the `let` binding!

That is to say:

1. In the expression `let var = e₁ in e₂`, the type of `e₁` is generalized. The method of generalization is:
    * The free variables in the current environment $Γ$ are $F(Γ)$
    * The free variables of $e_1$ are $F(e_1)$
    * For each free variable $x_i$ in $F(e_1) - F(Γ)$, $e_1:t$ becomes $e_1:∀x_i.t$
2. Whenever a generic variable $e:t_1$ is referenced from the current environment $Γ$, instantiate $e:t_1$ with a new type variable until $t_1$ no longer contains $∀$

And this is the basic idea of the Hindley-Milner algorithm. For specific implementations, there are algorithm J and algorithm W, please refer to Wikipedia or Milner's [original paper](https://www.sciencedirect.com/science/article/pii/0022000078900144?via%3Dihub). [My implementation](https://github.com/ayanamists/TypeInf) uses alogrithm J.

## Value restriction

If you really tried the previous code:

```fsharp
let s f g x = f x (g x)
let k x y = x
let i = s k k
let ϕ = s (k (s i)) k
```

You will find that in Haskell, this is no problem:

![1](https://pic.imgdb.cn/item/619f4b162ab3f51d911a8f2f.jpg)

But in F#, this will report an error:

![2](https://pic.imgdb.cn/item/619f4b552ab3f51d911ab740.jpg)

Why should it report an error? This has to start from a more tricky program. F# has mutable types:

```fsharp
let mutable x = 10
x <- 20
printfn "%A" x
```

If given such a program:

```fsharp
let mutable x = []
x <- [3]
List.map (fun x -> x = true) x
```

Then the first line will report an error. However, suppose the first line does not report an error, will the second line report an error? We need to study it carefully.

First, the type of `<-` is hidden in F#, but F# has a set of [deprecated](https://aka.ms/fsharp-refcell-ops) operators `!`, `:=` and `ref`, their types are:

```fsharp
ref  : 'a -> 'a ref
(!)  : 'a ref -> 'a
(:=) : 'a ref -> 'a -> unit
```

The above program is written as:

```fsharp
let x = ref []
x := [3]
List.map (fun x -> x = true) x
```

When `x` is referenced in the second line, the type of `x` is `'t list ref`, that is, `∀t.ref[list[t]]`, it should be instantiated as `ref[list[b]]`, so two equations are obtained:

$$
\begin{aligned}
\mathsf{ref}[\mathsf{list}[b]] =& \mathsf{ref}[a] \\\\
a =& \mathsf{list}[\mathsf{int}]
\end{aligned}
$$

The above equation gets $b = \mathsf{int}, a = \mathsf{list}[\mathsf{int}]$, **this will not report an error**. And the bigger problem is that after the type check of the second line, `b` indeed becomes `int`, but this is just the **instantiated** `x`, and the `x` in $Γ$, the type has always been `'t list ref`, that is to say, **the third line will not report an error anyway!**

This is very bad because it introduces unsoundness into the type system. The third line will undoubtedly result in a runtime error.

To solve this problem, ANDREW K. WRIGHT[^2] introduced value restriction. The meaning of value restriction is simple: only when `e₁` of `let var = e₁ in e₂` is a syntactic value, generalize `e₁`. If `e₁` is not a syntactic value and it has free variables not in $\Gamma$, then report an error.

Syntactic values in ML languages are constants, variables, λ expressions, and type constructors. Of course, a value of type `ref[a]` cannot be considered a syntactic value. In this way, the problem is obviously solved, because the above error is essentially due to the occurrence of side effects, and evaluating a syntactic value in ML cannot cause side effects.

This solution is over-approximate, and many correct programs (such as the one mentioned before) will also be rejected. Haskell does not have this problem because it does not have side effects. This is also a major reason why Haskell can use a lot of function combinations similar to `f . g`, but ML cannot.

[^2]: Simple imperative polymorphism, ANDREW K. WRIGHT

## Position of quantifiers?

In System F, quantifiers can appear anywhere in a type. For example, $∀x.(x → ∀y.y)$. In fact, although System F cannot construct the Y combinator (contrary to strong normalization), it can encode structures similar to untyped lambda calculus $(x x)$:

$$
λ(x:∀α.α→α).(x (∀α.α→α))x
$$

The type of this λ expression is $(∀α.α→α)→(∀α.α→α)$. The type of its first argument determines that it can only act on the `id` function `Λα.λ(x:α).x`. The form of applying it to `id` is written in untyped λ expression as:

$$
(λx.xx)(λx.x)
$$

Without a doubt, it will be β-reduced to $λx.x$.

From this, it can be seen that in System F, the type of the parameters of the λ expression can contain "polymorphic types". So why couldn't our previous type inference algorithm infer its polymorphic type?

This is because our previous inference algorithm can be regarded as the type inference algorithm of "simple type λ expression". And the HM type system is an extension of the simple type λ expression.

At this point, a natural question arises: Can System F perform similar inference? Or say, if we write this:

$$
λ(x:t_1).(x\ t_2)x
$$

Is there an algorithm that can get the values of $t_1$ and $t_2$ through this expression?

In 1985, Hans-J. Boehm's paper *PARTIAL POLYMORPHIC TYPE INFERENCE IS UNDECIDABLE* told us that if the `fix` combinator is added, then the above problem is undecidable. In 1995, J.B.Wells's paper *Typability and type checking in System F are equivalent and undecidable* further told us that if explicit $t₁$ and $t₂$ are not given, and only the final type is required to be checked, then this problem is still undecidable.

Therefore, the HM type system can be seen as a variant of two kinds of λ expressions:

* If it is seen as a variant of simple type λ expressions, then the HM type system has added the form of `let var = e₁ in e₂`, and allows `e₁` to generalize, implementing parametric polymorphism.
* If it is seen as a variant of System F, then the HM type system restricts the position of the quantifier--it can only be at the very beginning of the type. This restriction allows the HM type system to implement inference, while System F cannot implement type inference.

However, this does not mean that a stronger type system cannot perform type inference. In fact, Haskell has implemented stronger type inference, and languages like Scala, Typescript, Rust use a technique called "local type inference", which can eliminate most unnecessary type annotations based on some type annotations given by the programmer.
