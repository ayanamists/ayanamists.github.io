---
title: "君の知らない物語 -- Churchと彼のシステム"
date: 2024-03-15T22:29:00+08:00
draft: false
categories:
  - コンピューターの歴史
---

{{<  admonition "warning" >}}
この記事はChatGPTによって中国語から翻訳されたもので、いくつかの誤りが含まれているかもしれません。不正確な部分があればご了承ください。
{{< /admonition >}}

関数型プログラミングの愛好者であれば、Churchと彼の$\lambda$計算については知っているでしょう。再帰理論を学んでいれば、$\lambda$定義可能性、$\lambda$計算の各種性質（Church-Rosser定理、不動点定理など）、そしてそれらがどのように組み合わさって$\beta$等価性の不可決定性証明を与えるのかも知っているかもしれません。

もちろん、これらの言葉が何を意味しているのか全く知らないかもしれませんが、それでも大丈夫です。もし少しでもコンピューターサイエンスを学んでいれば、チューリングマシン（Turing Machine）については知っているはずです。チューリングと彼の論文をコンピューターサイエンスの起源と見なす人もいますが、実際には、Churchと彼の$\lambda$計算はチューリングマシンよりも早く、また優れていました[^2]。今日私たちが話す話は、実際には理論コンピューターサイエンスの起源の一つです。

ほとんどの人は知らないのですが、Churchが当初$\lambda$計算を発明した理由、そして彼が当初発明したシステムが現在私たちが知っている$\lambda$計算と何が違うのか。実際には、Churchのシステムは現在の$\lambda$計算よりもはるかに複雑です。なぜChurchのシステムは忘れ去られ、現在の$\lambda$計算がコンピューターサイエンスの基礎となったのでしょうか？

これは非常に興味深い話です。私は2023年のSICPクラス[^3]でこの話を学生たちと共有しましたが、今、私はこれを書き留め、この歴史をより多くの人々に知ってもらうべきだと思います。

{{<  admonition "note" >}}
この記事にはいくつかの技術的な詳細が含まれていますが、重要なのは物語です。理解できない部分はスキップしても、物語の理解に影響はありません。ただし、少なくとも最初のセクションの内容を理解しておくことをお勧めします。そうすれば、Churchが何をしていたのかを大まかに理解することができます。
{{< /admonition >}}


## 形式システム

Churchが1932年に発表したシステム[^1]は、現在では「形式システム」（formal system）と呼ばれています。簡単に言えば、形式システムには以下の三つの要素があります：

- 構文（syntax）- 合法的な表現（formula）を記述するための一連の記号と規則。ここでの「構文」はプログラミング言語の「構文」と同じで、システム内の表現がどのような形をしているかを規定しています。
- 公理群（axioms）- 真であるとされる一連の表現。これらの公理はシステムの基礎であり、他のすべての定理はこれらの公理から導き出されます。
- 推論規則（inference rules）- 既知の公理と定理から新たな定理を導き出すための一連の規則。

私は非常に単純な形式システム（$P$と表記）を提供することができます：

- 文法 - $a$ と $b$ から成る文字列。例えば $a$、$aa$、$ab$、$ba$ など
- 公理 - $a$、$b$
- 推論規則
  - (I.) もし $x$ が定理であれば、$axa$ と $bxb$ も定理である。

このシステムのすべての定理はいわゆる "回文"（palindrome）です。例えば、$aba$ は定理であり、その導出過程（これを "証明"、proofと呼びます）は次のようになります：

- $b$   (公理)
- $aba$ (I.)

論理学では、"$aba$ はシステム $P$ の定理である" は $P \vdash aba$ と書くことができます。ここでの $\vdash$ は "導出" を意味します。

逆に、$ab$ は定理ではありません。なぜなら、それを導出する規則が存在しないからです。これは $P \nvdash ab$ と書くことができます。

ここで言及している "公理"、"定理"、"証明" はすべて形式システムの用語です。それらは厳密な数学的意味を持ち、例えばこの場合、"公理" は $a$ と $b$ を指し、"定理" は回文を指し、"証明" は導出規則を満たす文字列の列を指します。しかし、明らかに、私たちは幼少期から数学教育を受けており、"公理"、"定理"、"証明" はすでに自然言語の意味を持っています。私が少し時間をかけて、それらの関係を説明させてください。

自然言語の "公理" は、以下では _公理_ と表記します。一方、形式システムの "公理" は `公理` と表記します。他の概念も同様に扱います。

