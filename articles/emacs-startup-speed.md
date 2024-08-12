---
title: "Emacsの起動速度を上げてみる"
emoji: "🚅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
publication_name: optimind
---

会社で毎週エンジニア交流会というものを開催しています。社内LTとかハッカソンとかツール紹介とかしています。先日は便利ツールやスクリプトなどの紹介回だったのですが、こんなことがありました。

---

私「Emacsの起動速度が気になるので、こういう工夫をしています（ﾄﾞﾔ」

誰か「なるほど。そもそもVimは起動が速いので我々には関係ない話ですね。」

私「🥺」

※注：これはお互いの信頼があった上でのコントです。実際にこんな辛辣なコメントが飛び交う場ではありません。

---

今までなんとなくEmacsは起動が遅いと思って使っていたのですが、正直もう何年もEmacsの起動時間チューニングなんてしていなかったので、この機会に自身の知識と設定まわりのアップデートを図ろうと思い立ちました。今回は私が今まで気になっていたEmacsの起動速度という問題に対して、どんな対応方法があるのか調査しどれくらい速くなるのか実験してみようと思います。

## 前提

私がいろいろ調べてまとめるまでもなく、Emacsのプロフェッショナルの方々がいろいろとまとめてくださっています。本記事は体験記でしかないので、より網羅的に学びたい方はこれらを熟読することをおすすめします。

以下はtakeさんの記事。基本的にこれに沿って進めていきます。

https://zenn.dev/takeokunn/articles/56010618502ccc

以下は上記の記事でも多く参照されているzk_phiさんの本で、こちらも大変参考になります。

https://zenn.dev/zk_phi/books/cba129aacd4c1418ade4/viewer/a53ba0ad0d729886a1dc

## 改善前

以降では `M-x emacs-init-time` を使って起動時間を測定します。私の環境ではマイクロ秒単位で表示されましたが、以下ではわかりやすく四捨五入してミリ秒で記載していきます。

一旦現状では以下の通りです。

```text
emacs-init-time: 558ms
```

まだ始まる前であれですが、意外と速いなと思いました。昔はもっと遅かったような。Emacsの進化か、端末の進化か、あるいはどこかで設定を見直したんだったか。何はともあれ、ここからさらなる高速化ができないかを検討していきます。

