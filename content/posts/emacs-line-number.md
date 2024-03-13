---
title: "在 coq-mode 下开启行号显示"
date: 2023-12-04T19:14:08+08:00
draft: false
---

行号（Line Number）的显示，似乎不应该成为一个问题。vscode 仓库里还有一个 [issue](https://github.com/Microsoft/vscode/issues/52735) 在询问怎么方便地关闭行号显示。而对于 Vim 用户来说，一句 `set :number` 就可以开启行号显示。

没错，Emacs 就是这么特别的编辑器。对我这样偶尔用一用的用户来说，行号显示也确实是一个困扰我的问题。

## 问题是什么

我使用的是一个 Emacs 的发行版，或者说一个比较成熟的配置文件，[Spacemacs](https://www.spacemacs.org/). 在 Spacemacs 的官方文档里，要开启行号显示，似乎只需要在 `.spacemacs` 里设置一下变量就可以了：

```elisp
(setq-default dotspacemacs-line-numbers t)
```

这句话的意思就是，把 `dotspacemacs-line-numbers` 设置为 `t`.

同时，为了支持 coq 语言，我还装载了 `coq` 层（layer）. 但如果打开一个 coq 文件，你也许会惊奇地发现，这时 coq 源代码是没有行号的：

![coq-wtf](https://pic.superbed.cc/item/656efcb312f8d922c1008d4a.png)

按照正常的直觉，设置为 `t` 应该是会在所有情况下显示行号的。那么，为什么没有显示行号呢？

## 临时解决方案

解决这个问题，其实比搞清楚原因要容易。

稍加搜索，我发现 `spc t n` [^1]可以在当前 buffer 打开行号，这确实可以解决问题。按照 Emacs wiki 的说法，这个快捷键调用的实际上是 Emacs 26.1 版本后引入的行号命令 `display-line-numbers-mode`.

Emacs 的命令可以通过 `m-x`，或者 `spc spc` 执行。实际上，一个命令就是一个 Emacs Lisp 的函数，参考 Programming Elisp 的[文档](https://www.gnu.org/software/emacs/manual/html_node/eintr/Interactive-multiply_002dby_002dseven.html)，只要一个函数的函数定义里有 `(interactive)`，那么它就可以当成一个命令调用。


显然，Spacemacs 也是通过这个函数进行行号相关设置。虽然不知道因为什么原因，Spacemacs 没有给我的 `.v` 文件开启行号，但是我们可以自己开启。

`coq` 层实际上使用了 `coq-mode`. `coq-mode` 是一个 Emacs 的[主模式（Major Mode）](https://www.gnu.org/software/emacs/manual/html_node/emacs/Major-Modes.html)，它给 coq 语言提供了支持，并定义了一大堆快捷键。

Emacs 支持一种叫做 [Hooks](https://www.gnu.org/software/emacs/manual/html_node/emacs/Hooks.html) 的机制，它允许我们在某个模式加载时，执行一系列操作。既然 Spacemacs 不会给 `coq-mode` 开启行号，我们就通过这种机制手动执行 `display-line-numbers-mode`.

```elisp
(defun dotspacemacs/user-config ()
  "Configuration for user code:
This function is called at the very end of Spacemacs startup, after layer
configuration.
Put your configuration code here, except for variables that should be set
before packages are loaded."
  (add-hook 'coq-mode-hook (lambda () (display-line-numbers-mode 1)))
)
```

这里的 `dotspacemacs/user-config` 是 Spacemacs 配置文件的一部分，在这里可以执行一些自定义操作。

这当然就解决了问题。其实，Spacemacs 在文档里还提到了一个选项 `:enabled-for-modes`. 在这个选项里加上 `coq-mode` 可以直接解决问题。不过我直到搞清楚了问题的来源，才发现了有这个选项。

```elisp
(setq-default dotspacemacs-line-numbers
  '(:enabled-for-modes coq-mode))
```

## 为什么

我首先注意到的是 `dotspacemacs-line-numbers` 的注释：

```elisp
;; Control line numbers activation.
;; If set to `t', `relative' or `visual' then line numbers are enabled in all
;; `prog-mode' and `text-mode' derivatives. If set to `relative', line
;; numbers are relative. If set to `visual', line numbers are also relative,
;; but only visual lines are counted. For example, folded lines will not be
;; counted and wrapped lines are counted as multiple lines.
;; This variable can also be set to a property list for finer control:
;; (default nil)
```

这段注释里提到，如果把这个变量设置为 `t`，那么它会默认在 `prog-mode` 和 `text-mode` 里开启行号。这两个 mode 被称为 [Basic Major Mode](https://www.gnu.org/software/emacs/manual/html_node/elisp/Basic-Major-Modes.html). 一个很自然的猜想是, `coq-mode` 不是它们之一。

打开一个 coq 文件，执行 `m-x describe-mode`，会看到：

```
Coq mode defined in ‘coq-mode.el’:
Major mode for Coq scripts.
```

进入 `coq-mode.el` 的源代码查看一下，玄机就在第 182 行：

```elisp
(defun coq--parent-mode ()
  (if coq-use-pg (proof-mode) (prog-mode)))

;;;###autoload
(define-derived-mode coq-mode coq--parent-mode "Coq"
  "Major mode for Coq scripts.
```

这个 `coq--parent-mode` 显然就是 `coq-mode` 的父模式（parent mode），Emasc 的模式也有一个继承树，一个模式继承另一个父模式，这棵树的根不确定，有几种情况：

- 自己
- Fundamental Mode
- Basic Major Mode

`coq-use-pg` 的值在我的环境里为 `t`，它的父模式是 `proof-mode`. `proof-mode` 的父模式又是什么呢？

StackOverflow 的[一个问题](https://emacs.stackexchange.com/questions/58073/how-to-find-inheritance-of-modes)提供了相关代码：

```elisp
(defun derived-modes (mode)
  "Return a list of the ancestor modes that MODE is derived from."
  (let ((modes   ())
        (parent  nil))
    (while (setq parent (get mode 'derived-mode-parent))
      (push parent modes)
      (setq mode parent))
    (setq modes  (nreverse modes))))
```

用这个函数，确实可以得到一些“正常”模式的继承路径，比如我们刚刚说的 `proof-mode`:

```elisp
ELISP> (derived-modes 'proof-mode)
(prog-mode)
```

是的，`proof-mode` 的父模式就是 `prog-mode`. 如果你还记得刚才的文档的话，`prog-mode` 在 `dotspacemacs-line-numbers` 设置为 `t` 的时候也应该显示行号。

那么，究竟是什么地方错了呢？不妨给这个函数输入 `'coq-mode`:

```elisp
ELISP> (derived-modes 'coq-mode)
(coq--parent-mode)
```

它的输出是 `(coq--parent-mode)`，这只是父模式的符号，而不是求值后的父模式。可见，`(get 'coq-mode 'derived-parent-mode)` 这个表达式只能拿到语法上的符号，而非它真正的父模式。检查 `define-derived-mode` 的源代码，可以发现 `(get mode 'derived-mode-parent)` 拿到的是这句话设置的父模式：

```elisp
(put ',child 'derived-mode-parent ',parent)
```

那么，Spacemacs 是这样得到父模式的吗？Spacemacs 在 `spacemacs/layers/+spacemacs/spacemacs-defaults/funcs.el` 的 `spacemacs//linum-enabled-for-current-major-mode` 函数中检查是否在当前模式下开启行号显示：

```elisp
(defun spacemacs//linum-enabled-for-current-major-mode ()
  "Return non-nil if line number is enabled for current major-mode."
  (let* ((disabled-for-modes
          (spacemacs/mplist-get-values dotspacemacs-line-numbers
                                       :disabled-for-modes))
         (user-enabled-for-modes
          (spacemacs/mplist-get-values dotspacemacs-line-numbers
                                       :enabled-for-modes))
         ;; default `enabled-for-modes' to '(prog-mode text-mode), because it is
         ;; a more sensible default than enabling in all buffers - including
         ;; Magit buffers, terminal buffers, etc. But don't include prog-mode or
         ;; text-mode if they're explicitly disabled by user
         (enabled-for-modes (or user-enabled-for-modes
                                (seq-difference '(prog-mode text-mode)
                                                disabled-for-modes
                                                #'eq)))
         (enabled-for-parent (or (and (equal enabled-for-modes '(all)) 'all)
                                 (apply #'derived-mode-p enabled-for-modes)))
         (disabled-for-parent (apply #'derived-mode-p disabled-for-modes)))
    (or
     ;; special case 'all: enable for any mode that isn't specifically disabled
     (and (eq enabled-for-parent 'all) (not disabled-for-parent))
     ;; current mode or a parent is in :enabled-for-modes, and there isn't a
     ;; more specific parent (or the mode itself) in :disabled-for-modes
     (and enabled-for-parent
          (or (not disabled-for-parent)
              ;; handles the case where current major-mode has a parent both in
              ;; :enabled-for-modes and in :disabled-for-modes. Return non-nil
              ;; if enabled-for-parent is the more specific parent (IOW derives
              ;; from disabled-for-parent)
              (spacemacs/derived-mode-p enabled-for-parent disabled-for-parent)))
     ;; current mode (or parent) not explicitly disabled
     (and (null user-enabled-for-modes)
          enabled-for-parent            ; mode is one of default allowed modes
          disabled-for-modes
          (not disabled-for-parent)))))
```

可以看到，它通过 `derived-mode-p` 函数询问当前 buffer 的 major mode 是不是预设的 `enabled-for-modes` 的子模式。`dervied-mode-p` 是 Emacs 提供的一个函数，[文档](https://doc.endlessparentheses.com/Fun/derived-mode-p.html)声称，如果当前 buffer 的主模式是 `modes` 的子模式，那么 `(derived-mode-p modes)` 将返回非 `nil` 值。

再检查 `derived-mode-p` 的源代码，我们终于发现了问题的根源。

```elisp
(defun provided-mode-derived-p (mode &rest modes)
  "Non-nil if MODE is derived from one of MODES or their aliases.
Uses the `derived-mode-parent' property of the symbol to trace backwards.
If you just want to check `major-mode', use `derived-mode-p'."
  (while
      (and
       (not (memq mode modes))
       (let* ((parent (get mode 'derived-mode-parent))
              (parentfn (symbol-function parent)))
         (setq mode (if (and parentfn (symbolp parentfn)) parentfn parent)))))
  mode)

(defun derived-mode-p (&rest modes)
  "Non-nil if the current major mode is derived from one of MODES.
Uses the `derived-mode-parent' property of the symbol to trace backwards."
  (apply #'provided-mode-derived-p major-mode modes))
```

`dervied-mode-p` 同样通过 `(get mode 'derived-mode-parent)` 来得到某个模式的父模式，而这对于 `coq-mode` 来说会得到 `coq--parent-mode`, 继承链在这里就断了。

Emacs 中的模式定义最后确实是定义了一个函数。然而，如果 `'parent` 不能求值出正确的父模式，Emacs 对于 mode 的预设函数就会得不到期望的结果。显然，`coq-mode` 不应该这样定义 `coq--parent-mode`. 那么，到底应该怎么定义 `coq--parent-mode` 呢？

## 最终解决方案

Emacs Lisp 的全局变量和全局函数是不同的，这和 Scheme 很不一样。如果直接把 `coq--parent-mode` 定义为一个变量，会使得 `coq-mode` 无法加载。

```elisp
(defvar coq--parent-mode
  (if coq-use-pg 'proof-mode 'prog-mode))
```

万幸的是，Emacs Lisp 提供了一个叫做 `defalias` 的函数，它可以把 `coq--parent-mode` 定义为一个 alias.

```elisp
(defalias 'coq--parent-mode
  (if coq-use-pg 'proof-mode 'prog-mode))
```

上面的修改就足以使得 Spacemacs 可以自动侦测到 `coq-mode` 派生自 `prog-mode`，从而开启行号显示。我已经提交了一个 [PR](https://github.com/ProofGeneral/PG/pull/717)，并合入了主分支。

[^1]: 如果你不熟悉这种快捷键记法，可以参考[文档](https://kb.iu.edu/d/aghb)
