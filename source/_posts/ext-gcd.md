---
title: 形式化递归循环变换的局限性--以扩展欧几里得算法为例
date: 2021-01-22 09:57:40
tags: PL
---

## 扩展欧几里得算法

“欧几里得算法”是数论中用于求得两个整数$a$和$b$的最大公约数$c=(a,b)$的算法。我之前的文章中已经介绍了这个算法的正确性来源于$(a,b)=(a-k b), k\in \Z$。而“扩展欧几里得算法”是一个可以得到
$$
sa+t b=(a,b)
$$
中的系数$s$和$t$的算法。

我第一次看到的这个算法实现，来自于一位精通算法竞赛的同学，他给出的实现类似于

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

这个实现把参数`x`和`y`当作传递结果的寄存器，有比较浓厚的命令式风格。

用比较易于理解的函数式风格实现如下：

```fsharp
let rec ext_gcd a b = 
  if b = 0 then (1, 0, a)
  else 
    let quo = a / b
    let r = a % b
    let (s, t, gcd) = ext_gcd b r
    (t, s - quo * t, gcd)
```

实际上，上面的两种实现都是基于一个手工的代换过程。以下述欧几里得算法过程为例：
$$
20 \div 7 = 2 ... 6 \\
7 \div 6 = 1 ... 1 \\
6 \div 1 = 6 ... 0
$$
将每个除法式都变成无除法的加减法式：
$$
\begin{aligned}
20 - 2 \times 7 = 6 \\
7 - 1 \times 6 = 1 \\
6 - 1 \times 6 = 0
\end{aligned}
$$
考虑倒数第二个式子
$$
7 - 1 \times \textcolor{red}{6} = 1
$$
把这个式子里的$\textcolor{red}{6}$用$20 - 2 \times 7 = \textcolor{red}{6}$代换，我们得到：
$$
7 - 1 \times (20 - 2 \times 7) = 1 \\
-1 \times 20  + 3 \times 7 = 1
$$
更一般地说，两个式子为
$$
\begin{aligned}
s_1 a_1 + t_1 b_1 = c_1 \\
s_2 b_1 + t_2 c_1 = c_2
\end{aligned}
$$
那么，把他们进行合并代换，就会变为：
$$
(s_1 t_2) a_1 + (s_2 + t_1 t_2) b_1 = c_2
$$
显然地，可以把这个过程抽象为一个对三元组的运算，这里用$*$表示，即：
$$
(s_1, t_1, c_1) * (s_2,t_2,c_2)=(s_1t_2, s_2 + t_1 t_2, c_2)
$$
这时再看那个函数式程序，我们会清晰很多：

`ext_gcd b r`求出的是$s b +  tr = gcd$的三元组$(s, t, gcd)$，而此三元组与本次除法定义的$1 \times a + (-quo) \times b = r$之三元组$(1, -quo, r)$正好可以进行$*$运算，而
$$
(1, -quo, r) * (s, t, gcd) = (t, s-(quo)t, gcd)
$$
这正是代码中的返回值`(t, s - quo * t, gcd)`.

我们可以给出直接使用这个运算的程序：

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



## 从递归到循环

### Wand 的模式

在40多年以前，MITCHELL WAND发表了一篇论文，叫做Continuation-Based Program Transformation Strategies，他总结了把递归程序变成循环程序的很多通用方法。

当然，我们这里的“循环”和“递归”的真正意义还需要进一步规范，比如说，

```scheme
(define (factor n)
  (let loop ([n n] [cont (lambda (x) x)])
    (if (= n 0)
        (cont 1)
        (loop (- n 1) (lambda (x) (cont (* n x)))))))
```

这种“循环”不是此处研究的“循环”，我们所说的“循环”必须是“使用固定空间，空间复杂度为$O(1)$”的东西。

通过研究类似于`factor`这种函数的两种写法：

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

WAND总结出了第一个模式：

