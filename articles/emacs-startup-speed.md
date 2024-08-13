---
title: "Emacsの起動速度を上げてみた"
emoji: "🚅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["emacs"]
published: true
publication_name: optimind
---

今までなんとなくEmacsは起動が遅いと思って使っていたのですが、正直もう何年もEmacsの起動時間チューニングなんてしていなかったので、この機会に自身の知識と設定まわりのアップデートを図ろうと思い立ちました。起動速度向上としてどんな対応方法があるのか調査し、それぞれどれくらい効果があるのか検証してみようと思います。

## 前提

高速化の手法については私がいろいろ調べてまとめるまでもなく、Emacsのプロフェッショナルの方々がまとめてくださっています。本記事は体験記でしかないので、より網羅的に学びたい方はこれらをおすすめします。

https://zenn.dev/takeokunn/articles/56010618502ccc

https://zenn.dev/zk_phi/books/cba129aacd4c1418ade4/viewer/a53ba0ad0d729886a1dc

## 改善前

以降では `M-x emacs-init-time` を使って起動時間を測定します。私の環境ではマイクロ秒単位で表示されましたが、以下ではわかりやすく四捨五入してミリ秒で記載していきます。

一旦現状では以下の通りです。

```text
emacs-init-time: 582ms
```

まだ始まる前ですが、意外と速いなと思いました。昔はもっと遅かったような。Emacsの進化か、端末の進化か、あるいはどこかで設定を見直したのかわかりませんが、何はともあれ、ここからさらなる高速化ができないかを検討していきます。