私たちは幼少期から様々な _定理_ を学んできました。例えば $1 + 1 = 2$ などです。これらの _定理_ のほとんどは、現代の論理が確立する前に _証明_ されました。したがって、論理学が存在する前から、_証明_ や _定理_ といった概念が存在していました。これらの自然言語で記述され構築された数学は、ほとんどの場合、それが "存在し" 正しいと考えられています。

一方、形式システムの `定理` は、厳密な数学的オブジェクトであることをすでに述べました。その性質を知り、それがどのように公理から導出されるかを知っています。この導出過程は、コンピュータで検証することができます。要するに、これらの数学的オブジェクトは明確で厳密です。しかし、自然言語の数学は、少なくともそのような厳密さはありません。例えば、数学の教科書をランダムに選んで、その中のすべての証明を簡単に検証できるプログラムが存在すると想像できますか？

だから私の見解では、形式システムの主な目的は、厳密な数学的オブジェクトを用いて自然言語の数学を "固定化" することです。"固定化" とは、より学術的な言葉で表現すると "形式化" ということです。

自然言語で記述された数学と形式システムで記述された数学は、アルゴリズムとプログラムのようなものです。アルゴリズムは非形式的なプログラムであり、プログラムは形式化されたアルゴリズムです。

自然言語の数学的証明を形式システムで形式化することは、非常に難しい問題です。この問題を解決するためには、より複雑で面倒な形式システムが必要です。では、どのような形式システムを選ぶべきでしょうか？

この難問に対して、多くの数学者は19世紀後半に発展した別の学問分野である集合論が答えだと考えています。ほとんどの数学的なオブジェクトは集合論で記述することができ、集合論は形式システムで記述することができます。Russellが1910-1913年に発表した _Principia Mathematica_ や Zermelo-Fraenkel 集合論（1908, 1922）は、このアプローチの実践例です。ZF集合論は基本的に現代数学の基礎となっています。

## Churchのシステム

Churchは1920年代に、集合論ではない方法で形式システムを提供しようと試みました。Churchは1932年の論文で、Russellの_Principia Mathematica_やZF集合論が、有名な"Russell's Paradox"を処理するために"不自然な"方法を採用していると述べています。

彼が提案した方法は、「排中律の使用を制限する」というものです。技術的には、私は彼のこの方法がむしろ「不自然」であると感じており、この記事の範囲内で明確な説明を提供するのは難しいです。とにかく、彼は37の（そう、あなたが読んだ通り、37の）公理と5つの推論規則を持つ形式システムを設計しました。これら5つの推論規則は、現代の$\lambda$計算につながっています。このシステムは、1932年に発表された_A Set of Postulates for the Foundation of Logic_で提供されました。

Churchのシステムでは、文法に集合やクラスは存在せず、以下の記号で構成されています：

> 5. 定義されていない用語。私たちは今、私たちの形式論理の未定義の用語のリストを作成する準備ができました。それらは次のとおりです：
$$
\\{\\}(),\\ \lambda\[\],\\ \Pi,\ \Sigma,\ \\&, \\ \sim, \\ \iota, \\ A
$$

ここで、$\\{\\}()$ は現代の $\lambda$ 計算での「適用」（apply、いわゆる関数呼び出し）を表し、$\lambda [\ ]$は自然に$\lambda$抽象を表します。$\Pi$は比較的複雑な文法で、$\Pi(P, Q)$は大体$\forall x. (P(x) \to Q(x))$[^5]を表します。$\Sigma$は存在を表し、$\\&$は論理積、$\sim$は論理否定を表します。$\iota$ は再帰関数の中の $\mu$ 演算子に似ており、$\iota(F)$は「 $\\{F\\}(x)$ を満たす $x$」を指します。$A$ については、この記号は特に抽象的で、ここでは議論しません。

その後、Churchはいくつかのシンタックスシュガー、つまり「マクロ」を定義しました。例えば、彼は$V$を定義しました。これは現代の論理学での$\lor$に相当し、定義は以下のようになります：

$$
V(P, Q) \equiv \sim . \sim P . \sim Q
$$

今日の記号で表現すると：

$$
P \lor Q \equiv \neg (\neg P \land \neg Q)
$$