ちなみに、現状は [leaf](https://github.com/conao3/leaf.el) を使ってパッケージ管理しています。ほぼGitHubのReadmeにある通りの設定をしています。leafによってバイトコンパイルや遅延ロードなどが行われている（はず）という前提でお読みください。

::: details leafの設定
```lisp
(eval-and-compile
  (customize-set-variable
   'package-archives '(("org" . "https://orgmode.org/elpa/")
                       ("melpa" . "https://melpa.org/packages/")
                       ("gnu" . "https://elpa.gnu.org/packages/")))
  (package-initialize)
  (unless (package-installed-p 'leaf)
    (package-refresh-contents)
    (package-install 'leaf))

  (leaf leaf-keywords
    :ensure t
    :init
    ;; optional packages if you want to use :hydra, :el-get, :blackout,,,
    ;; (leaf hydra :ensure t)
    ;; (leaf el-get :ensure t)
    ;; (leaf blackout :ensure t)

    :config
    ;; initialize leaf-keywords.el
    (leaf-keywords-init)))
```
:::

元記事に記載されている以下の通りのイメージかと思います。

> - 100ms 〜 1000ms
>     - パッケージ管理ツールで高速化をすると大体この辺になる
>     - そこそこ頑張る必要がある

（leafにお任せなので頑張ってないですが）

ちなみに環境は以下の通りです。Homebrewで `brew install emacs` で入れたものを使っています。

- GNU Emacs 29.4 (build 1, aarch64-apple-darwin23.4.0) of 2024-06-23
- Macbook Air (M1, 2020)

普通に使う分には起動速度自体に不満は無いのですが、例えばGitのCommit時などにパッとエディタを立ち上げようとすると、これだと若干待たされる感があります（その場合、よっぽどinit.elをフルで読み込む必要は無いと思いますが）。できることならもっと速度を詰めていきたいところです。ただし普段の使い勝手は損ねたくないので、パッケージ管理としてleafはそのまま使っていきたいと思います。

## （余談）emacsclient

実は今まで全く知らなかったのですが、emacsclientというものがあるんですね。デーモンでemacsを立てておき、必要なときにemacsclientから接続して使うという方法です。たしかにこれだと（クライアントの）起動は爆速になりそうですね。

ただ今回は今までの使い勝手をなるべく変えたくないことと、そもそもの起動速度を上げたいということが目的となっている取り組みなので、emacsclientの使用は除外します。この機能自体はとてもすごいと思いますし、実用上はこれで解決なのではないかと思うくらいです。

## NativeComp

今までEmacsはHomebrewで `brew install emacs` でインストールしていたものを使っていました。これを手元のマシンでビルドしたものに置き換えます。ソースからビルドすることもできますが、HomebrewからNativeComp対応のものをインストールすることもできるので、手っ取り早くこちらを使用してみます。

今回は [Emacs plus](https://github.com/d12frosted/homebrew-emacs-plus) というものを試してみます。私はGUI版は使っていないので `--without-cocoa` をつけています。

```
$ brew tap d12frosted/emacs-plus
$ brew install emacs-plus@29 --with-native-comp --without-cocoa
```

私は既にHomebrewでEmacsを入れていたので、 コマンドへのシンボリックリンク `emacs` が重複し上書きされませんでした。起動する際はフルパスで `/opt/homebrew/Cellar/emacs-plus@29/29.4/bin/emacs -nw` のようにします。

`M-x describe-variable [RET] system-configuration-features` をしてみると、ちゃんと `NATIVE_COMP` の文字列が見えます。

```
ACL GMP GNUTLS JSON LCMS2 LIBXML2 MODULES NATIVE_COMP NOTIFY KQUEUE PDUMPER SQLITE3 THREADS TREE_SITTER XIM ZLIB
```

さて、では結果はというと。

```text
emacs-init-time: 803ms
```

あれ…？ `558ms -> 803ms` と、むしろ遅くなっています。パッケージをインストールし直したりしましたが変わらず。`eln-cache` に `*.eln` ファイルがあることも確認できています。私の環境では起動速度への良い影響はありませんでした（起動後の挙動については調べていないので、そちらで恩恵が受けられる可能性はありますが）。

今回は起動速度にフォーカスしますので、これ以降はNativeComp版ではなく元の `brew install emacs` でインストールしたものを使っていきます。

## init.elをコンパイルする

次は `init.el` ファイルをコンパイルしてみます。ためしに `M-x byte-compile-file` でバイトコンパイルしてみます。結果は以下の通り。

```text
emacs-init-time: 557ms
```

`558ms -> 557ms` と、これはさすがに誤差の範囲かと思います。意外と効かないですね。私の `init.el` が空行を入れても約600行で複雑な処理もほぼないので、そもそもそこまで重くないのでしょう。この方法を採用するならファイルの変更があるたびに再コンパイルをするような処理を書かないといけなくなるので、ほぼ効果が無いのであればこの方法は見送ろうと思います。

## Magic File Name を起動中は無効にする

[元記事](https://zenn.dev/zk_phi/books/cba129aacd4c1418ade4/viewer/dcebc13578d42055f8a4#gc-を減らす)およびその参照先の [Emacs の起動時間を""詰める""](https://zenn.dev/zk_phi/books/cba129aacd4c1418ade4/viewer/dcebc13578d42055f8a4#magic-file-name-を一時的に無効にする) にあるとおり `init.el` を読み込む間だけ Magic File Name の機能を無効化します。

```text
emacs-init-time: 440ms
```

`557ms -> 440ms` とこれは結構効果がありました。少なくとも私の使い方において特にデメリットも無いのでこれは良いですね。

## GCを減らす

こちらも[元記事](https://zenn.dev/takeokunn/articles/56010618502ccc#gcの設定)および[その参照先](https://zenn.dev/zk_phi/books/cba129aacd4c1418ade4/viewer/dcebc13578d42055f8a4#gc-を減らす)で紹介されている通りです。メモリは十分あるので、GCのしきい値を上げておきます。無限にしてしまうと万が一の場合が怖いので、そこそこの値にしてみました。

```lisp
(setq gc-cons-threshold (* 512 1024 1024))
```

結果は以下。

```text
emacs-init-time: 326ms
```

`440ms -> 326ms` でした。これも効果ありですね。

## まとめ

あらためてEmacsの起動速度を見直し、以下を実施してみました。

- NativeComp版の Emacs plus を使う: `558ms -> 803ms (+245ms)`
- init.elをバイトコンパイルする: `558ms -> 557ms (-1ms)`
- Magic File Name を起動中は無効にする: `558ms -> 440ms (-118ms)`
- GCを減らす: `440ms -> 326ms (-114ms)`

結果として `558ms` から `326ms` まで高速化できました。`EDITOR=emacs` で使えるかというと個人的にはちょっと微妙なラインですが、それ以外の使い方なら十分良いかなと思います。

### 余談

ちなみに私はGitのCommit時のエディタ用に以下の設定をしています。

```bash
EDITOR="/path/to/emacs -nw -q -l ~/.emacs.d/init_light.el"
```

つまり、最低限の設定のみを記述した軽量版のinitファイルを作り、それを読み込ませることで高速に起動させています。外部パッケージはほぼ読み込んでおらず、設定内容はキーバインドや見た目の設定くらいです。これだと2ms程度で起動します。軽量版の設定をメンテするコストが発生しますが、そんなにしょっちゅう触るものでもないのでこの方法は結構気に入っています。
