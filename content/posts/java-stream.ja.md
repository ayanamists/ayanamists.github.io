---
title: "Java Stream：「関数型プログラミング」のコスト"
date: 2023-08-30T15:31:25+08:00
draft: true
tags:
---

{{<  admonition "warning" >}}
この記事はChatGPTによって中国語から翻訳されたもので、いくつかの誤りが含まれているかもしれません。不正確な部分があればご了承ください。
{{< /admonition >}}

## Java Streamとは何か

2014年、Java 8がリリースされました。それに伴い、Java言語の革命的な進歩がいくつかありました、それには無名関数、streamなどが含まれます。

streamがあるおかげで、Javaプログラマーはついに `map`、`filter`、`reduce` などの関数型プログラミングの基本関数を使ってプログラムをスムーズに構築することができます。例えば、リストを `String` のリストに変換したい場合、以下のように書くことができます：

```Java
List<A> l = ...
List<String> res = l.stream().map(A::toString).toList()
```

しかし、Java Streamがパフォーマンスに与える影響は常に議論の的でした。
