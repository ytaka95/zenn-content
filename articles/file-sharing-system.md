---
title: "AWSでセキュアなファイル送信システムを作る"
emoji: "🔒"
type: "tech"
topics: ["AWS", "CloudFront", "S3", "CloudFormation"]
published: true
published_at: 2022-12-20
---

# はじめに

みなさんは業務上で自分以外の（特に社外の）人とファイルのやりとりをしたいときどうしていますか？USBメモリに入れる、メールに添付する、DropboxやGoogleドライブなどで共有する、ファイル送信サービスを利用する、などなどいろいろな方法がありますね。今回は最後のファイル送信サービスのようなシステムを、なるべくAWSのマネージドサービスを使いつつ自分で作ってみたいと思います。説明のためにサンプルとなるCloudFormationのコードも随時添えておきます。

:::message
ここでご紹介する方法や実装にセキュリティ上の問題が無いことは保証しておりません。この記事を参考にして実際に試す場合はすべて自己責任でお願いします。
:::


# 背景

先にも書いたとおりファイルの受け渡し方法には以下のように様々なものがあります。

- USBメモリに入れて渡す
- メールに添付して送る
- 普段利用するクラウドストレージサービス（DropboxやGoogleドライブなど）で特定ユーザー向けに共有する
- 普段利用するクラウドストレージサービスでパブリックな共有リンクを発行する
- ファイル送信サービスなどを利用する（ギガファイル便やfirestorageなど）

基本的には「使い勝手」と「セキュリティリスク」のバランスの問題になります。特に業務で用いる場合、情報流出は会社全体としてかなり大きなインシデントとなりますので、後者のセキュリティリスクを重視することが多いのではないかと思います。

ざっくりと各方法の特徴をまとめてみます。

|方法|セキュリティレベル|使い勝手|説明|
|:--|:--|:--|:--|
|USBメモリ|×|△|手渡しや郵送などが必要。データ流出（紛失）のリスクが高く、さらに大量のデータを一度に扱いやすいことも相まってインシデント発生時の被害が大きくなりがち。あまりにリスクが高いため企業として禁止している場合が多い。|
|メール添付|△|○|手軽でセキュリティもそこそこであるが、誤送信などに気づいても送信の取り消しやファイルの削除ができない。送受信者ともに多大な手間をかけてでも微小なセキュリティレベルの向上を図るという目的で、ファイルにパスワードを設定してメールで送るという手法が取られることがある。|
|特定ユーザーとの共有|◎|○|普段のファイル管理の延長で共有可能。認証が入るのでセキュリティレベルは高い。ただし該当サービスに登録している相手にしか送れない。|
|パブリックな共有リンクを発行する|○|◎|普段のファイル管理の延長で共有可能。パブリックであるため有効期限を設けることが望ましい。|
|ファイル送信サービス|○|○|毎回外部サービスにアップロードする必要がある。それ以外は基本的に「パブリックな共有リンクを発行する」と同様。|

## 問題点

セキュリティレベルを考えると、普段利用するクラウドストレージサービス（DropboxやGoogleドライブなど）で __特定ユーザーとの共有__ をするのが一番良さそうです。ただし、この方法は「該当サービスに登録している相手にしか送れない」という大きな問題があります。つまりGoogleドライブで共有するには相手がGoogleアカウントを利用している必要があります。常に使える方法ではないので他の方法も用意しておく必要があります。

次に __パブリックな共有リンクを発行する__ という方法も良さそうです。普段使っているサービスからファイルを選択し、リンクを知っていれば誰でもアクセスできるという設定でリンクを発行すればよいので、なかなか使い勝手も良さそうです。

:::note info
誰でもアクセスできるという設定は一見リスクが高いようにも見えますが、リンク自体が流出する可能性があるということはそもそも渡したデータが流出する可能性があることと同様のため、他の方法より明確にリスクが高いとは言えないでしょう。またリンクの無効化ができるという点でよりセキュアであるとも言えます。
:::

ただこの方法にもデメリットがあります。それはこれらのサービスを __ドメインごとアクセスブロックしている企業がある__ ということです。DropboxやGoogleなどは大変良いサービスを提供しているのですが、世の中にはこれらのサービスをドメインごとブロックしてアクセスできないようにしている企業があります。その理由やメリットは私にはよくわからないのですが存在するのは事実です。

そうすると特定ユーザーと共有することもパブリックな共有リンクを発行することも __常に使えるわけではない__ という根本的な問題を抱えていることになります。

そこで考えられるのが最後の __ファイル送信サービス__ という選択肢です。仕組みとしてはパブリックな共有リンクを発行することと同じなのですが、普段使っているクラウドストレージサービスではなくファイルを送るためだけに別サービスにアップロードするという点が違います。

