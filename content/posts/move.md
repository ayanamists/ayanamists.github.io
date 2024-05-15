---
title: "本站即将迁移"
date: 2024-05-15T17:12:36+08:00
draft: false
---

tl;dr: 本站即将迁移至 Pandoc + React 技术栈。查看[目前的新网站](https://panda.ayayaya.org)。

## 再见，Hugo

经过了不到一年的使用，我决定彻底放弃 Hugo 极其相关技术栈。

主要原因是：

- Haskell 社区有 Pandoc 这一功能强大的文档转换器. 作为长期的函数式编程爱好者和 Haskell 用户，我认为我应该使用 Pandoc 而非 Hugo.
- Hugo 采取一种自有的模板格式，可读性、可维护性不佳。
- Hugo 基本上无法利用 React / Vue 社区的技术栈。
- 不知道是我用的 Love-It 主题的问题还是 Hugo 程序本身的问题，LaTeX 代码要使用特有的语法，必须用 `\\` 代替 `\`. 这是我最无法忍受的一点。

因此，我决定迁移到 Pandoc + React 技术栈。

## 技术选型

怎么把 Pandoc 和 React 结合起来，是一个未被探索的问题。我目前采用的方法是

- 写一个 Haskell 程序，把 Pandoc 语法树转换为 jsx （目前被我叫作 `panda`，计划是后续替换成直接封装一个自己的 `pandoc` 程序）
- 用一个 webpack loader, 在这个 loader 里调用上面的程序
- 基于 Next.js，实现静态 HTML 导出

可以看到，某种意义上来说，这就是 MDX 的方案。不过我们的优势是，理论上这套方案可以支持所有 Pandoc 能够解析的 markup 语言，例如 markdown, org-mode, reStructuredText 等等。

## 计划

目前我已经有一个能跑的[实现](https://github.com/ayanamists/Panda.js)，等我有空的时候，我将会把本站完全迁移过去。