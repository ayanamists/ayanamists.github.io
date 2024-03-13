---
title: "coq-modeで行番号を表示する方法"
date: 2023-12-04T19:14:08+08:00
draft: false
---

{{<  admonition "warning" >}}
この記事はChatGPTによって中国語から翻訳されたもので、いくつかの誤りが含まれているかもしれません。不正確な部分があればご了承ください。
{{< /admonition >}}

行番号（Line Number）の表示は、問題になるべきではないように思えます。vscodeのリポジトリには、行番号の表示をどのように簡単にオフにするかを尋ねる[issue](https://github.com/Microsoft/vscode/issues/52735)があります。Vimユーザーにとっては、`set :number`と一言言えば行番号が表示されます。

そう、Emacsはそういう特別なエディタなのです。私のようなたまに使うユーザーにとって、行番号の表示は実際に問題になります。

## 問題は何か

私が使用しているのはEmacsのディストリビューション、あるいは成熟した設定ファイルである[Spacemacs](https://www.spacemacs.org/)です。Spacemacsの公式ドキュメントによれば、行番号を表示するには、`.spacemacs`で変数を設定するだけでよいようです：

```elisp
(setq-default dotspacemacs-line-numbers t)
```

この文は、`dotspacemacs-line-numbers`を`t`に設定するという意味です。

また、coq言語をサポートするために、私は`coq`レイヤーをロードしました。しかし、coqファイルを開くと、驚くことに、この時点ではcoqのソースコードには行番号が表示されません：

![coq-wtf](https://pic.superbed.cc/item/656efcb312f8d922c1008d4a.png)

通常の直感では、`t`に設定するとすべての状況で行番号が表示されるはずです。では、なぜ行番号が表示されないのでしょうか？

## 一時的な解決策

この問題を解決するのは、原因を理解するよりも簡単です。

少し検索してみると、`spc t n` [^1]で現在のバッファに行番号を表示できることがわかりました。これは確かに問題を解決します。Emacs wikiによれば、このショートカットは実際にはEmacs 26.1以降で導入された行番号コマンド`display-line-numbers-mode`を呼び出しています。

Emacsのコマンドは`m-x`や`spc spc`で実行できます。実際には、コマンドはEmacs Lispの関数で、Programming Elispの[ドキュメント](https://www.gnu.org/software/emacs/manual/html_node/eintr/Interactive-multiply_002dby_002dseven.html)によれば、関数定義に`(interactive)`がある場合、それはコマンドとして呼び出すことができます。

明らかに、Spacemacsもこの関数を使って行番号の設定を行っています。なぜSpacemacsが私の `.v` ファイルに行番号を開けなかったのかはわかりませんが、自分で開くことができます。

`coq` レイヤーは実際には `coq-mode` を使用しています。`coq-mode` はEmacsの[メジャーモード（Major Mode）](https://www.gnu.org/software/emacs/manual/html_node/emacs/Major-Modes.html)の一つで、coq言語のサポートを提供し、多くのショートカットキーを定義しています。

Emacsは [Hooks](https://www.gnu.org/software/emacs/manual/html_node/emacs/Hooks.html) というメカニズムをサポートしており、特定のモードがロードされたときに一連の操作を実行することができます。Spacemacsが `coq-mode` に行番号を開けないのであれば、このメカニズムを使って手動で `display-line-numbers-mode` を実行します。

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

ここでの `dotspacemacs/user-config` はSpacemacsの設定ファイルの一部で、ここで一部のカスタム操作を実行することができます。

これで問題は解決しました。実は、Spacemacsのドキュメントでは `:enabled-for-modes` というオプションも紹介しています。このオプションに `coq-mode` を追加すると、問題が直接解決します。しかし、私が問題の原因を完全に理解するまで、このオプションの存在に気づきませんでした。

```elisp
(setq-default dotspacemacs-line-numbers
  '(:enabled-for-modes coq-mode))
```

## なぜ

私が最初に気づいたのは `dotspacemacs-line-numbers` のコメントです：

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

このコメントでは、この変数を `t` に設定すると、デフォルトで `prog-mode` と `text-mode` の行番号が有効になると述べています。これらのモードは [Basic Major Mode](https://www.gnu.org/software/emacs/manual/html_node/elisp/Basic-Major-Modes.html) と呼ばれています。自然な推測は、`coq-mode` はその一部ではないということです。

coq ファイルを開き、`m-x describe-mode` を実行すると、次のように表示されます：

```
Coq mode defined in ‘coq-mode.el’:
Major mode for Coq scripts.
```

`coq-mode.el` のソースコードを見てみると、秘密は 182 行目にあります：

```elisp
(defun coq--parent-mode ()
  (if coq-use-pg (proof-mode) (prog-mode)))

;;;###autoload
(define-derived-mode coq-mode coq--parent-mode "Coq"
  "Major mode for Coq scripts.
```

この `coq--parent-mode` は明らかに `coq-mode` の親モード（parent mode）で、Emacsのモードにも継承ツリーがあり、一つのモードが他の親モードを継承します。このツリーのルートは確定していませんが、いくつかのケースがあります：

- 自分自身
- Fundamental Mode
- Basic Major Mode

私の環境では `coq-use-pg` の値は `t` で、その親モードは `proof-mode` です。では、`proof-mode` の親モードは何でしょうか？

StackOverflow の[ある質問](https://emacs.stackexchange.com/questions/58073/how-to-find-inheritance-of-modes)では、関連するコードが提供されています：

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

この関数を使うと、いくつかの「通常」のモードの継承パスを得ることができます。例えば、先ほど話した `proof-mode`:

```elisp
ELISP> (derived-modes 'proof-mode)
(prog-mode)
```

はい、`proof-mode` の親モードは `prog-mode` です。もし先ほどのドキュメントを覚えているなら、`prog-mode` は `dotspacemacs-line-numbers` が `t` に設定されているときにも行番号を表示するはずです。

では、一体何が間違っているのでしょうか？この関数に `'coq-mode` を入力してみましょう：

```elisp
ELISP> (derived-modes 'coq-mode)
(coq--parent-mode)
```

出力は `(coq--parent-mode)` で、これは親モードのシンボルであり、評価後の親モードではありません。したがって、`(get 'coq-mode 'derived-parent-mode)` という式は構文上のシンボルしか取得できず、実際の親モードは取得できません。`define-derived-mode` のソースコードを調べると、`(get mode 'derived-mode-parent)` が取得するのは、この文が設定する親モードであることがわかります：

```elisp
(put ',child 'derived-mode-parent ',parent)
```

では、Spacemacsは親モードをどのように取得しているのでしょうか？Spacemacsは `spacemacs/layers/+spacemacs/spacemacs-defaults/funcs.el` の `spacemacs//linum-enabled-for-current-major-mode` 関数で、現在のモードで行番号表示が有効になっているかどうかをチェックします：

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

見ての通り、それは `derived-mode-p` 関数を使って現在のバッファの major mode が予設の `enabled-for-modes` の子モードであるかどうかを問い合わせています。`dervied-mode-p` はEmacsが提供する関数で、[ドキュメント](https://doc.endlessparentheses.com/Fun/derived-mode-p.html)によれば、現在のバッファの主モードが `modes` の子モードであれば、`(derived-mode-p modes)` は非 `nil` 値を返します。

`derived-mode-p` のソースコードをさらにチェックすると、問題の根源がついに明らかになります。

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

`dervied-mode-p` もまた `(get mode 'derived-mode-parent)` を使って特定のモードの親モードを取得していますが、これは `coq-mode` に対しては `coq--parent-mode` を返し、ここで継承チェーンが断絶します。

Emacsのモード定義は最終的には関数を定義します。しかし、もし `'parent` が正しい親モードを評価できなければ、Emacsのモードに対する予設関数は期待した結果を得られません。明らかに、`coq-mode` は `coq--parent-mode` をこのように定義すべきではありません。では、`coq--parent-mode` はどのように定義すべきなのでしょうか？

## 最終的な解決策

Emacs Lispのグローバル変数とグローバル関数は異なり、これはSchemeとは大いに異なります。もし `coq--parent-mode` を直接変数として定義すると、`coq-mode` はロードできなくなります。

```elisp
(defvar coq--parent-mode
  (if coq-use-pg 'proof-mode 'prog-mode))
```

幸運なことに、Emacs Lispには `defalias` という関数があり、これを使って `coq--parent-mode` をエイリアスとして定義することができます。

```elisp
(defalias 'coq--parent-mode
  (if coq-use-pg 'proof-mode 'prog-mode))
```

上記の修正だけで、Spacemacsは `coq-mode` が `prog-mode` から派生していることを自動的に検出し、行番号の表示を開始できます。私はすでに [PR](https://github.com/ProofGeneral/PG/pull/717) を提出し、主分支にマージしました。

[^1]: このようなショートカットキーの記法に慣れていない場合は、[ドキュメント](https://kb.iu.edu/d/aghb)を参照してください。