この方法であればこれまでの問題は解決しやすいのですが、それでも完璧ではありません。そのサービスは自社の重要なファイルを預けるのに十分なセキュリティや信頼を担保したサービスですか？業務に用いることを会社から許可してもらえるように説明できますか？受け取り手に対してあなたの会社がそのサービスを使っていることをリスクと思わないようにきちんと説明できますか？

あなたはただファイルを送りたいだけですが、業務で使うというだけでこれだけの手間をかけなければいけません。使いたいサービスのセキュリティがどうとか、信頼できるかとか、調査するのも説明するのも面倒ですよね。普段からZennを読んでいるイケてるエンジニアであるあなたはきっと思うわけです。「自分で作った方が早い」と。

# 設計と実装

ようやく本題です。自分が保有するファイルをアップロードし、ダウンロードするための一時的なリンクを発行するシステムを作ってみます。

## 要件

以下がざっくりとした要件です。

- アップロードは自社の社員が行う
    - 自分以外もなるべく簡単にリンクの発行ができる
    - 悪意のある行動はしない前提
- ダウンロードは社外の任意の人が行う
    - 受け取り手はリンクを受け取ってクリックするだけで簡単にダウンロードできる
    - 悪意のある攻撃者がいても大丈夫である必要がある
- 期限を過ぎたら自動的にダウンロードできなくする

また、作る上で以下のような意識をしています。

- なるべくマネージドサービスを使う
- なるべくサーバーレスサービスを使う

なるべくマネージドサービスを使うのは ~~好きだからです~~ セキュリティリスクを下げるためです。設計やコーディングをすればするほどバグによって予期しないセキュリティリスクを生む可能性があり、なるべく用意された機能を使う方がそのリスクを低減できます。

また、なるべくサーバーレスサービスを使うのは ~~好きだからです~~ 定常的にかかるコストをなるべく減らしたいからです。お金は大事です。実際今回のものを自分だけが使うのであれば、おそらくほぼ0円で運用できるのではないでしょうか。

ただし以下のような状況ですのでそれについてはご承知おきください。

- ダウンロードリンクのドメインは `cloudfront.net` のため、これをブロックしている環境からはアクセスできません
- サンプルのコードは汎用性にこだわっていないのでそのままで簡単に使えるものではないです
- Webフロント画面などはありません（本当は簡単なものでも作りたかったですが、いかんせん時間とスキルが足りませんでした。フロントエンドができる人は本当にすごい。）
- アップロードしたファイルはシステムの構築者（S3バケットの所有者やCloudFrontに登録した秘密鍵の所有者）が中身を見ることができます。アップロードする人がそれを了承する必要があります。

## 設計

ざっくりとした全体の構成は以下の通りです。

![全体構成](/images/file-sharing-system/koseizu_zentai.webp)

ファイルを置くS3バケットを中心として、AWSの機能でアップロードリンクとダウンロードリンクの発行を行い、最終的に受け取り手はCloudFront経由でファイルをダウンロードします。

S3バケットのCloudFormationテンプレートの例を記載します。

:::details CloudFormationテンプレート

```yml:origin_bucket.yml
OriginBucket: 
  Type: "AWS::S3::Bucket"
  Properties: 
    PublicAccessBlockConfiguration: 
      BlockPublicAcls: "true"
      BlockPublicPolicy: "true"
      IgnorePublicAcls: "true"
      RestrictPublicBuckets: "true"
    OwnershipControls: 
      Rules: 
      - ObjectOwnership: "BucketOwnerEnforced"
```

:::

以下ではそれぞれの手順におけるインフラ構成についてCloudFormationテンプレートを添えつつ詳しく説明します。

### 1. アップロードリンクを取得

![アップロードリンクを取得する部分の構成図](/images/file-sharing-system/koseizu_1_uploadlink.webp)

ファイルの送信者は自分に限らないので、S3バケットへのアクセス権限を持っていない人でもS3にファイルをアップロードできる必要があります。かと言って世界中の誰でもアップロードできるようにしておくのは嫌ですよね。でも利用者全員にIAMロールを付与するのも面倒です。実はAWSの仕組みで期限付きのS3アップロード用URLを発行することができますので、今回はそれを利用します。

具体的な仕組みやコードについては以下を参考にしました。

https://dev.classmethod.jp/articles/create-pre-signed-url-with-lambda/

AWSリソースとしてはS3バケットへPutObjectする権限を持つロールが必要です。このロールをAssumeRoleして署名を付与するので、なるべく最小限の権限にしておきます。

:::details CloudFormationテンプレート

