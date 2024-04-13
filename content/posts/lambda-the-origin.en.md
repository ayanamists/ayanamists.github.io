---
title: "Untold Stories -- Church and His System"
date: 2024-03-15T22:29:00+08:00
draft: false
categories:
  - Computer History
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

As a functional programming enthusiast, you must know Church and his $\lambda$ calculus. If you have studied recursion theory, you may also know about $\lambda$ definability, various properties of $\lambda$ calculus (Church-Rosser theorem, fixed point theorem...), and how they are combined to give an undecidable proof of $\beta$ equivalence.

Of course, you may not know what these words are talking about, and that's okay. If you have studied a little bit of computer science, you must know about the Turing Machine. Some people regard Turing and his paper as the origin of computer science, but in fact, Church and his $\lambda$ calculus are earlier and more elegant[^2]. The story we are going to tell today is actually one of the origins of theoretical computer science.

Few people know why Church invented $\lambda$ calculus at that time, and what is different between the system he invented at that time and the $\lambda$ calculus we know now. In fact, Church's system is much more complicated than today's $\lambda$ calculus. Why was Church's system forgotten, and today's $\lambda$ calculus has become the cornerstone of computer science?

This is a very interesting story. I shared this story with my classmates in the SICP course in 2023[^3], and now, I think it should be written down to let more people know about this history.

{{<  admonition "note" >}}
This article has some technical details, but the important thing is the story. If you don't understand, you can skip it, it won't affect the understanding of the story. However, I suggest at least understanding the content of the first section, so you know what Church is roughly doing.
{{< /admonition >}}


## Formal Systems

The system that Church published in 1932[^1] is now called a "formal system". Simply put, a formal system has three elements:

- Syntax - A set of symbols and rules to describe legal expressions (formula). The "syntax" here is no different from the "syntax" of a programming language, they both specify what the expressions in this system look like.
- A set of axioms - A set of expressions that are considered true. These axioms are the foundation of the system, and other theorems are derived from these axioms.
- Inference rules - A set of rules used to derive new theorems from known axioms and theorems.

I can propose an extremely simple formal system (denoted as $P$):

- Syntax - Strings composed of $a$ and $b$. Such as $a$, $aa$, $ab$, $ba$, etc.
- Axioms - $a$, $b$
- Inference rules
  - (I.) If $x$ is a theorem, then $axa$ and $bxb$ are also theorems.

All the theorems of this system are the so-called "palindromes". For example, $aba$ is a theorem, and its derivation process (we call it "proof") is:

- $b$   (Axiom)
- $aba$ (I.)

In logic, "$aba$ is a theorem of system $P$" can be written as $P \vdash aba$. Here $\vdash$ means "derives".

Conversely, $ab$ is not a theorem because we cannot derive it through the inference rules. This can be written as $P \nvdash ab$.

The "axioms", "theorems", and "proofs" mentioned here are all terms of formal systems. They have precise mathematical meanings. For example, here, "axioms" are $a$ and $b$, "theorems" are palindromes, and "proofs" are sequences of strings that satisfy the inference rules. However, obviously, we have been receiving mathematical education from a young age, and "axioms", "theorems", and "proofs" have natural language meanings. Please allow me to take some time to explain their relationship.

The "axiom" in natural language, we will represent it as _axiom_ below, and the "axiom" in the formal system, we represent it as `axiom`. We do similar treatment for other concepts.

We have learned various _theorems_ from a young age, such as $1 + 1 = 2$. These _theorems_ were mostly _proven_ before modern logic was established. Naturally, before logic, there were concepts such as _proof_ and _theorem_. This mathematics described and constructed through natural language, we mostly consider it to be "existing" and correct.

And the `theorem` in the formal system, as we have said, is a strict mathematical object. We know its properties, we know how it is derived from axioms. This derivation process can be verified by a computer. In short, these mathematical objects are clear and strict. And the mathematics of natural language, at least does not have such strictness. For example, can you imagine a program that can easily verify all the proofs in a randomly selected mathematics textbook?

