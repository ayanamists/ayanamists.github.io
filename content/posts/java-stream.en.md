---
title: "Java Stream: The Cost of 'Functional Programming'"
date: 2023-08-30T15:31:25+08:00
draft: true
tags:
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

## What is Java Stream

In 2014, Java 8 was released. Along with it came some revolutionary advancements in the Java language, including anonymous functions, stream, and so on.

With stream, Java programmers can finally construct programs fluently with basic functions of functional programming such as `map`, `filter`, `reduce`, etc. For example, if you want to turn a list into a list of `String`, it can be written as follows:

```Java
List<A> l = ...
List<String> res = l.stream().map(A::toString).toList()
```

However, the impact of Java Stream on performance has always been a controversial topic.