```yml:put_object_role.yml
PutRole: 
  Type: "AWS::IAM::Role"
  Properties: 
    AssumeRolePolicyDocument: 
      Version: "2012-10-17"
      Statement: 
      - Effect: "Allow"
        Principal: 
          AWS: 
          - Ref: "AWS::AccountId"
        Action: 
        - "sts:AssumeRole"
    Policies: 
    -
      PolicyName: 
        Fn::Sub: "PutObject-${OriginBucket}"
      PolicyDocument: 
        Version: "2012-10-17"
        Statement: 
        - Effect: "Allow"
          Action: 
          - "s3:PutObject"
          Resource: 
          - Fn::Sub: "arn:aws:s3:::${OriginBucket}/*"
```

:::

### 2. アップロード

![アップロードする部分の構成図](/images/file-sharing-system/koseizu_2_upload.webp)

アップロードするだけです。参考URLにもある通り、curlコマンドであれば以下の通りにアップロードが可能です。

```bash
curl -X PUT --upload-file $PATH_TO_FILE $URL 
```

このURLを知っている人であれば各自のAWSの認証情報などを一切使うことなくファイルのアップロードができることがわかるかと思います。もちろん有効期限が切れた後はこのURLではアップロードできません。

### 3. ダウンロードリンクを取得

![ダウンロードリンクを取得する部分の構成図](/images/file-sharing-system/koseizu_3_downloadlink.webp)

インフラの設定や署名をする処理などについては以下を参考にしました。

https://zenn.dev/may_solty/articles/807dbad3a30de8

https://dev.classmethod.jp/articles/cloudfront-signed-url-and-cookie-using-python/

必要なAWSリソースは以下の通りです（括弧内は対応するCloudFormationテンプレートの名前）。