So in my opinion, the primary goal of the formal system is to "solidify" the mathematics of natural language with strict mathematical objects. "Solidify", to express it in a more academic term, is called "formalize".

The mathematics described by natural language and the mathematics described by the formal system are like algorithms and programs. An algorithm is a non-formal program, and a program is a formalized algorithm.

Formalizing mathematical proofs in natural language with formal systems is a very difficult problem. To solve this problem, we need more complex and troublesome formal systems. So, what kind of formal system should we choose?

For this difficult problem, most mathematicians believe that another discipline developed in the second half of the 19th century--set theory--is the answer. Most mathematical objects can be described by set theory, and set theory can be described by formal systems. _Principia Mathematica_ published by Russell from 1910 to 1913, and Zermelo-Fraenkel set theory (1908, 1922), are practices of this scheme. ZF set theory has basically become the foundation of modern mathematics.

## Church's System

In the 1920s, Church tried to provide a formal system using a non-set theory method. In his 1932 paper, Church said that he felt that both Russell's _Principia Mathematica_ and ZF set theory had adopted "unnatural" methods to deal with the famous "Russell's Paradox".

His method is to "restrict the use of the law of excluded middle". Technically speaking, I personally feel that his method is even more "unnatural", so that I find it difficult to give a clear explanation within the scope of this article. In short, he designed a formal system with 37 (yes, you read that right, just 37) axioms and 5 inference rules. These 5 inference rules led to modern $\lambda$ calculus. This system was given in _A Set of Postulates for the Foundation of Logic_ published in 1932.

In Church's system, there are indeed no sets or classes in the syntax, which are composed of these symbols:

> 5. Undefined terms. We are now ready to set down a list of the undefined terms of our formal logic. They are as follows:
$$
\\{\\}(),\\ \lambda\[\],\\ \Pi,\ \Sigma,\ \\&, \\ \sim, \\ \iota, \\ A
$$

Among them, $\\{\\}()$ represents the "action" (apply, commonly known as function call) in modern $\lambda$ calculus, $\lambda [\ ]$ is naturally $\lambda$ abstraction, $\Pi$ is a more complex syntax, $\Pi(P, Q)$ is roughly $\forall x. (P(x) \to Q(x))$[^5], $\Sigma$ is existence, $\\&$ is logical and, $\sim$ is logical not, $\iota$ is similar to the $\mu$ operator in recursive functions, $\iota(F)$ refers to "let $\\{F\\}(x)$" hold $x$". As for $A$, this symbol is particularly abstract, and will not be discussed here.

Afterwards, Church defined some syntactic sugar, or "macros". For example, he defined $V$, which is $\lor$ in modern logic, with the following definition:

$$
V(P, Q) \equiv \sim . \sim P . \sim Q
$$

Expressed in today's symbols, it would be:

$$
P \lor Q \equiv \neg (\neg P \land \neg Q)
$$

It's worth mentioning that Church used the "dot notation" popular at the time and adopted by Russell to represent logical operations. Simply put, the dot determines the "priority", which can reduce the use of parentheses. For example, $\sim . \sim P . \sim Q$ is a shorthand for $\sim (\sim P \land \sim Q)$. However, as far as I know, the dot symbol in Principia Mathematica does not seem to allow this usage, and this proposition in PM should be $\sim (\sim P . \sim Q)$. (Of course, this could be my misunderstanding, I referred to [The Notation in Principia Mathematica](https://plato.stanford.edu/entries/pm-notation/#Exam)).

In addition, Church also defined the implication symbol $\supset$ as a macro.

In summary, this is roughly the syntax of Church's system. His 37 axioms are more complex. The 15th one might be easier to understand:

$$
\text{15.}\ pq \supset_{pq} q 
$$

This refers to $p \land q \to q$. But most of the axioms are difficult to understand. For example, the first one:

$$
\text{1.}\ \Sigma(\varphi) \supset_{\varphi} \Pi(\varphi, \varphi)
$$

$\Pi(\varphi, \varphi)$ should hold without any premise (understood as $\forall x. (\varphi(x) \to \varphi(x))$), but due to Church's restrictions, $\Sigma(\varphi)$ must be ensured here.

