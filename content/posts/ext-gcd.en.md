---
title: Discussing the Extended Euclidean Algorithm
date: 2021-01-22 09:57:40
categories:
  - Algorithm
tags: 
  - Algorithm
  - Number Theory
  - Program Transformation
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

## Extended Euclidean Algorithm

The "Euclidean Algorithm" is an algorithm in number theory used to find the greatest common divisor $c=(a,b)$ of two integers $a$ and $b$. I have previously explained that the correctness of this algorithm comes from $(a,b)=(a-k b), k\in \Z$. The "Extended Euclidean Algorithm" is an algorithm that can obtain the coefficients $s$ and $t$ in

$$
sa+t b=(a,b)
$$

The first time I saw the implementation of this algorithm was from a classmate who was proficient in algorithm competitions. His implementation was similar to

```c++
int exgcd(int a,int b,int &x,int &y)
{
    if(b==0)
    {
        x = 1; y = 0; return a;
    }
    int r = exgcd(b, a % b, x, y);
    int t = y;
    y = x - (a / b) * y;
    x = t;
    return r;
}
```

This implementation uses the parameters `x` and `y` as registers to pass the result, which has a strong imperative style.

A more understandable functional style implementation is as follows:

```fsharp
let rec ext_gcd a b = 
  if b = 0 then (1, 0, a)
  else 
    let quo = a / b
    let r = a % b
    let (s, t, gcd) = ext_gcd b r
    (t, s - quo * t, gcd)
```

In fact, both of the above implementations are based on a manual substitution process. Take the following Euclidean algorithm process as an example:

$$
20 \div 7 = 2 ... 6 \\\\
7 \div 6 = 1 ... 1 \\\\
6 \div 1 = 6 ... 0
$$

Change each division formula into an addition and subtraction formula of the previous division:

$$
\begin{aligned}
20 - 2 \times 7 = 6 \\\\
7 - 1 \times 6 = 1 \\\\
6 - 1 \times 6 = 0
\end{aligned}
$$

Consider the penultimate formula

$$
7 - 1 \times \textcolor{red}{6} = 1
$$

Replace the $\textcolor{red}{6}$ in this formula with $20 - 2 \times 7 = \textcolor{red}{6}$, we get:

$$
7 - 1 \times (20 - 2 \times 7) = 1 \\\\
-1 \times 20  + 3 \times 7 = 1
$$

More generally, the two formulas are

$$
\begin{aligned}
s_1 a_1 + t_1 b_1 = c_1 \\\\
s_2 b_1 + t_2 c_1 = c_2
\end{aligned}
$$

Then, if we merge and substitute them, it will become:

$$
(s_1 t_2) a_1 + (s_2 + t_1 t_2) b_1 = c_2
$$

Obviously, this process can be abstracted into an operation on a triplet, represented here by $*$, that is:

$$
(s_1, t_1, c_1) * (s_2,t_2,c_2)=(s_1t_2, s_2 + t_1 t_2, c_2)
$$

Looking at the functional program again, we will be much clearer:

`ext_gcd b r` gets the triplet $(s, t, gcd)$ of $s b +  tr = gcd$, and this triplet can perform the $*$ operation with the triplet $(1, -quo, r)$ of $1 \times a + (-quo) \times b = r$ defined by this division, and

$$
(1, -quo, r) * (s, t, gcd) = (t, s-(quo)t, gcd)
$$

This is exactly the return value `(t, s - quo * t, gcd)` in the code.

We can give a program that directly uses this operation:

```fsharp
let F x1 x2 = 
  let s1, t1, c1 = x1
  let s2, t2, c2 = x2 
  (s1 * t2, s2 + t1 * t2, c2)

let rec ext_gcd2 a b =
  if b = 0 then (1, 0, a)
  else 
    let quo = a / b
    let r = a % b
    F (1, -quo, r) (ext_gcd2 b r)
```

## From recursion to loop

### Wand's pattern

More than 40 years ago, MITCHELL WAND published a paper called Continuation-Based Program Transformation Strategies, in which he summarized many general methods for transforming recursive programs into loop programs.

Of course, the real meaning of "loop" and "recursion" here needs further specification, for example,

```scheme
(define (factor n)
  (let loop ([n n] [cont (lambda (x) x)])
    (if (= n 0)
        (cont 1)
        (loop (- n 1) (lambda (x) (cont (* n x)))))))
```

This kind of "loop" is not the "loop" studied here, the "loop" we are talking about must be something with "fixed space, space complexity of $O(1)$".

By studying two ways of writing functions similar to `factor`:

```scheme
(define (factor n)
  (if (= n 0)
      1
      (* n (factor (- n 1)))))
```

```scheme
(define (factor n)
  (let loop ([n n][acc 1])
    (if (= n 0)
        acc
        (loop (- n 1) (* acc now)))))
```

WAND summarized the first pattern:

![1](https://img.imgdb.cn/item/600a8c5a3ffa7d37b31acd50.jpg)

The requirement is that $b$ is an operation of $A \rightarrow A \rightarrow A$ and has associativity, where $l_b$ is the right unit element (meaning $b(x, l_b)=x$).

Can we use this pattern to transform the above program? It seems possible, because here $p(x)$, $a(x)$, $b$, $F$, $c(x)$, $d(x)$ can be clearly given.

However, there is a problem: is the operation $*$ defined above associative?

### Associativity?

The "associativity" of an operation essentially describes whether this operation can change its operation order when it is "self-combined". Consider an operation $f$, it has associativity, that is to say

$$
f(a,f(b, c))=f(f(a, b), c)
$$

However, our operation seems to be non-associative:

![2](https://img.imgdb.cn/item/600ac9d13ffa7d37b338b0f9.jpg)

What's going on? Let's study a specific example:

$$
(54, 12) = 1 \\\\
1 \times 55 + (-3) \times 16 = 7, (1, -3, 7) \\\\
1 \times 16 + (-2) \times 7 = 2, (1, -2, 2) \\\\
1 \times 7 + (-3) \times 2 = 1, (1, -3, 1) \\\\
1 \times 2 + (-2) \times 1 = 0, (1, -2, 0) \\\\
$$

Take the first three as examples, if you do it in the correct order:

![3](https://img.imgdb.cn/item/600acc543ffa7d37b33a85ee.jpg)

This will result in a very surprising result, because

$$
6 \times 55 - 20 \times 16 = 10
$$

And our combined operation should theoretically yield

$$
6 \times 55 - 20 \times 16 = 1
$$

Where is the problem?

Consider $(1, -3, 7) * (1, -2, 2)=(-2, 7, 2)$, the meaning of the triplet on the right side of the equation is:

$$
-2 \times 55 + 7 \times 16 = 2
$$

This equation cannot be operated with

$$
1 \times 7 + (-3) \times 2 = 1
$$

using $\*$. Because the equation that can be operated with $-2 \times 55 + 7 \times 16 = 2$ using $*$ must be in the form of

$$
a \times 16 + b \times 2 = c
$$

As you can see, this is not just a problem of lack of associativity. For the $\*$ operation we defined, $(a\*b)\*c$ has no meaning at all.

In this way, it is impossible to transform this recursion into a loop simply by using Wang's method.

### How to implement in a loop?

Observe the equation above again

$$
1 \times 55 + (-3) \times 16 = \textcolor{red}{7}, (1, -3, 7) \\\\
1 \times 16 + (-2) \times \textcolor{red}{7} = \textcolor{blue}{2}, (1, -2, 2) \\\\
1 \times \textcolor{red}{7} + (-3) \times \textcolor{blue}{2} = 1, (1, -3, 1) \\\\
$$

We seem to have discovered a new pattern:

$$
s_{n-2} a_1 + t_{n-2} b_1 = c_{n-2} \\\\
s_{n-1} a_1 + t_{n-1} b_1 = c_{n-1} \\\\
s c_{n-2} + t c_{n-1} = c_3
$$

If the above relationship can be established, then $s_n$ and $t_n$ can be calculated:

$$
s (s_{n-2} a_1 + t_{n-2} b_1) + t (s_{n-1} a_1 + t_{n-1} b_1) = c_3 \\\\
s_n = ss_{n-2}+ ts_{n-1} \\\\
t_n = st_{n-2}+tt_{n-1}
$$

The headache is the establishment of the first two equations. In fact, if the reader has carefully studied the recursive version of the algorithm, the establishment of the last equation uses an interesting technique. We can also use the same technique here, but why this technique works is left to the reader to study:

$$
s_0 = 1, t_0 = 0 \\\\
s_1 = 0, t_1 = 1
$$

The implementation is as follows:

```fsharp
let rec ext_gcd_loop a b =
  if b = 0 then (1, 0, a)
  else 
    let rec loop a b s_n1 t_n1 s_n2 t_n2 = 
      let quo = a / b 
      let r = a % b
      let s = 1
      let t = -quo
      if r = 0 then (s_n1, t_n1, b)
      else 
        loop b r (s * s_n2 + t * s_n1) (s * t_n2 + t * t_n1) s_n1 t_n1
    loop a b 0 1 1 0
```
