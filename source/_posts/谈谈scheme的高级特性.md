---
title: 谈谈scheme的高级特性
date: 2020-10-13 23:43:50
tags:
---

## `quote` `quasiquote` `unquote` 的爱恨情仇

### quote

经常使用scheme写程序的人，一般都会知道【符号】到底是指什么。但是，很多人却并不真正知道`(quote A)`和【符号】到底是不是一个东西。

The Scheme Programming Language对`quote`的描述是这样的：

> `'*obj*` is equivalent to `(quote *obj*)`. The abbreviated form is converted into the longer form by the Scheme reader (see `read`).
>
> `quote` inhibits the normal evaluation rule for `*obj*`, allowing `*obj*` to be employed as data. Although any Scheme object may be quoted, quotation is not necessary for self-evaluating constants, i.e., numbers, boallowing `*obj*` to be employed as dataoleans, characters, strings, and bytevectors.
>
> Quoted and self-evaluating constants are immutable. That is, programs should not alter a constant via `set-car!`, `string-set!`, etc., and implementations are permitted to raise an exception with condition type `&assertion` if such an alteration is attempted. If an attempt to alter an immutable object is undetected, the behavior of the program is unspecified. An implementation may choose to share storage among different constants to save space.

说实话，如果你不知道`quote`到底是做什么的，你恐怕难以理解上面这些东西到底是在说什么。其实，`quote`的真正作用用一个词就可以概括：**不要求值**

如果不使用`quote`的话，在scheme中（进行完宏展开后）一个`s-expression`可能有这么两种情况：

- 是一个求值式或者说函数调用式，如`(+1 2)`
- 是一个由关键字决定的结构，如`(lambda (x) x)`

但是，我们其实想要有第三种情况：

- 是一个形如`(list a b ...)`的列表

换句话说，我们想写得简单一点，让`(list 1 2 3)`用`(1 2 3)`就可以表达。但问题是`(1 2 3)`在自然状态下是一个【求值式】，所以直接这么搞是不行的，而要使用`(quote (1 2 3))`，而`(quote (1 2 3))`可以被简写为：

```scheme
'(1 2 3)
```

我们再给出几个例子：

```scheme
> '1
1
> '"a"
"a"
```

```scheme
> (+ (car '(1 2 3)) 2)
3
```

```scheme
> (string? (car '("1" 2 3)))
#t
```

```scheme
> (vector-ref 
   (car '(#(1 2 3) 2 3))
   0)
1
```

可以看到，如果`a`本身是一个【self-evaluating constants, i.e., numbers, booleans, characters, strings, and bytevectors】，那么`(quote a)`就是`a`.

但是，这里还有一个问题。请思考一下下面的代码会得到什么：

```scheme
> (define a 10)
> 'a
```

是`10`吗？**请再次回忆我之前说的【不要求值】**

在scheme中，把一个`identifier`变成其对应的值，显然就是求值。

所以，这段代码不会`a`的值，它得到的是`a`本身：

```scheme
> (define a 10)
> 'a
a
```

这里的【`a`本身】就是所谓的【符号】，如果`x`是一个【标识符】，那么`(quote x)`就会得到一个【符号】：

```scheme
> (symbol? 'x)
#t
```

看到这里，你应该明白了什么叫做【allowing `*obj*` to be employed as data】了吧？喝一杯茶，我们下面要介绍一点更好玩的东西。

### quasiquote & unquote

`quote`可以很方便地构造【self-evaluating constants】的列表，但却不能实现下面的要求：

```scheme
(define a 10)
'(a a a) ;我们想要(list 10 10 10)而不是(list 'a 'a 'a)
```

这该如何是好呢？

`quote`实际上相当于引入了一个【不求值的环境】，那么，只要通过某种手段把这个【不求值的环境】变成【普通环境】，那么问题就迎刃而解了。scheme加入了`qusiquote`来支持【通过`unquote`变成普通环境】：

```scheme
> (define a 10)
> (quasiquote ((unquote a) (unquote a) (unquote a) a))
(10 10 10 a)
```

这样有点麻烦，scheme用\`表示`quasiquote`，用`,`表示`unquote`，上面的代码可以写成：

```scheme
`(,a ,a ,a a)
```

用这个语法，我们就可以方便地创建列表了。

## scheme宏系统

scheme的宏系统是比较难以掌握的一个部分，因为你不得不认真阅读the scheme programming language关于宏系统的部分。但是scheme的卫生宏系统是还没有被各类不基于s表达式的编程语言抄回来的特性，换句话说，这是它的*特色*，不得不品尝。

不过，我也懒得再写下去了，这里给出一个tspl里没有的拼字符串的例子吧：

```scheme
(define-syntax 缝合
  (lambda (x)
    (define (get-feng-syntax x y)
      (datum->syntax x (symbol-append (syntax->datum x)
                                      (syntax->datum y))))
    (syntax-case x ()
      ((_ a b)
       #`(define #,(get-feng-syntax #'a #'b) 10)))))

(缝合 我喜欢 拼字符串)

(printf "~a" 我喜欢拼字符串) ;得到10
```









