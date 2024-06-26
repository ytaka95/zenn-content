---
title: "ブラウザのタブ開きすぎ問題への対処"
emoji: "📚"
type: "idea"
topics: ["Chrome", "ブラウザ"]
published: true
published_at: 2023-12-04
publication_name: "optimind"
---

最近PCで使っているブラウザのタブを開きすぎてメモリ不足になってしまい、これは本格的にどうにかしないといけないと思い立ちました。どう「タブ開きすぎ問題」と向き合うかについて、私なりの考え方や運用方法をご紹介します。

:::message
この記事は「このツールさえ使えば解決！」というような安直な内容ではありません。タブが増えるのはなぜなのかを分析し、そこからどういった対処法が考えられるかを考察し提案する記事です。
:::

----

この記事は[株式会社オプティマインド](https://www.optimind.tech/)の社員による「Optimind Advent Calendar 2023」の4日目の記事です。

https://qiita.com/advent-calendar/2023/optimind


## タブ開きすぎ問題

みなさんはWebブラウザを使っていますか？私は使っています。Webブラウザを使う上でよくあるのが**タブ開きすぎ問題**です。

![タブを開きすぎてアイコンしか見えない様子](/images/managing-browser-tabs/toomanytabs.webp)

タブを開きすぎてアイコンしか見えない。タブが溢れて表示すらされない。誰しも経験したことがあるかと思います。

実際タブを開きすぎることによって、私は以下のようなデメリットを感じています。

- パソコンのメモリが枯渇して動作が遅くなる
- 多様なタブを管理することに脳のリソースを取られる
- 閉じてはいけないタブがあるかもしれないと思うとブラウザやOSを再起動することができない

タブが増えすぎることはよくある問題のようで、たとえばChromeの機能としてタブをグループ化したり開いているタブを検索したりする機能があります。ウインドウに収まらないくらい大量のタブがあっても上手く管理することができるすばらしい機能です。しかしどうしてもタブの数は増える一方であり、メモリ枯渇やブラウザを閉じられないという問題が加速していきます。

Chromeには[メモリセーバー](https://support.google.com/chrome/?p=chrome_memory_saver)機能も実装されましたが、アクティブだと判断されてセーブされないタブも一定あり、結局タブが増えれば増えるほどメモリは食われていき、私の環境ではメモリ不足に陥ってしまいました。

## 新進気鋭のブラウザ "Arc"

会社の同僚に話をしたところ「[Arc](https://arc.net/)はいいぞ」とおすすめされました。なにやらいろいろと新しい思想のブラウザのようで、タブが増えすぎることに対してのソリューションも含まれているとのこと。これは期待できそうです。

![Arcのホームページ画面。大きく "Arc is the Chrome replacement I’ve been waiting fo" と書かれている。](/images/managing-browser-tabs/arctoppage.webp)
*Arcのホームページ画面。なかなか攻めている。*

### Arcの思想

ここではすべてを詳しくは紹介しませんが、概念の１つとしてタブやよくアクセスするページなどをFavorites, Pin, Todayで管理するというものがありました。ざっくり説明すると、よく使うページはFavoritesやPinned Tabとしてあらかじめ保存しておき、一時的なタブはTodayに入り一定時間経つと自動で消える、というものです。これによりタブが無限に増えることがなく、常にすっきりとブラウザが使えるという考えのようです。

実際この考えに沿ってFavoritesやPinをちまちまと設定し運用してみたところ、たしかにタブが溜まることなくすっきりしました。よく使うものはPinに保存しておいて、このタブは取っておきたいなと思うものは（一定時間で消えてしまうので）他のツールに移しておいて、といった具合です。

### Arcからの学び

ただここで気づいたことがありました。これってArcじゃなくてもできるのでは？

FavoritesやPinはまさにブックマークです（厳密には同じではありませんが）。またタブが自動で消えるという仕組みはあれど、大事なのは消える事自体ではなく消えてもいいように事前に対応するということです。

つまり、タブ開きすぎ問題において大事なのは**いかにしてタブを消してもいい状態にするか**ということです。

大量のタブをどう管理するかということを追求しても結局タブは増えるだけで本質的な解決にはなりません。むしろChromeのタブグループやタブ検索などの機能はタブを減らすことのモチベーションを低下させタブ増えすぎ問題を助長する悪しき機能群です。そうではなく考えるべきはどうすればタブを消せるのかということだったのです。

これに気づかせてくれたArcありがとう！（ちなみに私はArcの使い勝手にはなじめず今はChromeやSafariを使っています）

## 我々はなぜタブを残すのか

必要なものを消すことはできません。今まで開いていたタブは何かの目的を持って開いていたはずです。まずはそれを整理してみたところ以下の４つに分類されました。

1. ToDo
1. ブックマーク
1. あとで読む
1. もしかしたらあとで使えるかも

それぞれについて説明します。

### 1. ToDo

人から頼まれたタスクや作業中のタスクを忘れてしまわないように、タブを開いたままにしておくパターンです。Googleドキュメントでレビューを頼まれたけどちょっと今すぐ読めないから、後でちらっと目に入って思い出せることを期待して開きっぱなしにしているわけです。つまりやるべきことを忘れないようにする。ToDoと呼ばれるものですね。ブラウザのタブをToDoリストとして使っているわけです。しかし**ToDoはToDoリストやタスク管理ツールで管理すべき**です。

有名なToDoリストアプリとしては[todoist](https://todoist.com/ja)や[Google ToDoリスト](https://support.google.com/tasks/answer/7675772?hl=ja)などがあります。かなり多種多様なアプリやサービスがありますのでお好みのものを探してもらえればと思いますが、個人的にはふとToDoが発生したときにサッと開いて保存できるものがおすすめです。

突発的にドキュメントのレビューを頼まれたならToDoリストアプリを開いて「ドキュメントのレビューをする」というタイトルとともにドキュメントのURLを貼り付けて終わりです。またはタスク管理・プロジェクト管理ツールで管理しているものに関連するならば、そこにメモとしてURLを貼り付けましょう。

これだけで今まで開きっぱなしにしていたタブも心置きなく消すことができます。

### 2. ブックマーク

GitHubとかNotionとかGmailとか勤怠管理とか、頻度高くアクセスするページがあると思います。例えば今取り組んでいるタスクに紐づいたリポジトリを開いておきたいとか、メールチェックするためにGmailを開いておきたいとか。より早くアクセスするためにはタブを開きっぱなしにした方がいいということですね。しかしすぐにアクセスしたいページは**ブックマーク**すべきです。

ただしあまりに細かくブックマークに登録すると、今度はブックマークの整理が大変になる可能性が高いです。そこでおすすめなのがChromeの[アドレスバー](https://support.google.com/chrome/answer/95440)と各サービス内での管理です。

Chromeのアドレスバーではブックマークや履歴をURLやページタイトルなどから検索できます。ブックマークバーから頑張って探す必要もないですし、一時的によくアクセスするけどブックマークするほどではないようなものも履歴から検索できるので大変便利です。

![アドレスバーで "apple" と入れた例。検索ワードの候補と並んで履歴からも候補が出ている。](/images/managing-browser-tabs/addressbar.webp)
*アドレスバーで "apple" と入れた例。検索ワードの候補と並んで履歴からも候補が出ている。*

またNotionやGoogleドライブなど、主要なSaaSではサービス内にブックマーク的な機能（お気に入り、ピン留めなど）があると思いますし、フォルダ構造などを整理することで素早く目的のページにアクセスできるように設計されているはずです。

![Notionでサイドバーにお気に入りが表示されている様子](/images/managing-browser-tabs/notionsidebar.webp)
*Notionはサイドバーにお気に入りを表示できる。*

こうして「ブラウザのタブ一覧から探す」のではなく「アドレスバーで探す」または「サービス内で探す」という運用にしてしまえば、タブを残しておく必要はありません。すぐにアクセスしたいからと思って今まで開きっぱなしにしていたタブも安心して消すことができます。

### 3. あとで読む

Slackで同僚がブログ記事をおすすめしていたけど今ちょっと読む時間が無いから後で読むために一旦タブで開いて置いておこう、というようなパターンです。いわゆる「あとで読む」というやつですね。これも **「あとで読む」機能を持つツール**で管理すべきものです。

有名どころだと[Pocket](https://getpocket.com/ja/)でしょうか。またRSSリーダーである[Feedly](https://feedly.com/)にもこの機能がありますし、Chromeにも[リーディングリスト](https://support.google.com/chrome/answer/7343019)がありますね。ブラウザ拡張として素早く追加できるようにしておくのがおすすめです。

後で読もうと思ったときはこれらに追加してしまえば、今まで開きっぱなしにしていたタブも何も気にせず消すことができます。

### 4. もしかしたらあとで使えるかも

エクセルの便利な関数とか、プログラミング言語の意外な仕様とか、これ知ってたら後で役に立ちそうだから一旦置いておきたいなというパターンです。

よくあるパターンですが、こうして残したタブは実際のところ今後一切読むことは無いです。**今すぐ消して問題ありません**。

よくよく思い出してみてください。このパターンで開いておいたタブをその後読むことがありましたか？そんなことはなかったかと思います。もしかしたらあとで使えるかもと思って残しておいたタブは結局読むことはありません。何度も言います。このパターンで開いておいたタブは、実はもう二度と読むことはありません。

この事実を知ってしまえば、今まで開きっぱなしにしていたタブも清々しい気持ちで消すことができます。

## 実際に運用してみた

以上を踏まえ、今はブラウザを以下のように使っています。

**準備**

- よく使うサービスのトップページまたはよく使うページをブックマーク
    - ブックマークは基本的にブックマークバーではなくフォルダに入れる（アドレスバーでの検索に統一するため）
- 各サービス内でのお気に入りやピン留めなどを整備

**ルール**

- １つの作業完了時にはタブをすべて閉じる
    - 消せないと感じたタブに対しては「ToDo」「ブックマーク」「あとで読む」のどれに該当するかを確認し、適切な場所へ保存してから閉じる
- アクセスしたいページがあるときは新規タブを開き、アドレスバーにURLやタイトルの一部を打って検索する（直接飛ぶか、またはサービスのページに飛んでから辿っていく）

この運用を徹底することで、常にブラウザのタブがスッキリするようになりました。メモリ不足が起きないとか、いつでも再起動できるとか、実用的なメリットはもちろんあるのですが、何よりきれいな状態であることは何にも代えがたい気持ちよさがあります。

## まとめ

ブラウザのタブを開きすぎてしまうという問題に対して、いかにしてタブを消すかが重要であるということを学びました。消せないタブは「ToDo」「ブックマーク」「あとで読む」のいずれかに分類可能であるという分析結果の元、それぞれに対してしかるべき対処を行うことで気持ちよくタブを閉じることができるとわかりました。

タブに限らずあらゆるものは自然と散らかっていきます。おそらく次はToDoリストとあとで読むが溢れてくることでしょう。誰かがそれらをきれいに保つ方法を見つけて記事にしてくれることを願ってやみません。

## さいごに

この記事は [Optimind Advent Calendar 2023](https://qiita.com/advent-calendar/2023/optimind) の4日目の記事でした。

オプティマインドでは「多様性が進んだ世の中でも、全ての人に物が届く世界を持続可能にする」という物流業界の壮大な社会課題を解決すべく、一緒に働く仲間を大募集中です。少しでも興味が湧いた方はカジュアル面談も大歓迎ですので、気軽にお声がけください！

https://recruit.optimind.tech/
