---
title: "Enabling Line Number Display in coq-mode"
date: 2023-12-04T19:14:08+08:00
draft: false
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

The display of line numbers (Line Number) should not be a problem. There is even an [issue](https://github.com/Microsoft/vscode/issues/52735) in the vscode repository asking how to conveniently turn off the line number display. For Vim users, a simple `set :number` can turn on the line number display.

Yes, Emacs is such a special editor. For occasional users like me, the display of line numbers is indeed a problem that troubles me.

## What is the problem

I am using a distribution of Emacs, or a mature configuration file, [Spacemacs](https://www.spacemacs.org/). In the official Spacemacs documentation, to enable line number display, you only need to set a variable in `.spacemacs`:

```elisp
(setq-default dotspacemacs-line-numbers t)
```

The meaning of this sentence is to set `dotspacemacs-line-numbers` to `t`.

At the same time, to support the coq language, I also loaded the `coq` layer. But if you open a coq file, you may be surprised to find that the coq source code does not have line numbers:

![coq-wtf](https://pic.superbed.cc/item/656efcb312f8d922c1008d4a.png)

According to normal intuition, setting to `t` should display line numbers in all cases. So, why are there no line numbers?

## Temporary Solution

Solving this problem is actually easier than figuring out the reason.

After a little search, I found that `spc t n` [^1] can turn on line numbers in the current buffer, which can indeed solve the problem. According to the Emacs wiki, this shortcut actually calls the line number command `display-line-numbers-mode` introduced after Emacs 26.1 version.

Emacs commands can be executed through `m-x`, or `spc spc`. In fact, a command is an Emacs Lisp function. According to the [documentation](https://www.gnu.org/software/emacs/manual/html_node/eintr/Interactive-multiply_002dby_002dseven.html) of Programming Elisp, as long as there is `(interactive)` in the function definition of a function, it can be called as a command.

Clearly, Spacemacs also sets line numbers through this function. Although I don't know why Spacemacs didn't turn on line numbers for my `.v` files, we can turn them on ourselves.

The `coq` layer actually uses `coq-mode`. `coq-mode` is a [Major Mode](https://www.gnu.org/software/emacs/manual/html_node/emacs/Major-Modes.html) of Emacs, which provides support for the coq language and defines a bunch of shortcuts.

Emacs supports a mechanism called [Hooks](https://www.gnu.org/software/emacs/manual/html_node/emacs/Hooks.html), which allows us to perform a series of operations when a mode is loaded. Since Spacemacs won't turn on line numbers for `coq-mode`, we manually execute `display-line-numbers-mode` through this mechanism.

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

Here, `dotspacemacs/user-config` is part of the Spacemacs configuration file, where you can perform some custom operations.

This of course solves the problem. In fact, Spacemacs also mentions an option `:enabled-for-modes` in the documentation. Adding `coq-mode` to this option can directly solve the problem. However, I didn't discover this option until I figured out the source of the problem.

```elisp
(setq-default dotspacemacs-line-numbers
  '(:enabled-for-modes coq-mode))
```

## Why

The first thing I noticed was the comment for `dotspacemacs-line-numbers`:

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

This comment mentions that if this variable is set to `t`, it will enable line numbers by default in `prog-mode` and `text-mode`. These two modes are known as [Basic Major Modes](https://www.gnu.org/software/emacs/manual/html_node/elisp/Basic-Major-Modes.html). A natural guess is that `coq-mode` is not one of them.

Open a coq file, execute `m-x describe-mode`, and you will see:

```
Coq mode defined in ‘coq-mode.el’:
Major mode for Coq scripts.
```

Enter the source code of `coq-mode.el` to take a look, the mystery lies in line 182:

```elisp
(defun coq--parent-mode ()
  (if coq-use-pg (proof-mode) (prog-mode)))

;;;###autoload
(define-derived-mode coq-mode coq--parent-mode "Coq"
  "Major mode for Coq scripts.
```

This `coq--parent-mode` is obviously the parent mode of `coq-mode` (parent mode), Emasc's mode also has an inheritance tree, a mode inherits another parent mode, the root of this tree is uncertain, there are several situations:

- Itself
- Fundamental Mode
- Basic Major Mode

The value of `coq-use-pg` in my environment is `t`, and its parent mode is `proof-mode`. What is the parent mode of `proof-mode`?

A [question](https://emacs.stackexchange.com/questions/58073/how-to-find-inheritance-of-modes) on StackOverflow provides the relevant code:

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

With this function, you can indeed get the inheritance path of some "normal" modes, such as the `proof-mode` we just mentioned:

```elisp
ELISP> (derived-modes 'proof-mode)
(prog-mode)
```

Yes, the parent mode of `proof-mode` is `prog-mode`. If you still remember the document just now, `prog-mode` should also display line numbers when `dotspacemacs-line-numbers` is set to `t`.

So, what went wrong? Let's input `'coq-mode` into this function:

```elisp
ELISP> (derived-modes 'coq-mode)
(coq--parent-mode)
```

Its output is `(coq--parent-mode)`, which is just the symbol of the parent mode, not the evaluated parent mode. It can be seen that the expression `(get 'coq-mode 'derived-parent-mode)` can only get the symbol in syntax, not its real parent mode. Checking the source code of `define-derived-mode`, you can find that what `(get mode 'derived-mode-parent)` gets is the parent mode set by this sentence:

```elisp
(put ',child 'derived-mode-parent ',parent)
```

So, is this how Spacemacs gets the parent mode? Spacemacs checks whether line number display is enabled in the current mode in the `spacemacs//linum-enabled-for-current-major-mode` function in `spacemacs/layers/+spacemacs/spacemacs-defaults/funcs.el`:

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

As you can see, it asks the current buffer's major mode if it is a child mode of the preset `enabled-for-modes` through the `derived-mode-p` function. `dervied-mode-p` is a function provided by Emacs, and the [documentation](https://doc.endlessparentheses.com/Fun/derived-mode-p.html) claims that if the current buffer's main mode is a child mode of `modes`, then `(derived-mode-p modes)` will return a non-`nil` value.

Upon further inspection of the source code of `derived-mode-p`, we finally found the root of the problem.

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

`dervied-mode-p` also gets the parent mode of a certain mode through `(get mode 'derived-mode-parent)`, but for `coq-mode`, it will get `coq--parent-mode`, and the inheritance chain breaks here.

The final definition of mode in Emacs is indeed a function. However, if `'parent` cannot evaluate the correct parent mode, Emacs' preset function for mode will not get the expected result. Obviously, `coq-mode` should not define `coq--parent-mode` in this way. So, how should `coq--parent-mode` be defined?

## Final Solution

The global variables and global functions of Emacs Lisp are different, which is very different from Scheme. If you directly define `coq--parent-mode` as a variable, it will make `coq-mode` unable to load.

```elisp
(defvar coq--parent-mode
  (if coq-use-pg 'proof-mode 'prog-mode))
```

Fortunately, Emacs Lisp provides a function called `defalias`, which can define `coq--parent-mode` as an alias.

```elisp
(defalias 'coq--parent-mode
  (if coq-use-pg 'proof-mode 'prog-mode))
```

The above modification is enough to make Spacemacs automatically detect that `coq-mode` is derived from `prog-mode`, thus turning on line number display. I have submitted a [PR](https://github.com/ProofGeneral/PG/pull/717) and merged it into the main branch.

[^1]: If you are not familiar with this shortcut notation, you can refer to the [documentation](https://kb.iu.edu/d/aghb)