Compared to the 37 incomprehensible axioms, his 5 inference rules seem very simple to us today. These 5 rules are quite similar to modern $\lambda$ calculus. These 5 rules (with some of my rephrasing, but no difference from Church's original text) are:

**I.** In a true formula $J$, if 

- All appearances of $x$ in $J$ are bound appearances
- $y$ does not appear freely in $J$,

Then, in $J$, the result $K$ of replacing $x$ with $y$ (denoted as $\text{S}_{y}^{x}\ J$) is also a true formula.

**II.** If 

- $(\lambda x. M) N$ is a sub-formula of true formula $J$
- $x$ does not appear bound in $M$
- $x$ appears freely in $M$ (*)
- $x$ does not appear freely in $N$

Then, in $J$, the result $K$ of replacing a specific $(\lambda x. M) N$ with $\text{S}_{N}^{x}\ M$ is also a true formula.

**III.** If

- $\text{S}_{N}^{x}\ M$ is a sub-formula of true formula $J$
- $x$ does not appear bound in $M$
- $x$ appears freely in $M$ (*)
- $x$ does not appear freely in $N$

Then, in $J$, the result $K$ of replacing a specific $\text{S}_{N}^{x}\ M$ with $(\lambda x. M) N$ is also a true formula.

**IV.** If $F(A)$, then $\Sigma(F)$.

**V.** If $\Pi(F, G)$, and $F(A)$, then $G(A)$.

These 5 rules, **IV.** and **V.** are similar to the "existential quantifier introduction" and modus ponens of first-order logic. The highlights are **I.**, **II.** and **III.**:

- **I.** is $\alpha$ equivalence
- **II.** is one direction of $\beta$ equivalence, which can be regarded as $\beta$ reduction
- **III.** is the other direction of $\beta$ equivalence, which can be regarded as $\beta$ abstraction

These three rules constitute the core of modern $\lambda$ calculus. It is worth mentioning the rule marked with (*), which is not in the modern $\lambda$ calculus. We will not discuss this matter for now, if you are interested, you can think about it yourself.

## Kleene and $\lambda$ Definability

I don't know how much you understand about these messy symbols above. If you showed it to me a few years ago, I would definitely be confused. Moreover, Church's system has 37 axioms, and it's overwhelming to think about proving in this system. Compared with ZF set theory, Church's system is obviously overly complicated. Moreover, his approach of directly bypassing first-order logic adds some difficulty to learning.

Fortunately, Church had a student named Kleene.

Kleene came to Princeton University to read Church's doctorate when he graduated from undergraduate at the age of 22 in 1931. Amazingly, he received his doctorate in 1934 at the age of 25. I have to exclaim that he is truly a "genius, super genius" [^6]!

In the fall semester of 1931, Church told Kleene and other students about his new invention--that is, the system we mentioned above. From Kleene's recollection, Church had the idea of $\beta$ reduction at that time. What is $\beta$ reduction? Simply put, it is to regard $(\lambda x. M) N$ as a function call and replace it with $\text{S}_{N}^{x} M$. In other words, it is to derive a formula in his system using axiom **II.**.

Importantly, this derivation can be seen as a "calculation". If given a $\lambda$ term $a$, any number of $\beta$ reductions are performed to get $b$, this can be seen as the "calculation" of $\lambda$ calculus. In fact, this "calculation" also defines a relationship, $a \rightarrow_{\beta} b$, which means that $a$ can be obtained through $\beta$ reduction to $b$.

Church also mentioned in his lecture at that time that he had found a way to define positive integers in his system, that is, the Church numbers we talk about today. $1, 2, 3$ are defined as:

$$
\mathsf{1} \equiv (\lambda f. \lambda x. f x) \\\\
\mathsf{2} \equiv (\lambda f. \lambda x. f (f x)) \\\\
\mathsf{3} \equiv(\lambda f. \lambda x. f (f (f x)))
$$

Combined with the idea of $a \to_{\beta} b$ above, Church found that some functions can be represented by $\lambda$ terms, such as the successor function $S(n) = n + 1$, which can be represented by $\lambda$ terms as:

$$
\mathsf{S} \equiv \lambda n. \lambda f. \lambda x. f (n f x)
$$

It has the property $\mathsf{S}\ \mathsf{1} \to_{\beta} \mathsf{2}$, more generally, Church proposed the concept of $\lambda$ definability. His idea is that if a function $f$ can be represented by a $\lambda$ term, then $f$ is $\lambda$ definable.

Formally speaking, for a (partial) function $f$, if there exists a $\lambda$ term $\mathsf{F}$, for natural numbers $n$ in the domain of $f$, there is

$$
\mathsf{F}\ \mathsf{n} \to_{\beta} \mathsf{f(n)}
$$

Then $f$ is $\lambda$ definable. The above $\mathsf{f(n)}$ refers to the Church number corresponding to $f(n)$ (a natural number).

Church discovered that not only the successor function, but also the addition function $f(x, y) = x + y$ is $\lambda$ definable. However, Church did not know whether the "predecessor" function ($P(n) = n - 1$) is $\lambda$ definable. Of course, his system allows him to "cheat", and he can get the predecessor function without $\lambda$ definability. His definition is as follows:

$$
\lambda r. \iota x. (\mathsf{S}\ x = r)
$$

The definition of this $\iota$ was seen in the previous section, simply put, Church defines the predecessor of $r$ through "the $x$ that satisfies $\mathsf{S}\ x = r$".

In early 1932, Church asked Kleene to try to prove the axioms of natural numbers (Peano Axioms) in his system. Kleene found that there must be a predecessor function here, and he was not particularly satisfied with Church's "cheating", so he began to study whether the predecessor function is $\lambda$ definable.

In Kleene's recollection (see [Kleene 1981]), he initially wanted to change the definition of natural numbers a bit, but Church still wanted to use his Church numbers. So, he racked his brains to think about how to give a predecessor function. Finally, at the end of January or the beginning of February 1932, Kleene suddenly thought of a way to solve the problem of $\lambda$ defining the predecessor function while at the dentist's office. Then, Kleene gave a $\lambda$ term that $\lambda$ defined the predecessor function. I won't show the specific definition here, interested students can think about it themselves.

When Kleene told Church about his invention, Church said that he was about to give up:

> When I brought this result to Church, he told me that he had just about convinced himself that there is no $\lambda$-definition of the predecessor function.

A function originally thought to be undefinable by $\lambda$ was proven to be definable by Kleene. Kleene and Church began to ponder a new question:

> What kind of function is $\lambda$ definable?

To this question, 90 years later today, we blurt out: **A function $f$ is $\lambda$ definable if and only if $f$ is computable**.

At that time, Kleene's main job was to prove some mathematical propositions in Church's system. His side job was to study $\lambda$ definability. He found that almost all intuitively computable natural number functions are $\lambda$ definable. In 1933, Gödel came to Princeton. In 1934, he reported a method for defining recursive functions, which is now the so-called Herbrand-Gödel computable function. Kleene saw this report and realized that this method was related to his $\lambda$ definability.

Eventually, in the papers published by Kleene and Church in 1935-36, a series of problems were proven:

- $\approx_{\beta}$, that is, $\beta$ equivalence, is $\lambda$ undefinable
- $\lambda$ definable functions and Gödel computable functions are equivalent
  
The rest of the story is all too familiar. Turing published his paper in 1936, proposed the Turing Machine, and proved that the Turing Machine and Church's $\lambda$ calculus are equivalent. Theoretical computer science was born in this way.

Unfortunately, many people at the time felt that $\lambda$ calculus was not very "mathematical", so Kleene's subsequent great work was all based on his defined $\mu$-recursive functions; and when it comes to "computing" models, everyone prefers Turing machines.

## How Church's system was forgotten

The study of $\lambda$ definability opened the door to modern computer science, but what about Church's system?

Kleene, in his memoirs, said somewhat playfully:

> This paper contains Gödel’s celebrated proof of the existence of undecidable sentences in formal systems embodying the usual elementary number theory, and his “second theorem” on the impossibility of a proof of the consistency of such a system within the system itself. Church’s immediate reaction was that his formal system, about which I am going to say more, is sufficiently different from the systems Gödel dealt with that Gödel’s second theorem might not apply to it (see Church 1933 top p. 843). Indeed, Church was right!

Church seemed to think that his system could escape Gödel's limitations (we now know this is impossible). And Kleene actually agreed? What's going on?

Oh, Kleene's next sentence explains the reason:

> In his system there is a proof of its own consistency, since in fact it is inconsistent (so all its sentences are provable), as Church had thought possible (1933 top p. 842) and as Rosser and I showed later (Kleene and Rosser 1935).

Kleene, is this how you joke about your teacher's system? This has a bit of a "hell joke" flavor. Yes, Church's system did indeed surpass Gödel's limitations, because it is inconsistent!

The so-called "inconsistency" means that this system $T$ can prove a proposition $P$, and at the same time it can prove $\neg P$. This is already bad intuitively: this system has some kind of "contradiction". Technically it's even worse, because it means that this system can prove any proposition. Suppose we have $T \vdash P$ and $T \vdash \neg P$, to prove an arbitrary proposition $Q$, we can make the following derivation:

1. $P$
2. $P \lor Q$
3. $\neg P$
4. $Q$

The reasoning rules used in this derivation are the introduction and elimination of "or", almost any formal system has this rule. So, if a system is inconsistent, then it can prove any proposition. Generally, we think such a system is meaningless.

Kleene and another student of Church, Rosser, proved in 1935 that Church's system was inconsistent. Their proof is called the "Kleene-Rosser paradox". This paradox is technically complex, but intuitively, Church's system can already encode any computable function, which strongly implies inconsistency.

In a 1984 article written by Rosser, he also introduced how to use the technique invented by Curry to prove the inconsistency of Church's system, which is a simple proof.

In fact, in the same-named (and 1932 same-named) paper published by Church in 1933, he found that his system was inconsistent. To resolve this inconsistency, he deleted axiom 19 and added two axioms, bringing the total number of axioms to 38. He thought this new system was consistent, but we now know that this system is also inconsistent.

So, in short, because Church's system was inconsistent and extremely complex, it was forgotten.

## Conclusion

Church's initial goal was to formalize mathematics with his system, but this system is now long forgotten; and a by-product of his system, $\lambda$ calculus, has become the cornerstone of computer science. These great names, Church, Kleene, Turing, Gödel, ..., are forever in the history of computer science.

Their work, in general, should still be called "recursion theory". Especially many of Kleene's studies are studying the relationship between uncomputable sets and their relationships. However, I still feel that if a subject called "computational theory" does not talk about $\lambda$ calculus, recursive functions, but talks about automata, it really lacks a lot of interesting things. If your "computational theory" is also a bunch of automata, I hope this article can make you interested in $\lambda$ calculus.

## Quiz

1. Church numbers were first published in Church's 1933 paper, which also gave the expression of Peano's axioms in his system. However, whether it is his expressed Church numbers or Peano's axioms, they all start from $1$, not from $0$. Can you think about why Church didn't put $0$ in his system? This is not just his taste, but more of a technical issue.
2. In modern $\lambda$ calculus, we generally define the constructor of (binary) tuples as
   $$(a, b) \equiv \lambda a \lambda b \lambda f. f a b$$
   fst and snd are defined as
   $$
   \begin{aligned}
   \text{fst} &\equiv \lambda p. p (\lambda a \lambda b. a) \\\\ 
   \text{snd} &\equiv \lambda p. p (\lambda a \lambda b. b)
   \end{aligned}
   $$
   The definition of tuples is an important step in Kleene's definition of the predecessor function, but can Kleene define tuples like this? Why? If not, can you give a definition of tuples that Kleene can use? (Hint: See [Kleene 1981] for the answer)
3. If it is said that Church's original system is inconsistent, then why is $\lambda$ calculus itself not affected by this problem? Wouldn't anyone think that $\lambda$ calculus would also be "inconsistent"?

## Further Reading

1. Kleene, S. C. (1981). Origins of recursive function theory. Annals of the History of Computing, 3(1), 52-67.
2. Church, A. (1933). A set of postulates for the foundation of logic. Annals of Mathematics, 34(4), 839-864.
3. Church A. (1936). An unsolvable problem of elementary number theory. American Journal of Mathematics, 58(2), 345-363.
4. Church A. (1941). The Calculi of Lambda-Conversion. Princeton University Press.
5. Church A. (1932). A set of postulates for the foundation of logic. Annals of Mathematics, 33(2), 346-366.
6. Rosser J. B. (1984). Highlights of the history of the lambda calculus. Annals of the History of Computing, 6(4), 337-349.

## Appendix: Church's 37 Axioms

Thanks to deep learning technology, I was able to simply extract the latex code of Church's 37 axioms.

1. $\boldsymbol{\Sigma}(\varphi) \supset_{\varphi} \Pi(\varphi, \varphi)$.
2. ' $x . \  \varphi(x) \supset_{\varphi} . \  \Pi(\varphi, \psi) \supset_\psi \psi(x)$.
3. $\mathbf{\Sigma}(\sigma) \supset_\sigma . \ \left[\sigma(x) \supset_x \varphi(x)\right] \supset_p . \  \Pi(\varphi, \psi) \supset_\psi . \  \sigma(x) \supset_x \psi(x)$.
4. $\mathbf{\Sigma}(\varrho) \supset_{\varrho} . \Sigma y\left[\varrho(x) \supset_x \varphi(x, y)\right] \supset_p . \ \left[\varrho(x) \supset_x \Pi(\varphi(x), \psi(x))\right] \supset_\psi$. $\left[\varrho(x) \supset_x \varphi(x, y)\right] \supset_y . \  \varrho(x) \supset_x \psi(x, y)$.
5. $\Sigma(\varsigma) \supset_{\varphi} . \  \Pi(\varphi, \psi) \supset_\psi . \  \varphi(f(x)) \supset_{f x} \psi(f(x))$.
6. ' $x . \  \varphi(x) \supset_p . \  \Pi(\varsigma, \psi(x)) \supset_\psi \psi(x, x)$.
7. $\varphi(x, f(x)) \supset_{q f x} . \  \Pi(\varphi(x), \psi(x)) \supset_\psi \psi(x, f(x))$.
8. $\mathbf{\Sigma}(\varrho) \supset_{\varrho} . \Sigma y\left[\varrho(x) \supset_x \varphi(x, y)\right] \supset_p . \ \left[\varrho(x) \supset_x \Pi(\varphi(x), \psi)\right] \supset_\psi$. $\left[\varrho(x) \supset_x \varphi(x, y)\right] \supset_y \psi(y)$.
9. ' $x . \  \varphi(x) \supset_{\varphi} \Sigma(\varphi)$.
10. $\Sigma x \varphi(f(x)) \supset_{f \varphi} \Sigma(\varphi)$.
11. $\varphi(x, x) \supset_{\varphi x} \Sigma(\varphi(x))$.
12. $\Sigma(\varphi) \supset_{\varphi} \Sigma x \varphi(x)$.
13. $\Sigma(\varphi) \supset_{\varphi} . \ \left[\varphi(x) \supset_x \psi(x)\right] \supset_\psi \boldsymbol{\Pi}(\varphi, \psi)$.
14. $p \supset_p . \  q \supset_q p q$.
15. $p q \supset_{q p} p$.
16. $p q \supset_{p q} q$.
17. $\Sigma x \Sigma \theta[\varphi(x) . \  \sim \theta(x) . \  \Pi(\psi, \theta)] \supset_{p \psi} \sim \Pi(\varphi, \psi)$.
18. $\sim \Pi(\varphi, \psi) \supset_{\varphi \psi} \Sigma x \Sigma \theta . \  \varphi(x) . \  \sim \theta(x) . \  \Pi(\psi, \theta)$.
19. $\Sigma x \Sigma \theta\left[\sim \varphi(u, x) . \sim \theta(u) . \Sigma(\varphi(y)) \supset_y \theta(y)\right] \supset_{\varphi u} \sim \Sigma(\varphi(u))$.
20. $\sim \boldsymbol{\Sigma}(\varphi) \supset_{\varphi} \boldsymbol{\Sigma} x \sim \varphi(x)$.
21. $p \supset_p \sim q \supset_q \sim p q$.
22. $\sim p \supset_p . \  q \supset_q \sim . \  p q$.
23. $\sim p \supset_p \sim q \supset_q \sim p q$.
24. $p \supset_p . \ [\sim . \  p q] \supset_q \sim q$.
25. $[\sim . \varphi(u) . \  \psi(u)] \supset_{\varphi \psi u} . \ \left[\left[[\varphi(x) . \  \sim \psi(x)] \supset_x \varrho(x)\right]\right.$.
$\left.\left[[\sim \varphi(x) . \  \psi(x)] \supset_x \varrho(x)\right] . \ [\sim \varphi(x) . \  \sim \psi(x)] \supset_x \varrho(x)\right] \supset_{\varrho} \varrho(u)$.
26. $p \supset_p \sim \sim p$.
27. $\sim \sim p \supset_p p$.
28. $\sim \Sigma(\varphi) \supset_{\varphi} . \  \Sigma(\psi) \supset_\psi \Pi(\varphi, \psi)$.
29. $\sim \boldsymbol{\Sigma}(\varphi) \supset_{\varphi} \sim \boldsymbol{\Sigma}(\psi) \supset_\psi \boldsymbol{\Pi}(\varphi, \psi)$.
30. ' $x . \  \varphi(x) \supset_{\varphi} . \ \left[\theta(x) . \  \psi(x) \supset_\psi \Pi(\theta, \psi)\right] \supset_\theta \varphi(\iota(\theta))$.
31. $\varphi(\iota(\theta)) \supset_{\theta \varphi} \boldsymbol{\Pi}(\theta, \varphi)$.
32. $E(\iota(\theta)) \supset_\theta \Sigma(\theta)$.
33. $\left[\varphi(x, y) \supset_{x y} . \  \varphi(y, z) \supset_z \varphi(x, z)\right]\left[\varphi(x, y) \supset_{x y} \varphi(y, x)\right] \supset_{\varphi} . \  \varphi(u, v) \supset_{u v}$. $\psi(A(\varphi, u)) \supset_\psi \psi(A(\varphi, v))$.
34. $\left[\varphi(x, y) \supset_{x y} . \  \varphi(y, z) \supset_z \varphi(x, z)\right]\left[\varphi(x, y) \supset_{x y} \varphi(y, x)\right] \supset_{\varphi}$. $\left[\varphi(x, y) \supset_{x y} \theta(x, y)\right] \supset_\theta \sim \theta(u, v) \supset_{u v} \sim . \psi(A(\varphi, u)) \supset_\psi \psi(A(\varphi, v))$.
35. $\left[\psi(A(\varphi, u)) \supset_\psi \psi(A(\varphi, v))\right] \supset_{\varphi u v} \varphi(u, v)$.
36. $E(A(\varphi)) \supset_{\varphi} . \  \varphi(x, y) \supset_{x y} . \  \varphi(y, z) \supset_z \varphi(x, z)$.
37. $E(A(\varphi)) \supset_{\varphi} \varphi(x, y) \supset_{x y} \varphi(y, x)$.

[^1]: I suggest not calling it the $\lambda$ system, because the $\lambda$ system often refers to the formal system that constructs $\lambda$ equivalence, not Church's original system.
[^2]: This is my personal opinion. However, both in terms of syntax and semantics, $\lambda$ calculus is much simpler than Turing machines.
[^3]: Nanjing University's "Construction and Interpretation of Computer Programs" course.
[^5]: In fact, Church's paper emphasizes the difference between $\Pi(P, Q)$ and $\forall x. P(x) \to Q(x)$, which is related to his "restriction on the law of the excluded middle". However, this detail is beyond the scope of this article.
[^6]: http://dangshi.people.com.cn/n/2014/0415/c85037-24898922-2.html