- 前出のファイルを置くS3バケットに付与するバケットポリシー(`bucket_policy.yml`）
- CloudFront関連リソース（`cloudfront.yml`）
    - Distribution
    - CachePolicy
    - PublicKey
    - KeyGroup
    - OriginAccessControl

CachePolicyで基本的なキャッシュをオフにしています（既存の Managed policy である `CachingDisabled` を使うことも考えましたが、環境ごとにIDを確認するのが面倒なのでここではリソースを作っています）。

KeyGroupを作ってそこにPublicKeyを登録します。PublicKeyはpem形式にして、CloudFormationテンプレートにそのままベタ貼りする必要があります。手元で試す場合は秘密鍵・公開鍵を各自で作成してください。

秘密鍵は署名のために必要です。今回はLambda関数に組み込むこととしました。Lambda関数のコードやファイルや環境変数は常に暗号化されており[^1]、AWSとしてもセキュアに扱う前提であるので、秘密鍵をコードと一緒に置いておくことは特にセキュリティリスクとなることはないと判断しています。


:::details CloudFormationテンプレート

```yml:bucket_policy.yml
OriginBucketPolicy: 
  Type: "AWS::S3::BucketPolicy"
  Properties: 
    Bucket: 
      Ref: "OriginBucket"
    PolicyDocument: 
      Version: "2008-10-17"
      Id: "PolicyForCloudFrontPrivateContent"
      Statement: 
      - Sid: "AllowCloudFrontServicePrincipal"
        Effect: "Allow"
        Principal: 
          Service: "cloudfront.amazonaws.com"
        Action: "s3:GetObject"
        Resource: 
          Fn::Sub: "${OriginBucket.Arn}/*"
        Condition: 
          StringEquals: 
            AWS:SourceArn: 
              Fn::Sub: "arn:aws:cloudfront::${AWS::AccountId}:distribution/${Distribution}"
```

```yml:cloudfront.yml
Distribution: 
  Type: "AWS::CloudFront::Distribution"
  Properties: 
    DistributionConfig: 
      HttpVersion: "http2"
      IPV6Enabled: "true"
      Origins: 
      -
        Id: 
          Ref: "OriginBucket"
        DomainName: 
          Fn::GetAtt: 
          - "OriginBucket"
          - "DomainName"
        OriginAccessControlId: 
          Fn::GetAtt: 
          - "OriginAccessControl"
          - "Id"
        S3OriginConfig: 
      DefaultCacheBehavior: 
        AllowedMethods: 
        - "GET"
        - "HEAD"
        CachedMethods: 
        - "GET"
        - "HEAD"
        CachePolicyId: 
          Ref: "CachePolicy"
        Compress: "false"
        MinTTL: "0"
        DefaultTTL: "0"
        MaxTTL: "0"
        SmoothStreaming: "false"
        TargetOriginId: 
          Ref: "OriginBucket"
        TrustedKeyGroups: 
        -
          Fn::GetAtt: 
          - "KeyGroup"
          - "Id"
        ViewerProtocolPolicy: "https-only"
      Enabled: "true"
CachePolicy: 
  Type: "AWS::CloudFront::CachePolicy"
  Properties: 
    CachePolicyConfig: 
      DefaultTTL: "0"
      MaxTTL: "0"
      MinTTL: "0"
      Name: "NoCache"
      ParametersInCacheKeyAndForwardedToOrigin: 
        CookiesConfig: 
          CookieBehavior: "none"
        EnableAcceptEncodingBrotli: "false"
        EnableAcceptEncodingGzip: "false"
        HeadersConfig: 
          HeaderBehavior: "none"
        QueryStringsConfig: 
          QueryStringBehavior: "none"
PublicKey: 
  Type: "AWS::CloudFront::PublicKey"
  Properties: 
    PublicKeyConfig: 
      CallerReference: "1b8383b6b30fdf9e005fcc642d72eb74"
      EncodedKey: "-----BEGIN RSA PUBLIC KEY-----\n中略\n-----END RSA PUBLIC KEY-----"
      Name: "publickey"
KeyGroup: 
  Type: "AWS::CloudFront::KeyGroup"
  Properties: 
    KeyGroupConfig: 
      Items: 
      - Ref: "PublicKey"
      Name: "keygroup"
OriginAccessControl: 
  Type: "AWS::CloudFront::OriginAccessControl"
  Properties: 
    OriginAccessControlConfig: 
      Name: 
        Fn::Sub: "OAC-${OriginBucket}"
      OriginAccessControlOriginType: "s3"
      SigningBehavior: "always"
      SigningProtocol: "sigv4"
```
:::

### 4. ダウンロードリンクを送付

![ダウンロードリンクを送付する部分の構成図](/images/file-sharing-system/koseizu_4_share.webp)

ファイルを渡したい人へリンクを送信します。当然ですがこのリンクを知っている人は誰でもファイルをダウンロードできてしまうので流出しないように慎重に取り扱う必要があります。とは言っても有効期限がついていますので、それが切れた後は特に何も考える必要はありません。

URLに含まれるトークンも十分に長く予測不能な文字列ですのでこれ自体が流出しない限りは有効期限内に第三者がアクセスすることは不可能と思ってよいでしょう。

１つ注意としては、このリンクはファイルへのダイレクトリンクですので、例えばSlackに投稿したときのように自動的にリンク先のコンテンツを取得するようなシステムに渡すと対象のファイルがシステム内でダウンロードされることになります。これが直接問題になることがあるかはわかりませんが、よくあるファイル送信サービスとは挙動が違うことは知っておく方が良いかもしれません。

実際に使う場合はこれをラップするWebページなどを作った方が使い勝手はいいかもしれませんね。

### 5. ダウンロード

![ダウンロードする部分の構成図](/images/file-sharing-system/koseizu_5_download.webp)

思う存分ダウンロードしてもらいましょう。ただし先にも書きましたが、ここでのダウンロードリンクのドメインはCloudFrontのもの（おそらく `cloudfront.net` ）です。さすがにこれをブロックしている環境は珍しいと思いますが、必要あればカスタムドメインを利用しましょう。

## 全体

以上のインフラに追加して「アップロードリンクの発行」と「ダウンロードリンクの発行」のLambda関数を追加することでそれぞれのAPIができます。これを使って「ローカルのファイルを指定したらアップロードして期限付きダウンロードリンクを発行する」ようなプログラムを作ることができますね。

他にもシステムとして運用することを考えるとまだまだ改善点がたくさんあります。

- 使いやすいフロントエンド
- ダウンロードリンクにURLパラメータが大量についていて扱いづらい（何よりダサい！）
- リンクの無効化やファイルの削除の仕組み
- システム構築者でもファイルを見られないような権限や鍵の管理の仕組み

改善が進んだら追って更新をしたいと思います！（ひとまずフロント作ってデモサイトくらいは公開したい。。。）

# まとめ

我々はただファイルを送りたいだけなんです。でもその方法を考えるだけでとても大変です。自分で作った方が早いと思うのも無理はありません。そんな方にとってAWSの署名付きURLの仕組みは心強い味方ですね。この方法で構築された方がいればぜひコメントください。また問題や改善点がありそうだという場合もぜひコメントいただけると嬉しいです。

ちなみに私や私の所属する会社でこの方法で運用しようとはしていません。自分で作るのはやっぱり大変ですからね。

# さいごに

オプティマインドでは「多様性が進んだ世の中でも、全ての人に物が届く世界を持続可能にする」という物流業界の壮大な社会課題を解決すべく、一緒に働く仲間を大募集中です。少しでも興味が湧いた方はカジュアル面談も大歓迎ですので、気軽にお声がけください！

https://recruit.optimind.tech/

[^1]: [AWS Lambdaでのデータ保護 | AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/security-dataprotection.html)
