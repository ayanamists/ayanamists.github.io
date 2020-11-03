---
author: ayanamists
date: 2020-09-12
title: 数论中的几个问题
---

## 辗转相除法的正确性

凡是学过编程的人，几乎没有不知道【辗转相除法】，或者【欧几里得算法】的。用scheme写出来，就是这样：

```scheme
(define (gcd a b)
  (if (= b 0)
      a
      (gcd b (modulo a b))))
```

大部分人还知道，GCD的理论基础是：

$$
(a\ mod\ b = c) \rightarrow
GCD(a,b) = GCD(b,c)
$$

但是，上面的这个式子为什么正确却有很多人不知道。我们这里来简单地证明一下这个式子的正确性。

首先，我们有：

$$
\frac{ {s, t}\in Z \\
a \mid b \\
a \mid c}{a\mid (s b-t c)}
$$

这可以通过考察整除的定义得到：

$$
\frac{\frac{\frac{a \mid b}{\exist{q},b=qa} \frac{a \mid c}{\exist{p},c=pa}}{sb-tc = sqa-tpa = a(sq-tp)}}{a\mid(sb-tc)}
$$

其实，证明了这个问题之后，上面的问题基本已经证完了，因为：

对任意的整数$ p $，$ p \mid a, p \mid b $，会有：

$$
p \mid sa - tb
$$

而$c$恰恰就是符合以上形式的：

$$
c = a - \lfloor\frac{a}{b}\rfloor b
$$

所以

$$
p \mid c
$$

这也就是说，任意的$ a $、$ b$公因数$ p $满足$ p $也是$ c $的因数，所以就有了原式。

我们会发现，不仅有

$$
GCD(a,b)=GCD(b,c)
$$

而且也有：

$$
GCD(a,c)=GCD(a,b)
$$

那么，为什么要选择上一个式子来构造循环呢？因为**我们需要让$$ c $$下降到0**

## 贝祖恒等式的特殊情况

贝祖恒等式是说，

$$
\exists{s,t}(sa+tb=GCD(a,b))
$$

如果 $a, b$ 互素，那么我们有

$$
\exists{s,t}(sa+tb=1)
$$

我们切记一个事实，当$a,b$互素时，这个定理可以反着用：

$$
\frac{\exists{s,t}(sa+tb=1)}{GCD(a,b)=1}
$$

这是为啥呢？其实原因特别简单。因为对于$a,b$的每一个公因数$d$都有

$$
d \mid(sa+tb)
$$

所以

$$
d\mid1
$$

这当然说明

$$
d=\pm1
$$
如果一个数只有1或-1的因子，它当然是1或-1，也就说明了$a$和$b$互素。

## 把整数看成向量

算数基本定理告诉我们，一个整数可以被唯一分解为素数的乘积：

$$
x=\prod_{i=1}^{n}{p^{a_i}_i}
$$

这给了我们一个全新的视角，我们可以把

$$
[2,3,5,7,11,..]
$$

这些素数看成一个【基向量】，把每个整数看成一个【向量】，其中的每个元素是对应位置基向量素数的次数，比如说，

$$
10=2\times5=[1,0,1,0,...] \\
20=2^2\cdot5=[2,0,1,0,...]
$$

可以证明，对于每一个整数，这种形式都是唯一的。

在这个形式下：

$$
a\cdot b=[a_0,a_1,..]\cdot[b_0,b_i,...]=[a_0+b_0,a_1+b_1,...]\\
GCD(a,b)=[\min(a_0,b_0),\min(a_1,b_1),...]\\
LCM(a,b)=[\max(a_0,b_0),\max(a_1,b_1),...]
$$

有了这种视角，很多事实一下子变得豁然开朗，如：

$$
LCM(a,b)=\frac{a\cdot b}{GCD(a,b)}
$$

就可以直接通过

$$
\max(a,b)=a + b - \min(a,b)
$$

得到。

## 关于欧几里得引理

【欧几里得引理】和【质数】是无关的，虽然在证明【算数基本定理】时用到的是质数。它的严格表述是：

>如果一个正[整数](https://zh.wikipedia.org/wiki/整数)整除另外两个正整数的乘[积](https://zh.wikipedia.org/wiki/积)，第一个整数与第二个整数[互质](https://zh.wikipedia.org/wiki/互質)，那么第一个整数整除第三个整数。
>
>对于正整数$a,b,c$，如果$a \mid bc$且$(a,b)=1$，那么$a \mid c $

这个定理看似显然，实际上是需要证明的。为什么它【显然】呢？因为在算数基本定理的保证下，这个结论很显然。但是【算数基本定理】恰恰就是通过【欧几里得引理】证明出来的，如果拿着【算数基本定理】证明【欧几里得引理】，显然是循环论证。

它的正确证明如下：

正整数$p \mid a \cdot b $，则存在整数$r$，使得：

$$
rp = ab
$$

由于$(a,p)=1$，由贝祖定理：

$$
\exists{s,t\in Z}(sa+tp=1)
$$

所以：

$$
\exists{s,t\in Z}(sab+tpb=b)
$$

注意到$ab=rp$，我们有：

$$
\exists{s,t\in Z}(srp+tpb=b)
$$

即

$$
p (sr+tb) = b
$$

即

$$
p \mid b
$$

## 关于二次剩余的求解

互联网上的二次剩余求解大都是基于【扩域】的算法，但在《信息安全数学基础》这们课里，老师教给我们的算法却不是这样的。这里给出Racket代码，请自行研究其内涵吧：

```scheme
#lang racket

(#%require (only racket/base random))
(#%require math/number-theory)

(define (modulo-muti a b p)
  (modulo (* a b) p))

(define fast-pow
  (lambda (a b p)
    (if (= b 0)
        1
        (let ((ret (fast-pow a (floor (/ b 2)) p)))
          (if (= (modulo b 2) 1)
              (modulo-muti (modulo-muti ret ret p) a p)
              (modulo-muti ret ret p))))))

(define Euler-pre
  (lambda (a p)
    (= 1 (fast-pow a (/ (- p 1) 2) p))))

(define (rand-choose p)
  (let ((this-time (random p)))
    (if (or
         (= (modulo this-time p) 0)
         (Euler-pre this-time p))
        (rand-choose p)
        this-time)))

(define (get-2^n p)
  (if (= (modulo p 2) 0)
      (let ((res (get-2^n (/ p 2))))
        (cons (+ (car res) 1)
              (cdr res)))
      (cons 0 p)))

(define (pow a b)
  (if (= b 0)
      1
      (* a (pow a (- b 1)))))

(define (solve-quad-res a p)
  (define a-1 (modular-inverse a p))
  (define t (car (get-2^n (- p 1))))
  (define s (cdr (get-2^n (- p 1))))
  (define b (pow (rand-choose p) s))
  (define (solve x_t t-k)
    ;(printf "~a ~a ~n" x_t t-k)
    (if (= t-k 0)
        x_t
        (let* ((a-1x^2 (modulo-muti a-1 (modulo-muti x_t x_t p) p))
              (y^t-k-1 (fast-pow a-1x^2 (pow 2 (- t-k 1)) p)))
          (solve
           (if (= y^t-k-1 1)
               x_t
               (modulo-muti x_t (fast-pow b (pow 2 (- (- t 1) t-k)) p) p))
           (- t-k 1)))))
  (solve (fast-pow a (/ (+ s 1) 2) p) (- t 1)))

(solve-quad-res 186 401)
(solve-quad-res 2 401)
(solve-quad-res 3 401)
(solve-quad-res 5 401)
(solve-quad-res 7 401)
(solve-quad-res 11 401)
```

