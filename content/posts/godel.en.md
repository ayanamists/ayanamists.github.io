---
title: "Is There Something Wrong with Proving Gödel's Incompleteness Using Multithreading?"
date: 2023-09-01T12:34:12+08:00
draft: false
tags:
    - Mathematical Logic
    - Computability
categories:
    - Academics
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

{{<  admonition "warning" >}}
This article is not a strict academic paper and may contain some errors and omissions. Please forgive me.
{{< /admonition >}}

## Introduction

Kurt Gödel's several famous theorems form the cornerstone of modern logic. Among them, Gödel's (first and second) Incompleteness Theorems are widely acclaimed.

For a long time, many people, including the author, have found the proof of Gödel's Incompleteness Theorem to be very cumbersome, with very complex technical details, so it is quite difficult to learn. At the [∞-type Café Summer School](https://m4p1e.github.io/ntype-cafe-summer-school/) in 2023, teacher Ocua gave a very elegant and concise comprehensive lecture on the proof of Gödel's Incompleteness Theorem. This lecture made me realize that Gödel's Incompleteness Theorem is actually closely related to computability, or [recursion theory](https://plato.stanford.edu/entries/recursive-functions/).

In this article, I will combine my understanding of Gödel's Incompleteness Theorem to introduce a simplest (perhaps not strict) proof of Gödel's Incompleteness Theorem, trying to make ordinary programmers understand it after learning simple first-order logic.

## First-Order Logic

First-order logic (First order logic, FOL) is the logical system discussed in this article. First-order logic has syntax and semantics. If the reader has not learned first-order logic in school, then you can understand it this way:

1. First-order logic is a language, similar to our common programming languages, it has a set of formalized syntax (Syntax)
    - The "sentences" of first-order logic can be seen as a syntax tree (Abstract Syntax Tree), or an algebraic data type (ADT)
    - This syntax tree has two layers, one is "proposition" (Statement), and the other is "expression" (Expression)
    - The expression layer includes variables, functions (calls), etc., $x$, $f(x)$, $g(x, y)$ are all expressions
    - The proposition layer is composed of expressions, and expressions are combined with equal signs or relations to form propositions, such as $f(x_1) = f(x_2)$, $P(x_1, f(x_2))$
    - Propositions can be connected with logical connectors to express specific meanings, such as $f(x_1) = f(x_2) \land P(x_1, f(x_2))$ is also a proposition. Such logical connectors also have $\rightarrow$, $\lor$, $\neg$
    - The free variables in the proposition (free variable) are bound with quantifiers to get new propositions. For example, $\forall x_1 \exists x_2. f(x_1) = f(x_2)$
