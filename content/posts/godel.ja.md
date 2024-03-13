---
title: "マルチスレッドでゲーデルの不完全性を証明するのは間違っているだろうか"
date: 2023-09-01T12:34:12+08:00
draft: false
tags:
    - 数理論理学
    - 計算可能性
categories:
    - 学術
---

{{<  admonition "warning" >}}
この記事はChatGPTによって中国語から翻訳されたもので、いくつかの誤りが含まれているかもしれません。不正確な部分があればご了承ください。
{{< /admonition >}}

{{<  admonition "warning" >}}
この記事は厳密な学術論文ではなく、一部誤りや欠落があるかもしれません。ご了承ください。
{{< /admonition >}}

## 序論

Kurt Gödelのいくつかの有名な定理は、現代の論理学の基礎を形成しています。中でも、Gödelの（第一および第二の）不完全性定理は広く称賛されています。

これまで、私を含む多くの人々は、Gödelの不完全性定理の証明は非常に複雑で、非常に複雑な技術的詳細があるため、学習するのはかなり困難だと感じてきました。2023年の[∞-type Café 夏季学校](https://m4p1e.github.io/ntype-cafe-summer-school/)で、ocau先生は非常に優雅で簡潔なGödelの不完全性定理の証明の講義を行いました。この講義により、Gödelの不完全性定理は実際には計算可能性、あるいは[再帰理論](https://plato.stanford.edu/entries/recursive-functions/)と非常に密接な関係があることに気づきました。

この記事では、私がGödelの不完全性定理について理解したことを基に、最も簡単な（おそらく厳密ではない）Gödelの不完全性定理の証明を紹介し、一般的なプログラマーが簡単な一階論理を学んだ後でも理解できるように努めます。

## 一階論理

一階論理（First order logic, FOL）は、本記事で議論する論理システムです。一階論理には、構文面と意味面があります。一階論理を学校で学んでいない読者は、以下のように理解することができます：

1. 一階論理は一種の言語であり、私たちがよく見るプログラミング言語と似ています。それは形式化された構文（Syntax）を持っています。
    - 一階論理の「文」は、抽象構文木（Abstract Syntax Tree）または代数データ型（ADT）と見なすことができます。
    - この構文木には二つのレベルがあります。一つは「命題」（Statement）、もう一つは「表現」（Expression）です。
    - 表現レベルには変数、関数（呼び出し）などが含まれます。$x$、$f(x)$、$g(x, y)$ はすべて表現です。
    - 命題レベルは表現から構成され、表現は等号や関係でつながれて命題を構成します。例えば、$f(x_1) = f(x_2)$、$P(x_1, f(x_2))$ などです。
    - 命題は論理接続詞でつながれ、特定の意味を表現します。例えば、$f(x_1) = f(x_2) \land P(x_1, f(x_2))$ も命題です。このような論理接続詞には $\rightarrow$、$\lor$、$\neg$ などがあります。
    - 命題内の自由変数（free variable）は量化子で束縛され、新たな命題を得ることができます。例えば、$\forall x_1 \exists x_2. f(x_1) = f(x_2)$ などです。
2. 一階論理という言語は、その意味を規定する必要があります。これはモデル（Model）を通じて実現されます。関連する内容は比較的複雑なので、ここでは議論しません。
3. 一階論理には多くの証明システムがあります。例えば、[Hilbert System](https://en.wikipedia.org/wiki/Hilbert_system)などです。これらの証明システムは意味とは独立しています。簡単に言えば、証明システムは「計算」によって $\Gamma \vdash \psi$ という命題を提供します[^1]。ここで、$\Gamma$ は命題の集合を表し、$\psi$ は証明する必要がある命題を表します。
4. 一階論理を使用するとき、私たちは一階理論（Theory）を使用します。一階理論は、記号表 $S$ と公理集合 $T$ から構成されます。記号表は、命題中に出現可能なすべての記号を規定します。これには、定数、関数記号、関係記号の三つが含まれます。公理集合は、この理論中で自明であると証明されない命題を規定します。これらは証明システムの証明の出発点となります[^2]。例えば、有名なPeano算術は一階理論として扱うことができ、その記号表には以下のものが含まれます：
    - 定数記号 $0$
    - 関数記号 $+$
    - 関数記号 $\times$
    - 関数記号 $S$（後続を表す）
    - 関係記号 $\le$

一階論理への紹介はここまでです。一階論理は非常に豊かな数学的性質を持っており、これが数理論理学の主要な議論の対象です。もし読者がこの記事を読んで不快な思いをしたなら、信頼できる数理論理学の教科書、例えば [1, 2] などを探して、関連知識を学んでみてください。

## ゲーデルの不完全性定理

"完全性"は一階理論の性質です。一階理論 $T$ が完全であるとは、以下のことを意味します：

> その理論の記号集合で構築された任意の命題 $\psi$ に対して、$T \vdash \psi$ または $T \vdash \neg \psi$ のどちらかが成り立つ

つまり、この理論内の任意の命題は、証明されるか、否定されるかのどちらかです。

これは純粋な構文的な概念であり、特定の証明システムを指定する必要があるように思えます。しかし、特別な構造がなければ、一階論理の各証明システムは証明能力において等価です。そして、ゲーデルの完全性定理が存在するため、$T \vDash \psi$ は $T \vdash \psi$ と同等であり、これは実際には意味論的な概念でもあります。したがって、通常は証明システムを指定する必要はありません。ゲーデルの不完全性定理の形式化 [3] では、このような指定が必要になる可能性があります。

一階理論については、それが完全であることを期待します。例えば、ペアノ算術では、以下の直感的な命題が証明されます：

- $\forall n. n \le S(n) $
- $\forall n_1, n_2 . n_1 + n_2 = n_2 + n_1$

それに対応して、直感的には成り立たない命題は否定されます。

一階理論が十分に単純であれば、不完全性は容易に達成されます。例えば、記号表が $\\{ c, e \\}$ で、公理が $\emptyset$ の一階理論 $A$ を考えてみましょう。明らかに以下が成り立ちます：

- $A \not\vdash c = e$
- $A \not\vdash c \neq e$

直感的には、これは理論の公理集合が少なすぎるためです。$A$ に公理（これは通常 "拡張"（extension）と呼ばれます）$c = e$ を追加すると、$A$ を完全な理論にすることができます。

任意の理論 $T$ について、$\psi$ がこの理論で証明も否定もできない場合（これを $\psi$ が $T$ に独立であると言います）、$\psi$ または $\neg \psi$ を $T$ の公理に追加することで、この不完全性を解消することができます。

これは美しい風景を示しているように思えます：私たちは全ての数学を完全な一階理論に基づいて構築することができ、この究極の理論では、任意の命題は証明されるか否定されるかのどちらかです。これにより、曖昧で混乱した数学の言語は、明確で整然とした一階論理に完全に置き換えられることになります……

残念ながら、1931年、Gödelは彼の論文 "Über formal unentscheidbare Sätze der Principia Mathematica und verwandter Systeme I" [13]で、形式体系$T$が初等算術[^3]を含む限り、その中には証明も否定もできない命題が必ず存在することを証明しました。公理を追加することは何の役にも立たない、なぜなら公理を追加した後も、不完全性定理の前件は依然として満たされ、後件を導き出すことができるからです。[^4]

これは、私たちが上で想像した美しい景色に死刑を宣告したものです。もちろん、私たちはプログラマであり、数学者ではないので、これが私たちに何の関係があるのでしょうか？

## 判定可能性

Gödelが1931年に論文を発表した後、ChurchとTuringは1936-1937年に、一階論理のもう一つの大きな問題--判定可能性の否定証明をそれぞれ提出しました。

前述の通り、一階論理の命題は構文木（または文字列）と見なすことができ、一階理論は記号表と公理集合であり、ある条件下（例えば、以下で説明する再帰的公理化）では、これらはすべてプログラムのデータと見なすことができます。

明らかに、私たちは次のような質問をします：命題$\psi$が$T$内で証明可能かどうかを判断するプログラム$\texttt{p}$は存在するのでしょうか？

この問いの答えは否定的です：**そのようなプログラムは存在しません**。証明方法は後述します。この事実は少し直感に反するかもしれませんが、公理系の証明は理論的には**列挙可能**です。

どのように列挙するのでしょうか？Peano算術とHilbert系を考えてみましょう。Peano算術の公理の中で、帰納公理は無限に存在しますが、それには固定したパターンがあります：

$$
\begin{aligned}
\forall \vec{y_k}. (& (\varphi(0, \vec{y_k})\ \land \\\\
    & (\varphi(x, \vec{y_k}) \rightarrow \varphi(S(x), \vec{y_k}))) \\\\ 
    & \rightarrow \forall x. \varphi(x, \vec{y_k}))
\end{aligned}
$$

ここで列挙する必要があるのは$k$と$\varphi$です。$k$は言うまでもなく、$\varphi$はすべての述語で、その構文は確定しており、固定のルールで生成することができます。しかし、直接`gen`関数を書くのはかなり複雑です。ここではGödelの方法を参考にします。

まず、Peano算術の記号集は無限ですが、私たちはそれを二つの領域に分けることができます：

- 十個の論理記号（簡略化後）
- 無限の変数記号（$v_0$、$v_1$、$v_2$、...と表記）

これら二つの領域の記号を順番にエンコードすると、エンコードとデコードの関数を提供できます。エンコード関数は記号を自然数にマッピングし、デコード関数は自然数を記号にマッピングします。

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

同様に、命題（つまり文字列、文字のリスト）もエンコードとデコードが可能です。Gödelが使用した技術は次のようなものです：

$$
\sharp [ a_1, a_2, a_3, ..., a_n ] = p_1^{\sharp a_1} \times p_2^{\sharp a_2} \times \cdots \times p_n^{\sharp a_n}
$$

ここで、$\sharp a_i$は先ほど定義した`encode(aᵢ)`を表し、$p_i$は$i$番目の素数を表します。

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

`decode_str`の値域はすべての文字列を含みます。これにより、文字列の反復は自然数の反復で実現できます。

文字列が必ずしも有効な命題であるとは限らないため、文字列が有効な命題であるかどうかを判断する関数が必要です。この関数があれば、すべての命題を列挙することができます：

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

関数 `is_valid_stmt`はプログラミング言語の[パーサ](https://en.wikipedia.org/wiki/Parsing)に似ており、書くのは難しいかもしれませんが、確実に書くことができます。

同様に、すべての証明（つまり命題のリスト）を列挙し、命題が証明可能かどうかを判断する関数を構築することができます：

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

プログラム `can_prove`に対する観察：

- リストの最後のものが証明すべき$\psi$である場合、明らかに$T \vdash \psi$は成り立ちます。
- 我々の列挙はすべての証明をカバーしているので、もし$T \vdash \psi$であるなら、つまり要求される証明が存在するなら、その証明は必ず見つかります。
- もし$T \not\vdash \psi$であるなら、このアルゴリズムは全く停止せず、永遠にループします。

これは停止問題の状況と非常に似ており、計算可能性理論では「半決定性」（semi-decidable）と呼ばれています。半決定性は我々が以下の証明で使用する核心概念であり、詳しく議論する価値があります。

> 集合[^5] $S$ は半決定可能である、つまり、すべての $e \in S$ に対して $\texttt{p}(e) = \texttt{True}$ となるプログラム $\texttt{p}$ が存在する場合と同義です。

半決定可能性のキーは、$e \not\in S$ のときにプログラムが停止するかどうかは議論されていないということです。もし $\texttt{p}(e) = \texttt{True}$、つまりプログラムが $\texttt{True}$ を出力するなら、それは $e \in S$ であるということです。しかし、$e_1 \not\in S$ という入力があった場合、$\texttt{p}(e)$ が停止するかどうかは定義には含まれていません。したがって、決定可能な集合は必ず半決定可能な集合であり、半決定可能な集合が必ずしも決定可能な集合であるとは限りません。

> ハルティング問題は半決定可能であり、決定可能ではない

証明は非常に簡単で、半決定可能性が存在するそのプログラムは次のように構築されます：

```python
def p(prog, x):
    exec(prog, x)
    return True
```

もし `prog(x)` が停止するなら、インタープリタ `exec(prog, x)` も停止します。

半決定可能性にはいくつかの等価な定義があります：

1. 再帰的に列挙可能（recursively enumerable, r.e.）。集合 $S$ が再帰的に列挙可能であるとは、$S = \emptyset$ または再帰的な全関数 $f$ が存在し、$S = \\{ y\ |\ \exists x. f(x) = y \\}$、つまり、$S$ は $f$ の値域であることを意味します。
2. $S$ の部分[特性関数](https://zh.wikipedia.org/zh-hans/%E6%8C%87%E7%A4%BA%E5%87%BD%E6%95%B0)は部分的に再帰的です。
3. $S$ はある部分再帰関数の値域です。

「部分関数」（partial function）とは何でしょうか？簡単に言うと、一部の入力で定義されていない関数のことを指します。一部の入力で停止しないプログラムの「像」、つまり $\\{ (x, y)\ |\ x \in A, \texttt{p}(x) = y \\}$ は部分関数です。

これらの等価性は証明が必要で、[1, 5]を参照してください。

`is_valid_stmt` と `is_valid_proof` の構築は、文字列が公理集合に含まれているかどうかを判断するアルゴリズムが存在することを要求します。これは公理集合に要求を課します。実際、ゲーデルの不完全性定理の条件には、公理集合が再帰集合である必要があり、再帰集合と決定可能性は等価で、これは半決定可能性の要求に対してすべての入力で停止するという制限を追加します。

## Gödel の不完全性定理の証明

すでに、少なくとも Peano 算術 $\mathsf{PA}$ については、$\mathsf{PA} \vdash \psi$ の問題が半決定的であることを知っています。この結果と少しのプログラミングの技巧を用いると、すぐに Gödel の不完全性定理を得ることができます。

まず、$\mathsf{PA}$ が完全であると仮定します。つまり、任意の $\psi$ について、$\mathsf{PA} \vdash \psi$ または $\mathsf{PA} \vdash \neg \psi$ です。$\mathsf{PA}$ の半決定器を $\texttt{p}$ とし、$\mathsf{PA}$ の決定器 $\texttt{p\\_total}$ を構築できます。構築は以下の通りです：

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

思いもよらなかったでしょう、このプログラムのキーは実は並行性にあります！並行性のもう一つの名前は「非決定性計算」で、実際、私はここでこの方法を用いて非決定性計算を模倣したいだけです。（つまり、`can_prove_input(ψ)` と `can_prove_input(¬ψ)` を「同時に」計算します）

$\texttt{p\\_total}$ を詳しく分析してみましょう。完全性を考慮に入れて、場合分けを行います：

- もし $\mathsf{PA} \vdash \psi$ ならば、`can_prove_input(ψ)` は停止し、`result.get()` は `ψ` を返すので、全体の関数は `True` を返します。注意してください、この時 `can_prove_input(¬ψ)` は停止しないので、`result.get()` は必ず `ψ` を返します。
- もし $\mathsf{PA} \vdash \neg\psi$ ならば、`can_prove_input(¬ψ)` は停止し、`result.get()` は `¬ψ` を返し、全体の関数は `False` を返します。$\mathsf{PA}$ の一貫性に基づいて、この時 $\mathsf{PA} \not\vdash \psi$ です。

したがって、$\texttt{p\\_total}$ は $\mathsf{PA} \vdash \psi$ の決定器です。これにより、$\mathsf{PA}$ は決定可能となり、これは不決定性の結果と矛盾します。したがって、仮定は成り立たず、$\mathsf{PA}$ は完全ではありません。

読者は納得しないかもしれません：これはズルではないか？ Gödel は何のマルチスレッドを知っているのか？実際、非決定性のチューリングマシンと決定性のチューリングマシンは等価であり、マルチスレッドもシングルスレッドでシミュレートすることができます。しかし、私たちはまだ「正常な」$\texttt{p\\_total}$ を提供することができます。

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

ここでは、元の `can_prove(ψ)` を `can_prove_n(ψ, n)` に変更しました。つまり、$n$ ステップ以内に $\psi$ を証明できるかどうかです。これにより、`can_prove_n(ψ, n)` は必ず停止するため、`p_total` の中で直接 `can_prove_n` を順次計算することができます。

## 決定不能性の証明

上記の議論から、次のことがわかります：

- 半決定性は比較的複雑なプログラムを書く必要がありますが、全体的には直感的です
- 半決定性と決定不能性があれば、直接に Gödel の不完全性定理を得ることができます

問題のキーは、$\mathsf{PA}$ の決定不能性をどのように得るかです。この記事では、証明を厳密に提供するのではなく、証明のアプローチを簡単に紹介します。

### Church の結果を利用する

決定不能性を証明するためには、比較的複雑な概念である表現可能性（representability）を導入する必要があります。Gödel 自身はこの概念を直接使用していないようです [4]、代わりにその等価形を使用しました。

Church は1936年の論文 [11] で初めて決定不能性の問題が存在することを証明しました（一部の学者は彼の証明に満足していないようです）。その問題は次の通りです：

> 与えられた $a, b \in \Lambda^{\circ}$ に対して、$a \approx_{\beta} b$ が成り立つか？

ここで、$\Lambda^{\circ}$ は「閉じた lambda 項」を指し、$\approx_{\beta}$ は β-等価関係を指します。つまり、$a$ と $b$ の間には $\beta$ 簡約と $\beta$ 抽象からなるパスが存在します。彼の論文はさらに、任意の ω-一貫性のシステムは判定器を構築できないと指摘しています。[11, 12]

Church の証明を引き続き使用する場合、まず $\approx_{\beta}$ を算術化する必要があります。つまり、整数上の関係 $P$ を見つけ出し、$P(\sharp M, \sharp N)$ が成り立つ場合と唯一に $M \approx_{\beta} N$ が成り立つ場合を見つけ出す必要があります。ここで $\sharp$ はエンコーディングを表します。明らかに、$P$ は良好な表現を持つ必要があります。なぜなら、次に $\mathsf{PA}$ の文 $\mathsf{P(x, y)}$ を見つけ出し、任意の $n_1, n_2$ に対して

$$
\begin{aligned}
P(n_1, n_2) &\rightarrow \mathsf{PA \vdash P(n_1, n_2)} \\\\
\neg P(n_1, n_2)& \rightarrow \mathsf{PA \vdash \neg P(n_1, n_2)}
\end{aligned}
$$

を満たす必要があるからです。

$\mathsf{P(n_1, n_2)}$ の中の $\mathsf{n_1}$ は、メタ言語の自然数 $n_1$ が $\mathsf{PA}$ の中で対応する記号列を指しており、つまり

$$
\underbrace{S S S \cdots S}_{n_1 の数}\ 0
$$

$\mathsf{P(x, y)}$ は**確定的な**、2つの自由変数を持つ $\mathsf{PA}$ 文です。これにより：

- $\mathsf{PA}$ が判定可能であると仮定すると、任意の $a, b \in \Lambda^{\circ}$ に対して、
- 判定器 $\texttt{p}$ に $\varphi = \mathsf{P(\sharp a, \sharp b)}$、$\psi = \mathsf{\neg P(\sharp a, \sharp b)} $ を入力します。
- $\texttt{p}(\varphi)$ を返します。

明らかに、このプログラムは Church が判定不能であると証明した問題を判定しています。実際、$P$ が $T$ で表現可能であれば、$P$ の判定問題は $T$ の判定可能性に還元できます。

しかし、$\mathsf{P}$ を見つけるのは、私個人的には面倒な作業だと思います。

### Kleene の結果を利用する

Gödel は1931年の論文で、原始的な再帰関数はすべて表現可能であることを証明しました。関数の表現可能性については、教科書 [1] と資料 [9] の説明が少し異なり、資料 [9] は関数の表現可能性と関係の表現可能性を同一視しています。我々は一時的に後者の説明[^7] を採用し、関数 $f$ が表現可能であるとは、次のような $\mathsf{P}$ が存在するとき、かつそのときに限ります。

$$
f(\vec{m}) = n \rightarrow T \vdash \mathsf{P(\vec{m}, n)} \\\\
f(\vec{m}) \neq n \rightarrow T \vdash \mathsf{\neg P(\vec{m}, n)}
$$

Kleene は正規形定理を証明しました：

> 三項の原始的な再帰関数 $C(x, y, z)$ と一項の原始的な再帰関数 $U(x)$ が存在し、任意の部分的な再帰関数 $f$ が $e$ の値が与えられたときに次のように表現できる：
> $$
> f(n) = U(\mu z[C(e, n, z)])
> $$

この定理は一見難解に見えますが、実際にはインタープリタのようなシステムを説明しています：

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

ここでの $e$ は、プログラム（再帰関数）がエンコードされた結果として理解できます。この定理を通じて、Kleeneは対角線法を用いて以下の定理を証明しました：

> すべての部分再帰関数が潜在的に再帰的（potentially recursive）であるわけではない

「潜在的に再帰的」とは、部分再帰関数 $g$ に対して、全関数 $f$ が存在し、$g$ が定義されている場所では $f(n) = g(n)$ となることを意味します。直感的には、この定理は「すべての半決定可能な集合が決定可能であるわけではない」と似ています。

今、$T$ の判定器 $\texttt{p\\_t}$ が存在すると仮定します。Gödelの証明によれば、公式 $\mathsf{C(e, n, z, t)}$ が $C(e, n, z)$ を表現しているとします。そこで、新たなスーパーインタプリタを構築することができます。これは、すべての部分再帰関数 $g$ に対応する全関数 $f$ を見つけることができます。任意の $g$ を考え、その $e$ 値を $e_1$ とします。$f$ を以下のように構築します：

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

Kleeneの定理によれば、$g$ が停止しない唯一の方法は $\forall z. C(e_1, n, z) \neq 0$ です。したがって、$C$ 自体が $T$ で表現可能な原始再帰関数であるため、$T$ が決定可能であれば、任意の $n$ に対して $g(n)$ が停止するかどうかは決定可能です。

## Gödelの原始証明との比較

Gödelの原始証明 [1, 13] はかなり複雑で、大まかには以下のようにまとめることができます：

1. すべての原始再帰関数が表現可能であることを証明する。
2. 一階論理と初等算術の文法を原始再帰関数で算術化し、ここでは `can_prove_n(ψ, n)` を書くことで解決します。Gödelは初等再帰関数を定義し、対応する初等再帰関係を定義しました。彼は45の再帰関係を書き、最終的に証明可能性（有名なBew）を定義しました。
3. Bewは原始再帰関係であるため、表現可能です。したがって、公式 $\mathsf{P(\sharp\psi)}$ がそれを表現することができます。
4. Gödelは不動点定理を証明しました：任意の一項公式 $\psi(x)$ に対して、文 $\sigma$ が存在し、$T \vdash \sigma \leftrightarrow \psi(\mathsf{\sharp\sigma})$ となります。
5. したがって、$T \vdash \sigma \leftrightarrow \neg \mathsf{P(\sharp \sigma)}$ となる $\sigma$ が存在します。注意してください、この文は嘘つきパラドックスに似た命題を構築します。
6. ω-完全性を利用して、$T \vdash \sigma$ を仮定するか $T \vdash \neg \sigma$ を仮定するかにかかわらず、矛盾が生じます。これにより、$\sigma$ と $T$ は独立していることがわかります。

それは明らかで、私たちの証明は1.と2.を避けていません、これらは最も複雑なステップです。しかし、半決定性は理解しやすく、それを先に持ってくると、証明は非常に短くなります。1.と2.は実際には半決定性を証明するために導入されたものです。

## さらなる読み物

- もしGödelの元の証明を学びたいなら、直接 [1] またはGödelの元の論文 [13] を読んでください。杨睿之先生の本は非常によく書かれています。
- このような証明のアプローチは最初にKleeneから来たようで、彼の著作 [10] とこのノート [9] を参照してください。
- この記事 [4] もこのアプローチを用いて不完全性定理を紹介しています。
- ocau先生のAgda形式化は参考になりますが、彼の講義のビデオは公開されていません（理由は不明）。彼のアプローチはこのCoq形式化の作業 [6] から来た可能性があります、この論文を直接読むことで代替できます。
- Gödelの不完全性定理の完全なCoq形式化 [3]（上記の論文は総合的なもので、ここで議論されているものとは大きな違いがあります）
- Gödelの不完全性定理についての2冊の本 [2, 8] はどちらもよく書かれており、一部の章を選んで読むことができます。
- 再帰関数に関連する知識については、 [1, 10] と宋先生の [7] を参照してください。同様に、計算可能性に関連する概念については [5] を参照してください。

## 参考文献

- [1] [数理論理学：証明とその限界](https://book.douban.com/subject/35210053/)
- [2] [Gödelの定理への導入](https://www.karlin.mff.cuni.cz/~krajicek/smith.pdf)
- [3] [算術の本質的な不完全性がCoqによって検証された](http://r6.ca/Goedel/goedel1.html)
- [4] [Gödelの不完全性現象—計算的に](https://www.cairn.info/revue-philosophia-scientiae-2014-3-page-23.htm)
- [5] [計算可能性と複雑性：プログラミングの視点から](http://hjemmesider.diku.dk/~neil/comp2book2007/book-whole.pdf)
- [6] [涙なしのGödelの定理：合成計算可能性における本質的な不完全性](https://drops.dagstuhl.de/opus/volltexte/2023/17491/pdf/LIPIcs-CSL-2023-30.pdf)
- [7] [計算モデルへの導入](https://book.douban.com/subject/10785250/)
- [8] [不完全性と計算可能性：Gödelの定理へのオープンな導入](https://ic.openlogicproject.org/ic-screen.pdf)
- [9] [Gödelの定理のKleeneの証明](https://www.logicmatters.net/resources/pdfs/KleeneProof.pdf)
- [10] [メタ数学への導入](https://archive.org/details/BubliothecaMathematicaStephenColeKleeneIntroductionToMetamathematicsWoltersNoordhoffPublishing1971)
- [11] [初等数論の解けない問題](https://www.ics.uci.edu/~lopes/teaching/inf212W12/readings/church.pdf)
- [12] [A. Church on Computability](https://plato.stanford.edu/entries/church/supplementA.html)
- [13] [Principia Mathematicaと関連システムIの公式による決定不能命題について](https://monoskop.org/images/9/93/Kurt_G%C3%B6del_On_Formally_Undecidable_Propositions_of_Principia_Mathematica_and_Related_Systems_1992.pdf)

[^1]: ここでの命題は「メタ言語」の命題であり、私たちが議論している言語、すなわち「対象言語」、ここでは一階論理の命題とは異なります。混同しないでください。
[^2]: もちろん、Hilbert Systemでは、「論理公理」、つまり任意の一階理論が成り立つ公理も存在します。これらの公理も推論の出発点として使用でき、任意の論理が成り立つ「恒真式」（tautology）を導き出すことができます。
[^3]: 元の説明はかなり複雑で、ここでの「初等算術」はPeanoよりも弱いもので、これはessential incompletenessの概念に関連しています。
[^4]: 言い換えれば、新しい命題を見つけることができ、この新しい命題は公理を追加した後のシステムと独立しています。
[^5]: ここで言及されている「集合」は、一般的にはプログラムデータ集合の一部で、再帰論では自然数の部分集合と考えられています。
[^7]: [1]の主張は正しいようですが、それはより強い命題を提供しています。ここでは、比較的弱いこの形式だけで十分です。