![1](https://img.imgdb.cn/item/600a8c5a3ffa7d37b31acd50.jpg)

要求是$b$这个函数是$A \rightarrow A \rightarrow A$的运算，且具有结合性，其中$l_b$是右单位元（意思是说$b(x, l_b)=x$）。

我们是否可以用这个模式对上面的程序进行变换呢？看似是可以的，因为这里的$p(x)$，$a(x)$，$b$，$F$，$c(x)$，$d(x)$都可以明确地给出。

然而，这里有一个问题：上面定义的运算$*$到底是不是结合性的呢？

### 结合性？

一个运算的“结合性”，本质上描述了这个运算在“自组合”的时候能不能改变其运算顺序。考虑一个运算$f$，它具有结合性就是说
$$
f(a,f(b, c))=f(f(a, b), c)
$$
然而，我们的运算似乎没有结合性：

![2](https://img.imgdb.cn/item/600ac9d13ffa7d37b338b0f9.jpg)

这是怎么回事？找个具体例子研究一下：
$$
(54, 12) = 1 \\
1 \times 55 + (-3) \times 16 = 7, (1, -3, 7) \\
1 \times 16 + (-2) \times 7 = 2, (1, -2, 2) \\
1 \times 7 + (-3) \times 2 = 1, (1, -3, 1) \\
1 \times 2 + (-2) \times 1 = 0, (1, -2, 0) \\
$$
以前三个为例，如果正着做：

![3](https://img.imgdb.cn/item/600acc543ffa7d37b33a85ee.jpg)

这会得到令人非常吃惊的结果，因为
$$
6 \times 55 - 20 \times 16 = 10
$$
而我们的组合运算理论上应该得到
$$
6 \times 55 - 20 \times 16 = 1
$$
问题出在哪里呢？

考虑$(1, -3, 7) * (1, -2, 2)=(-2, 7, 2)$，等号右侧三元组的意义是：
$$
-2 \times 55 + 7 \times 16 = 2
$$
这个式子不能和
$$
1 \times 7 + (-3) \times 2 = 1
$$
用$*$进行运算。因为能与$-2 \times 55 + 7 \times 16 = 2$进行$*$运算的式子一定形如
$$
a \times 16 + b \times 2 = c
$$
可见，这已经不仅是没有结合性的问题。对于我们定义的$*$运算来说$(a*b)*c$根本没有意义。

这样一来，单纯地用Wang的方法不可能把这个递归变换为循环。

### 如何循环实现？

再次观察上面的式子
$$
1 \times 55 + (-3) \times 16 = \textcolor{red}{7}, (1, -3, 7) \\
1 \times 16 + (-2) \times \textcolor{red}{7} = \textcolor{blue}{2}, (1, -2, 2) \\
1 \times \textcolor{red}{7} + (-3) \times \textcolor{blue}{2} = 1, (1, -3, 1) \\
$$
我们好像发现了一种新的模式：
$$
s_{n-2} a_1 + t_{n-2} b_1 = c_{n-2} \\
s_{n-1} a_1 + t_{n-1} b_1 = c_{n-1} \\
s c_{n-2} + t c_{n-1} = c_3
$$
如果能够建立上面的关系，那么$s_n$和$t_n$就都是可以计算的：
$$
s (s_{n-2} a_1 + t_{n-2} b_1) + t (s_{n-1} a_1 + t_{n-1} b_1) = c_3 \\
s_n = ss_{n-2}+ ts_{n-1} \\
t_n = st_{n-2}+tt_{n-1}
$$
令人头痛的是开头两个式子的建立。实际上读者如果认真研究过递归版的算法，最后一个式子的建立用了一个有趣的技巧。我们这里也可以运用相同的技巧，但这技巧为何成立就留给读者自行研究吧：
$$
s_0 = 1, t_0 = 0 \\
s_1 = 0, t_1 = 1
$$
实现如下：

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

## 形式化递归循环变换的局限

形式化的递归-循环变换只能处理比较平凡的情况，类似于这种变换则有些束手无策。可见，不平凡的递归-循环变换仍然要依赖于人工完成。