2. This language of first-order logic needs to specify its semantics, which is implemented through the model (Model). The related content is more complicated and will not be discussed for the time being.
3. First-order logic has many proof systems, such as [Hilbert System](https://en.wikipedia.org/wiki/Hilbert_system), these proof systems are independent of semantics. Simply put, a proof system can give a proposition like $\Gamma \vdash \psi$ [^1] through "calculation", where $\Gamma$ represents a set of propositions, that is, a bunch of propositions, and $\psi$ represents the proposition to be proved.
4. When we use first-order logic, we use first-order theory (Theory). A first-order theory consists of a symbol table $S$ and a set of axioms $T$. The symbol table specifies all the symbols that may appear in the proposition, including constants, function symbols, and relation symbols; the axiom set specifies the self-evident propositions in this theory, they are the starting point of the proof system[^2]. For example, the famous Peano arithmetic number can be used as a first-order theory, its symbol table has:
    - Constant symbol $0$
    - Function symbol $+$
    - Function symbol $\times$
    - Function symbol $S$ (representing successor)
    - Relation symbol $\le$

Our introduction to first-order logic ends here. First-order logic has very rich mathematical properties, which are the main discussion objects of mathematical logic courses. If the reader feels uncomfortable during the reading process, you might as well find a reliable mathematical logic book, such as [1, 2], etc., to learn some related knowledge.

## Gödel's Incompleteness Theorem

"Completeness" is a property of a first-order theory. A first-order theory $T$ is complete, which means:

> For any proposition $\psi$ constructed from the symbol set of this theory, either $T \vdash \psi$ or $T \vdash \neg \psi$

In other words, any proposition in this theory can either be proved or refuted.

Notice that this is a purely syntactic concept, and it seems to require specifying a particular proof system. However, various first-order logic proof systems, if not specially constructed, are equivalent in proof power. And because of the existence of Gödel's completeness theorem, $T \vDash \psi$ if and only if $T \vdash \psi$, so this is actually also a semantic concept. In this way, we generally do not need to specify a proof system. In the formalization of Gödel's incompleteness theorem [3], such specification may be necessary.

For a first-order theory, we naturally hope that it is complete. For example, in Peano arithmetic, the following intuitive propositions can be proved:

- $\forall n. n \le S(n) $
- $\forall n_1, n_2 . n_1 + n_2 = n_2 + n_1$

And correspondingly, propositions that are not intuitively valid can be refuted.

If the first-order theory is simple enough, incompleteness is easy to achieve. For example, consider a first-order theory $A$ with a symbol table of $\\{ c, e \\}$ and axioms of $\emptyset$, it is obvious that:

- $A \not\vdash c = e$
- $A \not\vdash c \neq e$

Intuitively, this is because the axiom set of the theory is too small. Adding an axiom (often called "expansion", extension) $c = e$ to $A$ can make $A$ a complete theory.

For any theory $T$, if $\psi$ can neither be proved nor refuted in this theory (called $\psi$ is independent of $T$), then adding $\psi$ or $\neg \psi$ to the axioms of $T$ can eliminate this incompleteness.

This seems to reveal a beautiful picture: we can build all mathematics on some complete first-order theory, in this ultimate theory, any proposition can either be proved or refuted. In this way, the vague and chaotic mathematical language can be completely replaced by clear and neat first-order logic...

Disappointingly, in 1931, Gödel proved in his published paper "Über formal unentscheidbare Sätze der Principia Mathematica und verwandter Systeme I" [13] that as long as a formal system $T$ includes elementary arithmetic[^3], there must be propositions that can neither be proved nor disproved. Note that adding axioms is of no use, because after adding axioms, the antecedent of the incompleteness theorem still holds, and the consequent can still be derived. [^4]

This is equivalent to sentencing the beautiful picture we imagined above to death. Of course, we are programmers, not mathematicians, what does this have to do with us?

## Decidability

After Gödel published his paper in 1931, Church and Turing respectively proposed another major issue about first-order logic--the negative proof of decidability in 1936-1937.

As mentioned earlier, the propositions of first-order logic can be seen as a syntax tree (or string), and the first-order theory is a symbol table and a set of axioms. Under certain conditions (such as the recursive axiomatization to be discussed below), these can all be seen as data in the program.

Obviously we will ask: Is there a program $\texttt{p}$ that determines whether a proposition $\psi$ can be proved in $T$?

The answer to this question is negative: **there is no such program**. We will leave the proof method to the next section. This fact is a bit counter-intuitive, because the proof in an axiom system, theoretically speaking, can be **enumerated**.

How to enumerate? Consider Peano arithmetic and the Hilbert system. In the axioms of Peano arithmetic, the induction axiom is infinite, but it has a fixed pattern:

$$
\begin{aligned}
\forall \vec{y_k}. (& (\varphi(0, \vec{y_k})\ \land \\\\
    & (\varphi(x, \vec{y_k}) \rightarrow \varphi(S(x), \vec{y_k}))) \\\\ 
    & \rightarrow \forall x. \varphi(x, \vec{y_k}))
\end{aligned}
$$

What needs to be enumerated here are $k$ and $\varphi$. $k$ goes without saying, $\varphi$ is all predicates, its syntax is also determined, and can be generated by fixed rules. However, writing a `gen` function directly is quite complicated. We can borrow Gödel's method here.

First of all, the symbol set of Peano arithmetic is infinite, but we can divide it into two areas:

- Ten logical symbols (after simplification)
- Infinite variable symbols (denoted as $v_0$, $v_1$, $v_2$, ...)

By encoding the symbols of these two areas in order, we can give encoding and decoding functions. The encoding function maps symbols to natural numbers, and the decoding function maps natural numbers to symbols.

```python
# put '∀' to 1 , instead of 0
LOGIC_SYM = [ 'dummy', '∀', '0', 'S', '+', '×', '(', ')', '¬', '→', '=' ]

def decode(n):
    if n <= len(LOGIC_SYM):
        return LOGIC_SYM[n]
    else:
        # append '$' to variable head
        return f"$v_{n - len(LOGIC_SYM)}"

def encode(sym):
    if sym[0] == '$':
        # is variable, start with '$'
        index = sym[3:]
        return int(index) + len(LOGIC_SYM)
    elif sym in LOGIC_SYM and sym != 'dummy':
        return LOGIN_SYM.index(sym)
    else:
        # error
        return -1
```

Similarly, a proposition (that is, a string, a list of characters) can also be encoded and decoded. The technique used by Gödel is as follows:

$$
\sharp [ a_1, a_2, a_3, ..., a_n ] = p_1^{\sharp a_1} \times p_2^{\sharp a_2} \times \cdots \times p_n^{\sharp a_n}
$$

Here, $\sharp a_i$ represents the `encode(aᵢ)` we just defined. And $p_i$ represents the $i$th prime number

```python
from sympy import nextprime, Pow, factorint

def encode_str(stmt):
    pᵢ = 2
    res = 1
    for aᵢ in stmt:
        res *= Pow(pᵢ, encode(aᵢ))
        pᵢ = nextprime(pᵢ)
    return res

def decode_str(n):
    """
    n must be of form `Πᵢ pᵢ^{aᵢ}, aᵢ ≠ 0`, or this function is undefined
    """
    res = []
    factors = factorint(n)
    # factors is like { pᵢ : aᵢ }
    for pᵢ, aᵢ in sorted(factors.items()):
        res.append(decode(aᵢ))
    return res
```

The range of `decode_str` includes all strings. In this way, the traversal of strings can be implemented by the traversal of natural numbers.

A string may not be a valid proposition. Therefore, a function is needed to judge whether a string is a valid proposition. As long as we have this function, we can enumerate all propositions:

```python
def is_valid_stmt(stmt):
    # this function is hard to write
    pass

def gen_all_stmts():
    n = 0
    while True:
        stmt = decode_str(n)
        if is_valid_stmt(stmt):
            # do something here
        n += 1
```

The function `is_valid_stmt` is similar to the [parser](https://en.wikipedia.org/wiki/Parsing) of a programming language. Although it is difficult to write, it can definitely be written.

Similarly, we can enumerate all proofs (that is, a list of propositions), and then construct a function to judge whether a proposition can be proved:

```python
def is_valid_proof(stmt_l):
    pass

def encode_proof(proof):
    # like encode_str()
    pass

def decode_proof(n):
    pass

def can_prove(ψ):
    n = 0
    while True:
        stmt_l = decode_proof(n)
        if is_valid_proof(stmt_l):
            stmt_last = stmt_l[-1]
            if stmt_last == ψ:
                return True
        n += 1
```

For the program `can_prove`, the following observations are made:

- If the last one in the list is the $\psi$ to be proved, then obviously, $T \vdash \psi$ is established.
- Our enumeration traverses all proofs, so if $T \vdash \psi$, that is, there is a proof that meets the requirements, then that proof will definitely be found.
- If $T \not\vdash \psi$, this algorithm will not stop at all, it will keep looping.

This is very similar to the situation of the halting problem, which is called "semi-decidability" in computability theory. Semi-decidability is the core concept used in our proof below, and it is worth discussing carefully.

> A set[^5] $S$ is semi-decidable, if and only if there exists a program $\texttt{p}$, such that for all $e \in S$, $\texttt{p}(e) = \texttt{True}$.

The key to semi-decidability is that it does not discuss whether the program halts when $e \not\in S$. Although if $\texttt{p}(e) = \texttt{True}$, that is, when the program outputs $\texttt{True}$, according to if and only if, $e \in S$. But if the input $e_1 \not\in S$, whether $\texttt{p}(e)$ halts is not mentioned in the definition. Therefore, a decidable set must be a semi-decidable set, but a semi-decidable set is not necessarily a decidable set.

> The halting problem is semi-decidable, not decidable

The proof is simple, the program that exists for semi-decidability is constructed as follows:

```python
def p(prog, x):
    exec(prog, x)
    return True
```

If `prog(x)` halts, then the interpreter `exec(prog, x)` also halts.

Semi-decidability has several equivalent definitions:

1. Recursively enumerable (r.e.). A set $S$ is recursively enumerable, if and only if $S = \emptyset$ or there exists a recursive total function $f$, such that $S = \\{ y\ |\ \exists x. f(x) = y \\}$, that is, $S$ is the range of $f$.
2. The partial [characteristic function](https://zh.wikipedia.org/zh-hans/%E6%8C%87%E7%A4%BA%E5%87%BD%E6%95%B0) of $S$ is partially recursive.
3. $S$ is the range of some partial recursive function.

What is a "partial function"? Simply put, it is a function that is not defined on some inputs. A program that does not halt on some inputs, its "image", that is, $\\{ (x, y)\ |\ x \in A, \texttt{p}(x) = y \\}$ is a partial function.

These equivalences need to be proven, you can refer to [1, 5].

The construction of `is_valid_stmt` and `is_valid_proof` requires that there exists an algorithm to determine whether a string is in the axiom set. This puts requirements on the axiom set. In fact, in the conditions of Gödel's incompleteness theorem, there is a requirement that the axiom set needs to be a recursive set, which is equivalent to being decidable, and it adds the restriction that it must halt on any input to the requirement of semi-decidability.

## Proof of Gödel's Incompleteness Theorem

We already know that, at least for Peano arithmetic $\mathsf{PA}$, the problem of $\mathsf{PA} \vdash \psi$ is semi-decidable. Based on this result and a little programming trick, we can immediately get Gödel's incompleteness theorem.

First, assume that $\mathsf{PA}$ is complete, that is, for any $\psi$, $\mathsf{PA} \vdash \psi$ or $\mathsf{PA} \vdash \neg \psi$. Let the semi-decider of $\mathsf{PA}$ be $\texttt{p}$, a decider $\texttt{p\\_total}$ of $\mathsf{PA}$ can be constructed. The construction is as follows:

```python
from multiprocessing.pool import ThreadPool

def can_prove_input(ψ):
    can_prove(ψ)
    return ψ

def p_total(ψ):
    pool = ThreadPool(processes=2)
    result = pool.apply_async(can_prove_input, (ψ, ¬ψ))
    which = result.get()
    if which == ψ:
        return True
    else:
        return False
```

Unexpectedly, the key to this program is concurrency! Another name for concurrency is "non-deterministic computation". In fact, I just want to simulate non-deterministic computation with this method. (That is, "simultaneously" calculate `can_prove_input(ψ)` and `can_prove_input(¬ψ)`)

Analyze $\texttt{p\\_total}$ carefully, consider completeness, and discuss it in different cases:

- If $\mathsf{PA} \vdash \psi$, then `can_prove_input(ψ)` will halt, `result.get()` returns `ψ`, so the entire function returns `True`. Note that at this time `can_prove_input(¬ψ)` will not halt, so `result.get()` will definitely return `ψ`.
- If $\mathsf{PA} \vdash \neg\psi$, then `can_prove_input(¬ψ)` will halt, `result.get()` returns `¬ψ`, the entire function returns `False`. According to the consistency of $\mathsf{PA}$, $\mathsf{PA} \not\vdash \psi$ at this time.

Therefore, $\texttt{p\\_total}$ is a decider of $\mathsf{PA} \vdash \psi$. In this way, $\mathsf{PA}$ is decidable, which contradicts the undecidable result. Therefore, the assumption is not established, $\mathsf{PA}$ is not complete.

Readers may not be convinced: Isn't this a trick? Where does Gödel know about multithreading? In fact, non-deterministic Turing machines and deterministic Turing machines are equivalent, and multithreading can also be simulated by single-threading. However, we can still give a more "normal" $\texttt{p\\_total}$.

```python
def can_prove_n(ψ, n):
    n0 = 0
    while n0 <= n:
        stmt_l = decode_proof(n0)
        if is_valid_proof(stmt_l):
            stmt_last = stmt_l[-1]
            if stmt_last == ψ:
                return True
        n += 1
    return False

def p_total(ψ):
    n = 0
    while True:
        if can_prove_n(ψ, n):
            return True
        elif can_prove_n(¬ψ, n):
            return False
        
        n += 1
```

Here, we have changed the original `can_prove(ψ)` to `can_prove_n(ψ, n)`, which is whether $\psi$ can be proved within $n$ steps. In this way, `can_prove_n(ψ, n)` will definitely halt, so it can be directly called in `p_total` for sequential calculation.

## Proof of Undecidability

From the above discussion, we can see:

- Although semi-decidability requires a relatively complex program, it is overall intuitive
- With semi-decidability and undecidability, we can directly obtain Gödel's incompleteness theorem

The key to the problem is how to obtain the undecidability of $\mathsf{PA}$. This article will not strictly provide a proof, but will simply introduce the idea of the proof.

### Using Church's Results

To prove undecidability, we must introduce a relatively complex concept--representability. Gödel himself does not seem to have directly used this concept [4], but used an equivalent form of it.

Church first proved in his 1936 paper [11] that there exists an undecidable problem (although some scholars are not very satisfied with his proof), this problem is:

> Given $a, b \in \Lambda^{\circ}$, does $a \approx_{\beta} b$ hold?

Here, $\Lambda^{\circ}$ refers to "closed lambda terms", $\approx_{\beta}$ refers to the β-equivalence relation, roughly speaking, there is a path between $a$ and $b$ composed of β-reduction and β-abstraction. His article further pointed out that any ω-consistent system cannot construct a decider. [11, 12]

If we want to follow Church's proof, then, we first need to arithmetize $\approx_{\beta}$. That is to say, we need to find a relation $P$ on integers such that $P(\sharp M, \sharp N)$ if and only if $M \approx_{\beta} N$, where $\sharp$ represents encoding. We obviously need $P$ to have some good representation, because we want to find a sentence $\mathsf{P(x, y)}$ in $\mathsf{PA}$ such that for any $n_1, n_2$

$$
\begin{aligned}
P(n_1, n_2) &\rightarrow \mathsf{PA \vdash P(n_1, n_2)} \\\\
\neg P(n_1, n_2)& \rightarrow \mathsf{PA \vdash \neg P(n_1, n_2)}
\end{aligned}
$$

The $\mathsf{P(n_1, n_2)}$ contains $\mathsf{n_1}$, which refers to the symbol string corresponding to the natural number $n_1$ in the meta-language in $\mathsf{PA}$, that is,

$$
\underbrace{S S S \cdots S}_{n_1 times}\ 0
$$

$\mathsf{P(x, y)}$ is a **determined**, $\mathsf{PA}$ sentence with two free variables. In this way:

- Assuming $\mathsf{PA}$ is decidable, for any $a, b \in \Lambda^{\circ}$,
- Input $\varphi = \mathsf{P(\sharp a, \sharp b)}$, $\psi = \mathsf{\neg P(\sharp a, \sharp b)} $ in the decider $\texttt{p}$,
- Return $\texttt{p}(\varphi)$

Obviously, this program has decided the problem that Church proved to be undecidable. In fact, if $P$ is representable in $T$, then the decision problem of $P$ can be reduced to the decidability of $T$.

However, I personally think it is a troublesome job to find $\mathsf{P}$.

### Using Kleene's results

In Gödel's 1931 paper, he proved that all primitive recursive functions are representable. Regarding the representability of functions, textbooks [1] and materials [9] have different opinions. Material [9] equates the representability of functions with the representability of relations. We temporarily adopt the latter's statement[^7], a function $f$ is representable if and only if there exists a $\mathsf{P}$, such that

$$
f(\vec{m}) = n \rightarrow T \vdash \mathsf{P(\vec{m}, n)} \\\\
f(\vec{m}) \neq n \rightarrow T \vdash \mathsf{\neg P(\vec{m}, n)}
$$

Kleene proved the normal form theorem:

> There exists a three-element primitive recursive function $C(x, y, z)$ and a one-element primitive recursive function $U(x)$, such that any partial recursive function $f$ can be represented as:
> $$
> f(n) = U(\mu z[C(e, n, z)])
> $$

This theorem seems quite puzzling. In fact, it describes a system similar to an interpreter:

```python
def U(x):
    # some calculation 
    pass

def C(e, y, z):
    # some calculation
    pass

def μ(f):
    n = 0
    while f(n) != 0:
        n += 1
    return n

def f(n):
    return n * 2

def f_another(n):
    # note: e may not be 10, we just use a fixed value here
    return U(μ(lambda z: C(10, n, n)))

# ∀ n, f(n) == f_anthor(n)
```

Here, $e$ can be roughly understood as the result after the program (recursive function) is encoded. Through this theorem, Kleene used the diagonal method to prove the following theorem:

> Not every partial recursive function is potentially recursive

The so-called "potentially recursive" means that for a partial recursive function $g$, there exists a total function $f$ such that $f(n) = g(n)$ where $g$ is defined. Intuitively, this theorem is similar to "not all semi-decidable sets are decidable".

Now suppose there is a decider $\texttt{p\\_t}$ for $T$. According to Gödel's proof, there is a formula $\mathsf{C(e, n, z, t)}$ that represents $C(e, n, z)$. Then, a new super interpreter can be constructed that can find the corresponding total function $f$ for all partial recursive functions $g$. Consider any $g$ with $e$ value $e_1$, construct $f$ as follows:

```python
def f(n):
    if p_t(∀ z. ¬C(e₁, n, z, 0)):
        # g(n) can never halt
        # return a random value, doesn't matter
        return 0
    else:
        # g(n) will halt
        # return g(n)
        return U(μ(lambda z: C(e₁, n, z)))
```

Kleene's theorem guarantees that the only way $g$ does not halt is $\forall z. C(e_1, n, z) \neq 0$. Therefore, since $C$ itself is a primitive recursive function that can be represented in $T$, once $T$ is decidable, whether $g(n)$ halts is decidable for any $n$.

## Comparison with Gödel's original proof

Gödel's original proof [1, 13] is quite complex and can be summarized as:

1. Prove that all primitive recursive functions are representable.
2. Arithmetize the syntax of first-order logic and elementary arithmetic with primitive recursive functions. We solve this by writing `can_prove_n(ψ, n)`; Gödel defines elementary recursive functions and corresponding elementary recursive relations. He wrote 45 recursive relations and finally defined provability (the famous Bew).
3. Since Bew is a primitive recursive relation, it is representable. In this way, a formula $\mathsf{P(\sharp\psi)}$ can be found to represent it.
4. Gödel proved the fixed-point theorem: for any unary formula $\psi(x)$, a sentence $\sigma$ can be found such that $T \vdash \sigma \leftrightarrow \psi(\mathsf{\sharp\sigma})$.
5. Therefore, there exists $\sigma$ such that $T \vdash \sigma \leftrightarrow \neg \mathsf{P(\sharp \sigma)}$. Note that this sentence constructs a proposition similar to the liar paradox.
6. Using ω-completeness, whether assuming $T \vdash \sigma$ or $T \vdash \neg \sigma$, a contradiction will be obtained. In this way, $\sigma$ and $T$ are independent.

As can be seen, our proof did not avoid 1. and 2., which are the most complex steps. However, semi-decidability is relatively simple to understand, and if it is brought up at the moment, the proof will be very short. 1. and 2. are actually introduced to prove semi-decidability.

## Further Reading

- If you want to learn Gödel's original proof, please read [1] or Gödel's original paper [13]. Professor Yang Ruizhi's book is very well written.
- The idea of this proof seems to originate from Kleene, you can refer to his work [10] and these notes [9].
- This article [4] also introduces the incompleteness theorem with this idea.
- Professor ocau's Agda formalization is worth referring to, but his lecture videos are not public (for unknown reasons). His idea may come from this Coq formalization work [6], you can read this paper as a substitute.
- A complete Coq formalization of Gödel's incompleteness theorem [3] (the above paper is comprehensive and has a big difference from what is discussed here)
- There are two books about Gödel's incompleteness theorem [2, 8] that are quite good, you can choose to read some chapters.
- For knowledge related to recursive functions, you can refer to [1, 10] and Professor Song's [7]. Similarly, concepts related to computability can refer to [5]

## References

- [1] [Mathematical Logic: Proof and Its Limits](https://book.douban.com/subject/35210053/)
- [2] [An Introduction to Gödel's Theorems](https://www.karlin.mff.cuni.cz/~krajicek/smith.pdf)
- [3] [Essential Incompleteness of Arithmetic Verified by Coq](http://r6.ca/Goedel/goedel1.html)
- [4] [Gödel's Incompleteness Phenomenon—Computationally](https://www.cairn.info/revue-philosophia-scientiae-2014-3-page-23.htm)
- [5] [Computability and Complexity: From a Programming Perspective.](http://hjemmesider.diku.dk/~neil/comp2book2007/book-whole.pdf)
- [6] [Gödel’s Theorem Without Tears: Essential Incompleteness in Synthetic Computability](https://drops.dagstuhl.de/opus/volltexte/2023/17491/pdf/LIPIcs-CSL-2023-30.pdf)
- [7] [Guide to Computational Models](https://book.douban.com/subject/10785250/)
- [8] [Incompleteness and Computability: An Open Introduction to Gödel's Theorems](https://ic.openlogicproject.org/ic-screen.pdf)
- [9] [Kleene’s proof of Gödel's Theorem](https://www.logicmatters.net/resources/pdfs/KleeneProof.pdf)
- [10] [Introduction to Metamathematics](https://archive.org/details/BubliothecaMathematicaStephenColeKleeneIntroductionToMetamathematicsWoltersNoordhoffPublishing1971)
- [11] [An Unsolvable Problem of Elementary Number Theory](https://www.ics.uci.edu/~lopes/teaching/inf212W12/readings/church.pdf)
- [12] [A. Church on Computability](https://plato.stanford.edu/entries/church/supplementA.html)
- [13] [On Formally Undecidable Propositions of Principia Mathematica and Related Systems I](https://monoskop.org/images/9/93/Kurt_G%C3%B6del_On_Formally_Undecidable_Propositions_of_Principia_Mathematica_and_Related_Systems_1992.pdf)

[^1]: The proposition here is a proposition in the "meta-language", not the language we are discussing, that is, the "object language", here is the proposition of first-order logic. Do not confuse.
[^2]: Of course, in the Hilbert System, there are also "logical axioms", that is, axioms that hold in any first-order theory. These axioms can also be used as the starting point of reasoning, and any logic that holds the "tautology" can be derived.
[^3]: The original narrative is more complicated. The "elementary arithmetic" here is weaker than Peano, which involves the concept of essential incompleteness.
[^4]: In other words, a new proposition can be found. This new proposition is independent of the system after the axiom is added.
[^5]: The so-called "set" here is generally a subset of the program data set, and recursion theory generally considers it a subset of natural numbers.
[^7]: The statement in [1] seems to be correct, but it gives a stronger proposition. We only need this weaker form here.
