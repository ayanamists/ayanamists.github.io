---
title: "Java Stream: “函数式编程”的代价"
date: 2023-08-30T15:31:25+08:00
draft: true
tags:
---

## 什么是 Java Stream

2014 年，Java 8 发布。随之而来的是一些 Java 语言的革命性进步，包括匿名函数、stream 等等。

有了 stream，Java 程序员终于可以流畅地用 `map`, `filter`, `reduce` 等等函数式程序的基本函数构造程序。例如，如果想要把一个列表变为一个 `String` 的列表，可以写成如下形式：

```Java
List<A> l = ...
List<String> res = l.stream().map(A::toString).toList()
```

然而，Java Stream 对性能造成的影响一直是一个很有争议的话题。
