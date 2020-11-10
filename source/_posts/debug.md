---
title: 欢饮来到真实世界--原生程序的错误与调试
date: 2020-11-10 14:32:34
tags: 
---

## 原生程序的错误

如果学习编程的人像我以及大部分中国大陆的计算机系学生一样，从C语言开始学习编程的话，那么，他从一开始就处于了【真实世界】。

但是，在我的观念中，从C语言学习编程是很不正确的决定。

不过从C语言开始学习编程也不是没有一点好处。如果某人从scheme等等有GC的语言开始学习编程的话的话，那么他到了某些不得不用C/C++写程序的时候，会陷入一些很麻烦的问题之中，其中之一，就是C/C++都可能会出现【裸错误】，或者说，在硬件层面，操作系统层面的错误，会毫无保留地送到程序员面前。

比如说，考虑这样的C代码：

```c
#include <stdio.h>

int main()
{
	int a[10];
	a[10000] = 0;
}
```

编译运行之后，如果不出意外的话，shell会告诉你：

```shell
➜  ~ ./a            
[1]    398 segmentation fault  ./a
```

这是什么意思？考虑一下scheme里类似的操作：

```scheme
(vector-ref (vector 1 2) 10000)
```

scheme会告诉你：

```scheme
> (vector-ref (vector 1 2) 10000)
Exception in vector-ref: 10000 is not a valid index for #(1 2)
Type (debug) to enter the debugger.
```

很显然，scheme在抱怨程序访问了`(vector 1 2)`这个只有两个元素的向量的第`10000`个元素，**以一个异常（Exception）的形式**。

在scheme、java、C#等等几乎所有比较高级的（非纯函数式）语言中，任何的错误（这里指类似于访问两个元素是数组的`10000`个元素的错误）都会产生一个**异常（Exception）**，不会产生`segmentation fault`.

然而，C语言没有这种能力。这来源于C语言的本质属性--原生。

什么是原生呢？原生，即是说，C语言里的数据类型都是没有怎么封装的、和硬件紧密结合的。

比如说一个`Int`，在大多数情况下，它就是`32`位的整数，在内存里占`4`个字节。

再比如说一个数组，（一般来说）它就是一段连续的内存，访问数组就是寻址。

特别地，C语言里的【指针】，指针的值就是内存地址。

比如说，

```c
	int a = 10;
	int *b = &a;
	printf("%llx", b);
```

运行编译好的程序会在不同时候打出不同的值，但它们的本质都是一样的 --  `a`这个变量的内存地址。

![0](https://pic.downk.cc/item/5faa3b8d1cd1bbb86baf8a5c.jpg)

到了这里，我们终于能够回答`segmentation fault`究竟是个什么东西了。实际上，这是所谓的【内存访问错误】，比如说这样的代码：

```c
int *a = NULL;
*a = 10;
```

会出现`segmentation fault`，是因为现代计算机大都不允许使用`0`这一内存地址，访问它直接会引起操作系统层面的错误。

至于它为什么叫这个名字，这和历史有关，不在此赘述。

## 原生程序的调试

关于如何调试原生程序，我想，首先我们要明确【为什么要进行调试】。

进行调试，首先就是要**确定错误位置**。比如这样的代码：

```c
#include <stdio.h>

void a()
{
}

void b()
{
}

void c()
{
	int *a = NULL;
	*a = 0;
}

int main()
{
	a();
	b();
	c();
}
```

现在程序出了错误，你如何判断这个错误是在`a()`，还是在`b()`，抑或是在`c()`里触发的呢？

有人会说，这不简单？看一眼就知道了。呵呵，这确实没有那么简单，如果`a(),b(),c()`每一个都有`1000`行呢？

这个时候，我们必须借助调试器的力量。

>tips: 在编译程序时, 加入 -g 选项

首先，用`gdb <要调试的程序>`进入调试环境：

![1](https://pic.downk.cc/item/5faa3d741cd1bbb86bb00378.jpg)

然后使用`run`或者`r`，运行程序：

![2](https://pic.downk.cc/item/5faa3da31cd1bbb86bb00e45.jpg)

可见，它也出现了错误，并停了下来。

其实，这里我们已经可以知道错在哪里了。因为输出已经明确告诉了我们，错误在`c()`这个函数里，`a.c`文件的第`14`行。不过，有些时候这些信息还是不够的，必须要知道**谁调用了c()这个函数**。

使用`bt`或者`backtrace`，我们就可以知道这个信息：

![3](https://pic.downk.cc/item/5faa3e6e1cd1bbb86bb03e86.jpg)

非常明显，这个函数是`main()`调用的。

这些信息是非常宝贵的**定位信息**，我们有时还想知道一些**运行时信息**，比如在这个例子里，我很想知道，为什么`*a`会引起错误。

要知道这个问题，就不得不问问`gdb`，`a`这个变量里究竟装了什么，而这是通过`print`或者`p`命令实现的：

![4](https://pic.downk.cc/item/5faa3f501cd1bbb86bb07555.jpg)

哈哈，原来是因为`a`这个变量里装的是`0x0`，所以才会引起错误。

用这几招，我们可以解决大部分的`segmentation fault`问题。不过有些时候，我们还需要更进一步，比如下面的程序：

```c
#include <stdio.h>

int div(int a, int b)
{
	int c = a / b;
	return c;
}

int main()
{
	int a = 1000 - 1000;
	int b = 10;
	printf("%d", div(b, a));
}
```

这会造成：

```c
➜  ~ ./a            
[1]    567 floating point exception  ./a
```

用gdb调试，会出现:

![5](https://pic.downk.cc/item/5faa41191cd1bbb86bb0d451.jpg)

我们可以知道是因为`b=0`所以出了问题, 但我们更想知道, **这问题到底是怎么发生的**， 或者说， **为什么**`b=0`了呢?

这时, 可以到调用它的函数里去看看:

![6](https://pic.downk.cc/item/5faa41d91cd1bbb86bb10021.jpg)

`f`表示`frame`，这个命令是说，到`1`号函数帧，也就是调用`div(b, a)`的函数现场去看看，这里，我们可以知道，因为`main`函数中的`a`变量是`0`，所以调用`div(b,a)`时，`div`函数的第二个参数为`0`.

有些时候，程序没有出现`Segment Fault`，但是出现了一些别的错误，那就可以用`break`、`continue`、`step`这三个组合来进行所谓的**断点调试**了。不过断点调试和java、C#中的调试没有太大区别，如果想要继续了解，可以看看这两个页面：

+ [简单的教程](https://condor.depaul.edu/glancast/373class/docs/gdb.html)
+ [官方教程](https://ftp.gnu.org/old-gnu/Manuals/gdb/html_node/gdb_toc.html)