ちなみに、現状は基本的に [use-package](https://github.com/jwiegley/use-package) を使ってパッケージ管理しており、バイトコンパイルや遅延ロードなどが行われているはずです。元記事に記載されている以下の通りのイメージかと思います。

> - 100ms 〜 1000ms
>     - パッケージ管理ツールで高速化をすると大体この辺になる
>     - そこそこ頑張る必要がある

（ほぼuse-packageにお任せなので頑張ってないですが）

ちなみに環境は以下の通りです。Homebrewで `brew install emacs` で入れたものを使っています。

- GNU Emacs 29.4 (build 1, aarch64-apple-darwin23.4.0) of 2024-06-23
- Macbook Air (M1, 2020)

普通に使う分には起動速度自体に不満は無いのですが、こういうのは速ければ速いほど良いですからね。ただし普段の使い勝手は損ねたくないので、例えばパッケージ管理として use-package はそのまま使っていきたいと思います。

## 不要な設定の掃除

早速あまりtips感が無いのですが、そもそも `init.el` に不要な設定だったり非効率な箇所が無いかをチェックしていきます。今だと使っていないパッケージとかそもそも不要な処理が残っている部分などを4箇所ほど見つけたのでまるっと削除したところ、起動時間は以下の通りになりました。

```text
emacs-init-time: 553ms -> 382ms (-171ms)
```

これだけでもだいぶ改善ですね。定期的な整理整頓がいかに大切かを思い知らされました。

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
emacs-init-time: 382ms -> 530ms (+148ms)
```

あれ？むしろ遅くなっています。パッケージをインストールし直したりしましたが変わらず。`eln-cache` に `*.eln` ファイルがあることも確認できています。私の環境では起動速度への良い影響はありませんでした（起動後の挙動については調べていないので、そちらで恩恵が受けられる可能性はあります）。

今回は起動速度にフォーカスしますので、これ以降はNativeComp版ではなく元の `brew install emacs` でインストールしたものを使っていきます。

## init.elをコンパイルする

次は `init.el` ファイルをコンパイルしてみます。ためしに `M-x byte-compile-file` でバイトコンパイルしてみます。結果は以下の通り。

```text
emacs-init-time: 382ms -> 367ms (-15ms)
```

これは誤差の範囲と言って良いレベルかもしれません。意外と効かないですね。私の `init.el` が空行を入れても約600行で複雑な処理もほぼないので、そもそもそこまで重くないのでしょう。この方法を採用するならファイルの変更があるたびに再コンパイルをするような処理を書かないといけなくなるので（または毎回手動でコンパイルしてもいいですが）、入れた方がいいかどうか微妙なラインです。今回は一応採用しておこうと思います。以下のようなコードを追加しました。

```lisp
;; init.elを保存時に自動でバイトコンパイルする
(defun my-byte-compile-init-file ()
  (when (string= (buffer-file-name) (expand-file-name user-init-file))
    (byte-recompile-file (buffer-file-name))))

(add-hook 'emacs-lisp-mode-hook
          (lambda ()
            (add-hook 'after-save-hook #'my-byte-compile-init-file nil t)))
```


## 起動中は Magic File Name を無効にする

[Emacs の起動時間を""詰める""](https://zenn.dev/zk_phi/books/cba129aacd4c1418ade4/viewer/dcebc13578d42055f8a4#magic-file-name-を一時的に無効にする) にあるとおり `init.el` を読み込む間だけ Magic File Name の機能を無効化します。

```text
emacs-init-time: 367ms -> 325ms (-42ms)
```

`init.el` の処理にも依存すると思いますが、私の環境でもそこそこ効果がありました。 少なくとも私の使い方において特にデメリットも無いのでこれはこのまま採用します。

## GCを減らす

[Emacs の起動時間を""詰める""](https://zenn.dev/zk_phi/books/cba129aacd4c1418ade4/viewer/dcebc13578d42055f8a4#gc-を減らす)で紹介されている通りです。メモリは十分あるので、GCのしきい値を上げておきます。無限にしてしまうと万が一の場合が怖いので、そこそこの値にしてみました。

```lisp
(setq gc-cons-threshold (* 512 1024 1024))
```

結果は以下。

```text
emacs-init-time: 325ms -> 174ms (-151ms)
```

効果はバツグンでした。こちらも特にデメリットは無いと思われるので、入れて損は無いものかと思います。

## まとめ

あらためてEmacsの起動速度を見直し、以下を実施してみました。

- 不要なコードの削除: `553ms -> 382ms (-171ms)`
- NativeComp版の Emacs plus を使う: `382ms -> 530ms (+148ms)`
- init.elをバイトコンパイルする: `382ms -> 367ms (-15ms)`
- 起動中は Magic File Name を無効にする: `367ms -> 325ms (-42ms)`
- GCを減らす: `325ms -> 174ms (-151ms)`

結果として `553ms` から `174ms` まで高速化できました。一番効いたのが不要なコードやパッケージの削除ということでちょっと悲しいですが、これを学びとして今後は定期的に掃除をしようと思います。

### 余談１: emacsclient

実は今まで全く知らなかったのですが、emacsclientというものがあるんですね。デーモンでemacsを立てておき、必要なときにemacsclientから接続して使うという方法です。たしかにこれだと（クライアントの）起動は爆速になりそうですね。

ただ今回は今までの使い勝手をなるべく変えたくないことと、そもそもの起動速度を上げたいということが目的となっている取り組みだったのでemacsclientの使用は選択肢から除外しました。この機能自体はとても良いと思いますし実用上はこれで解決なのではないかと思うくらいです。

### 余談２: 軽量版のinit.el

GitのCommit時などで立ち上がるエディタもEmacsであってほしいですが、そこに数百ミリ秒かかっているとどうしてもちょっと突っかかるような感覚があります。ただ上記で見てきたようにどうしてもEmacsの起動をそこまでチューニングするにはかなりの労力をかけたりある程度の制約を受け入れる必要がありそうです。

そこで私は `init_light.el` として最低限の設定のみを記述した軽量版のinitファイルを作り、それを読み込ませることで高速に起動させています。環境変数として以下の設定をしています。

```bash
export EDITOR="/path/to/emacs -nw -q -l ~/.emacs.d/init_light.el"
```

外部パッケージはほぼ読み込んでおらず、設定内容はキーバインドや見た目の設定くらいです。これだと `emacs-init-time` は2~3ms程度になります。軽量版の設定をメンテするコストが発生しますが、そんなにしょっちゅう触るものでもないのでこの方法は結構気に入っています。