注目すべきは、Churchが当時流行していた、Russellが採用していた「ドット記法」（dot notation）を使用して論理演算を表現していたことです。簡単に言えば、$.$は「優先順位」を決定し、これにより括弧を少なく書くことができます。例えば、$\sim . \sim P . \sim Q$は$\sim (\sim P \land \sim Q)$の省略形です。しかし、私の知る限りでは、Principia Mathematicaのドット記号はこのような使用を許可していないようで、この命題はPMでは$\sim (\sim P . \sim Q)$となるはずです。（もちろん、ここでの誤解もあり得ます。私が参考にしたのは[The Notation in Principia Mathematica](https://plato.stanford.edu/entries/pm-notation/#Exam)です）.

また、Churchは含意記号$\supset$もマクロとして定義しました。

とにかく、Churchのシステムの文法は大体こんな感じです。彼の37の公理は、さらに複雑です。15.のようなものは比較的理解しやすいかもしれません：

$$
\text{15.}\ pq \supset_{pq} q 
$$

これは$p \land q \to q$を指しています。しかし、大部分の公理は非常に理解しにくいです。例えば、第1条：

$$
\text{1.}\ \Sigma(\varphi) \supset_{\varphi} \Pi(\varphi, \varphi)
$$

$\Pi(\varphi, \varphi)$は前提がなくても成り立つはずです（$\forall x. (\varphi(x) \to \varphi(x))$として理解すると）。しかし、Churchの制限により、ここでは$\Sigma(\varphi)$が保証されていなければなりません。

37の難解な公理と比べて、今日の我々にとって、彼の5つの推論規則は非常にシンプルに見えます。これら5つの規則は現代の$\lambda$計算と非常に似ています。これら5つの規則（私の再表現を含むが、Churchの原著とは違いはない）は：

**I.** 真公式 $J$ の中で、

- $x$ のすべての出現が束縛出現である
- $y$ が $J$ の中で自由に出現しない

場合、$J$ の中で $x$ を $y$ で置き換える（$\text{S}_{y}^{x}\ J$ と表記）と、結果の $K$ も真公式となります。

**II.** もし

- $(\lambda x. M) N$ が真公式 $J$ の部分公式である
- $x$ が $M$ の中で束縛出現しない
- $x$ が $M$ の中で自由に出現する (*)
- $x$ が $N$ の中で自由に出現しない

場合、$J$ の中で特定の $(\lambda x. M) N$ を $\text{S}_{N}^{x}\ M$ で置き換えると、結果の $K$ も真公式となります。

**III.** もし

- $\text{S}_{N}^{x}\ M$ が真公式 $J$ の部分公式である
- $x$ が $M$ の中で束縛出現しない
- $x$ が $M$ の中で自由に出現する (*)
- $x$ が $N$ の中で自由に出現しない

場合、$J$ の中で特定の $\text{S}_{N}^{x}\ M$ を $(\lambda x. M) N$ で置き換えると、結果の $K$ も真公式となります。

**IV.** もし $F(A)$ ならば、$\Sigma(F)$。

**V.** もし $\Pi(F, G)$ かつ $F(A)$ ならば、$G(A)$。

これら5つのルール、**IV.** と **V.** は一階論理の「存在量化子の導入」とmodus ponensに似ています。主役は **I.**, **II.**, **III.**:

- **I.** は $\alpha$ 等価性
- **II.** は $\beta$ 等価性の一方向で、$\beta$ 簡約と見なすことができます
- **III.** は $\beta$ 等価性のもう一方向で、$\beta$ 抽象と見なすことができます

これら三つのルールは、現代の $\lambda$ 計算の核心を形成しています。(*) のルールは現代の $\lambda$ 計算には存在しませんが、これは一旦置いておき、興味があれば自分で考えてみてください。

## Kleeneと$\lambda$定義可能性

上記の複雑な記号の中で、何が理解できたでしょうか。もし数年前の私に見せられたら、私は確かに混乱していたでしょう。さらに、Churchのシステムには37の公理があり、このシステムで証明を行うことを考えるだけで頭が痛くなります。ZF集合論と比較して、Churchのシステムは明らかに過度に複雑です。さらに、彼が一階論理を飛ばしてしまった方法は、学習の難易度をさらに高めています。

幸運なことに、ChurchにはKleeneという学生がいました。

Kleeneは1931年、22歳で学士を取得した後、Princeton大学でChurchの博士課程に進学しました。驚くべきことに、彼は1934年に博士号を取得し、その時点で25歳でした。彼はまさに「天才、超天才」[^6]と言えるでしょう！

1931年の秋学期に、ChurchはKleeneと他の学生に彼の新しい発明--つまり、上で述べたシステム--を教えました。Kleeneの回想によれば、Churchはすでに$\beta$簡約のアイデアを持っていました。$\beta$簡約とは何かと言うと、簡単に言えば、$(\lambda x. M) N$を関数呼び出しと見なし、それを$\text{S}_{N}^{x} M$に置き換えることです。つまり、彼のシステム内の一つの公式に対して、公理 **II.** を用いて推論することです。

重要なのは、この推論は一種の「計算」であると見なすことができるということです。$\lambda$項$a$が与えられ、任意の回数$\beta$簡約を行って$b$を得た場合、これは$\lambda$計算の「計算」と見なすことができます。実際、この「計算」は関係$a \rightarrow_{\beta} b$を定義します。これは、$a$が$\beta$簡約によって$b$を得ることができるという意味です。

当時の講義でChurchは、彼のシステムで正の整数を定義する方法をすでに見つけているとも述べていました。これは今日でいうChurch数です。$1, 2, 3$は次のように定義されます：

$$
\mathsf{1} \equiv (\lambda f. \lambda x. f x) \\\\
\mathsf{2} \equiv (\lambda f. \lambda x. f (f x)) \\\\
\mathsf{3} \equiv(\lambda f. \lambda x. f (f (f x)))
$$

上述の$a \to_{\beta} b$の考え方と組み合わせて、Churchは、いくつかの関数が$\lambda$項で表現できることを発見しました。例えば、後続関数$S(n) = n + 1$は、$\lambda$項で次のように表現できます：

$$
\mathsf{S} \equiv \lambda n. \lambda f. \lambda x. f (n f x)
$$

これは性質$\mathsf{S}\ \mathsf{1} \to_{\beta} \mathsf{2}$を持っています。より一般的に、Churchは$\lambda$定義可能性（$\lambda$ definability）の概念を提唱しました。彼の考え方は、関数$f$が$\lambda$項で表現できるなら、$f$は$\lambda$定義可能であるというものです。

形式的に言えば、ある（偏）関数 $f$ に対して、ある $\lambda$ 項 $\mathsf{F}$ が存在し、$f$ の定義域内の自然数 $n$ に対して

$$
\mathsf{F}\ \mathsf{n} \to_{\beta} \mathsf{f(n)}
$$

が成り立つなら、$f$ は $\lambda$ で定義可能であると言えます。上記の $\mathsf{f(n)}$ は、$f(n)$ （自然数）に対応する Church 数を指します。

Church は、後続関数だけでなく、加法関数 $f(x, y) = x + y$ も $\lambda$ で定義可能であることを発見しました。しかし、Church は "前任者" 関数（$P(n) = n - 1$）が $\lambda$ で定義可能かどうかを知りませんでした。もちろん、彼のシステムでは、$\lambda$ の定義性を使わずに "チート" をして前任者関数を得ることができました。彼の定義は次のようなものでした：

$$
\lambda r. \iota x. (\mathsf{S}\ x = r)
$$

この $\iota$ の定義は前のセクションで見たもので、簡単に言えば、Church は " $\mathsf{S}\ x = r$ を満たす $x$ " を用いて $r$ の前任者を定義しました。

1932年初頭、ChurchはKleeneに自然数の公理（Peano Axioms）を彼のシステムで証明する方法を考えるように頼みました。Kleeneは、ここには前任者関数が必要であることを発見し、Churchの "チート" には特に満足していませんでした。そこで、彼は前任者関数が $\lambda$ で定義可能かどうかを研究し始めました。

Kleeneの回想によると（[Kleene 1981]参照）、彼は最初、自然数の定義を少し変えようと考えましたが、Churchは彼のChurch数を使い続けたいと思っていました。そこで、彼はどのようにして前任者関数を提供するかを考え尽くしました。そして、1932年1月末または2月初に、Kleeneは歯科医院で待っている間に、$\lambda$で前任者関数を定義する問題を解決する方法を突然思いつきました。その結果、Kleeneは $\lambda$ 項を提供し、前任者関数を $\lambda$ で定義しました。具体的な定義はここでは示しませんが、興味のある方は自分で考えてみてください。

Kleeneがこの発明をChurchに伝えたとき、Churchはほとんど諦めていたと言いました：

> 私がこの結果をChurchに持って行ったとき、彼は前任者関数の$\lambda$-定義が存在しないとほとんど確信していたと言いました。

元々 $\lambda$ で定義できないと考えられていた関数が、Kleeneによって定義可能であることが証明されました。KleeneとChurchは新たな問題を考え始めました：

> 一体どのような関数が $\lambda$ で定義可能なのか？

この問題に対して、90年後の今日、我々はすぐに答えます：**関数 $f$ が $\lambda$ で定義可能であるとは、$f$ が計算可能であるとき、そしてそのときに限る**。

当時、Kleeneの主な仕事はChurchのシステム内で数学的な命題を証明することでした。副業として $\lambda$ の定義可能性を研究していました。彼は、直感的に計算可能な自然数関数のほとんどすべてが $\lambda$ で定義可能であることを発見しました。1933年にGödelがPrincetonに来て、1934年に彼は再帰関数の定義方法を報告しました。これが現在のHerbrand-Gödel計算可能関数です。Kleeneはこの報告を見て、この方法が彼の $\lambda$ の定義可能性と関連していることに気づきました。

最終的に、KleeneとChurchは1935-36年に発表した論文で、一連の問題を証明しました：

- $\approx_{\beta}$ 、つまり $\beta$ 等価性は、$\lambda$ で定義できない
- $\lambda$ で定義可能な関数とGödelの計算可能関数は等価である
  
その後の話は我々が非常によく知っています。Turingは1936年に彼の論文を発表し、Turing Machineを提案し、Turing MachineとChurchの $\lambda$ 演算が等価であることを証明しました。理論計算機科学はそこから生まれました。

残念なことに、当時の多くの人々は $\lambda$ 演算があまり "数学的" でないと感じていたため、Kleeneの後の偉大な仕事はすべて彼が定義した $\mu$-再帰関数に基づいており、 "計算" モデルについても、人々はTuring Machineを好んでいました。

## Churchのシステムはどのようにして忘れ去られたのか

$\lambda$ の定義可能性の研究は現代の計算機科学の扉を開きましたが、Churchのシステムはどうなったのでしょうか？

Kleeneは回顧録で、いくぶん皮肉っぽく言っています：

> この論文には、Gödelの有名な証明、つまり通常の初等数理論を体現する形式システムにおける決定不能な文の存在証明と、そのようなシステム自体の無矛盾性の証明の不可能性に関する "第二の定理" が含まれています。Churchの直接的な反応は、私がこれから詳しく説明する彼の形式システムが、Gödelが扱ったシステムとは十分に異なるため、Gödelの第二の定理がそれに適用されないかもしれないというものでした（Church 1933 top p. 843を参照）。実際、Churchは正しかった！

Churchは自身のシステムがGödelの制約を受けないと思っていたようです（現在ではそれは不可能であることがわかっています）。そしてKleeneもそれに賛成？これはどういうことなのでしょうか？

ああ、実はKleeneの次の文でその理由が説明されています：

> 彼のシステムでは、その無矛盾性の証明が存在します。なぜなら、実際にはそれは無矛盾性がない（したがって、そのすべての文は証明可能）であり、Churchが可能だと考えていた（1933年上部p.842）こと、そして私たちが後に示した（KleeneとRosser 1935）ことです。

Kleeneさん、先生のシステムをこんなに冗談にするなんて。これは少し「地獄のジョーク」のような感じですね。そうです、Churchのシステムは実際にGödelの制約を超えています。なぜなら、それは根本的に無矛盾性がないからです！

「無矛盾性がない」とは、つまり、このシステム$T$は、ある命題$P$を証明することができ、同時に$\neg P$も証明することができるということです。これは直感的には非常にまずい：このシステムにはある種の「矛盾性」があります。技術的にはさらにまずい、なぜならこれはこのシステムが任意の命題を証明できることを意味するからです。$T \vdash P$と$T \vdash \neg P$があると仮定し、任意の命題$Q$を証明するために次のような推論を行うことができます：

1. $P$
2. $P \lor Q$
3. $\neg P$
4. $Q$

この推論で使用される推論規則は「または」の導入と消去で、ほとんどの形式システムにこの規則があります。したがって、システムが無矛盾性がない場合、それは任意の命題を証明できます。一般的に、このようなシステムは意味がないと考えられます。

KleeneとChurchのもう一人の学生であるRosserは、1935年にChurchのシステムが無矛盾性がないことを証明しました。彼らの証明は「Kleene-Rosserパラドックス」と呼ばれています。このパラドックスは技術的には複雑ですが、直感的には、Churchのシステムがすでに任意の計算可能な関数をエンコードできるようになっていることが、無矛盾性がないことを強く示唆しています。

1984年にRosserが書いた論文では、Curryが発明した技術を使ってChurchのシステムの無矛盾性がないことを証明する方法も紹介されています。この証明は非常にシンプルです。

実際、Churchは1933年に発表した同名（および1932年の同名）の論文で、自身のシステムが無矛盾性がないことを発見しました。この無矛盾性のなさを解決するために、彼は公理19を削除し、2つの公理を追加しました。これにより公理の数は38になりました。彼はこの新しいシステムが一貫していると考えていましたが、現在では、このシステムも無矛盾性がないことがわかっています。

それで、簡単に言えば、Churchのシステムは無矛盾性がなく、非常に複雑であるため、人々に忘れられてしまったのです。

## 結論

Churchの最初の目的は、彼のシステムを使って数学を形式化することでしたが、そのシステムは今ではすでに誰も注目していません。しかし、彼のシステムの副産物である$\lambda$計算は、コンピュータ科学の基礎となりました。これらの偉大な名前、Church、Kleene、Turing、Gödel、...、は永遠にコンピュータ科学の歴史に刻まれています。

彼らの仕事は、全体としては「再帰論」と呼ばれるべきです。特にKleeneの多くの研究は、計算不能集合とその間の関係を研究しています。しかし、私はやはり、「計算理論」という学問が$\lambda$計算や再帰関数を教えず、自動機を教えるのであれば、それは確かに多くの面白いものを欠いていると感じます。もし、あなたの「計算理論」も一群の自動機であるなら、この記事があなたに$\lambda$計算に対する興味を持つきっかけになれば幸いです。

## クイズ

1. Church数は最初にChurchの1933年の論文で発表され、この論文では彼のシステムでのPeanoの公理の表現も示されています。しかし、彼が表現したChurch数もPeanoの公理も、$1$から始まり、$0$から始まらない。なぜChurchは$0$を彼のシステムに入れなかったのでしょうか？これは彼の好みだけでなく、より技術的な問題です。
2. 現代の$\lambda$計算では、我々は一般的に（二）タプルのコンストラクタ（constructor）を次のように定義します。
   $$(a, b) \equiv \lambda a \lambda b \lambda f. f a b$$
   fstとsndは次のように定義されます。
   $$
   \begin{aligned}
   \text{fst} &\equiv \lambda p. p (\lambda a \lambda b. a) \\\\ 
   \text{snd} &\equiv \lambda p. p (\lambda a \lambda b. b)
   \end{aligned}
   $$
   タプルの定義はKleeneが前任関数を定義するための重要なステップですが、Kleeneはこのようにタプルを定義できますか？なぜですか？もしできないなら、Kleeneが使用できるタプルの定義を提供できますか？（ヒント：答えは[Kleene 1981]を参照）
3. もしChurchの元のシステムが無矛盾性がないとしたら、なぜ$\lambda$計算自体はこの問題に影響されないのでしょうか？誰もが$\lambda$計算も「無矛盾性がない」と思うことはありませんか？

## 追加読み物

1. Kleene, S. C. (1981). Origins of recursive function theory. Annals of the History of Computing, 3(1), 52-67.
2. Church, A. (1933). A set of postulates for the foundation of logic. Annals of Mathematics, 34(4), 839-864.
3. Church A. (1936). An unsolvable problem of elementary number theory. American Journal of Mathematics, 58(2), 345-363.
4. Church A. (1941). The Calculi of Lambda-Conversion. Princeton University Press.
5. Church A. (1932). A set of postulates for the foundation of logic. Annals of Mathematics, 33(2), 346-366.
6. Rosser J. B. (1984). Highlights of the history of the lambda calculus. Annals of the History of Computing, 6(4), 337-349.

## 付録：Church の 37 の公理

深層学習技術のおかげで、私は簡単に Church の 37 の公理の latex コードを抽出することができました。

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

[^1]: 私はそれを $\lambda$ システムと呼ぶことをお勧めしません。なぜなら、$\lambda$ システムは現在、多くの場合、$\lambda$ 等価性を構築する形式システムを指すからです。それはChurchの原始システムを指すものではありません。
[^2]: これは私の個人的な見解です。しかし、文法的にも意味的にも、$\lambda$ 計算はチューリングマシンよりもはるかに単純です。
[^3]: 南京大学の「コンピュータプログラムの構築と解釈」のコース。
[^5]: 実際、Churchの論文では、$\Pi(P, Q)$ と $\forall x. P(x) \to Q(x)$ の違いを強調しています。これは彼の「排他的法則に対する制限」に関連しています。しかし、この詳細は本文の範囲を超えています。
[^6]: http://dangshi.people.com.cn/n/2014/0415/c85037-24898922-2.html
