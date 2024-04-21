---
title: "SQSとSNSによるPub/SubをCloudFormationで構築"
emoji: "🦛"
type: "tech"
topics: [aws, sqs, amazonsns, cloudformation]
published: true
published_at: 2019-12-30
publication_name: optimind
---

## はじめに

クラウドコンピューティングやマイクロサービスなどが盛り上がる中、スケールアウトや並列化との相性の良さなどからシステム間の通信方式としてPub/Subモデルが注目されています。Pub/Subモデルの特徴などは様々な記事にて語られています。

https://cloud.google.com/pubsub/docs/overview

https://qiita.com/unasaka/items/626d832e7281ed6965ae

本記事では [Amazon SNS](https://aws.amazon.com/jp/sns/) と [Amazon SQS](https://aws.amazon.com/sqs/) によりPub/Subモデルを実現する方法を紹介します。SQSのみでも実現可能ですがSNSを用いることでより柔軟なシステムを構築可能です。SNSを用いるメリットの詳細については以下の記事などをご参照ください。

https://dev.classmethod.jp/cloud/aws/sns-topic-should-be-placed-behind-sqs-queue/

さらに今回はIaC (Infrastructure as Code) の考えやCI/CDの実現、再現性やカスタマイズ性などを考慮し、[AWS CloudFormation](https://aws.amazon.com/cloudformation/) を用いて必要なAWSリソースを定義します。

## 各サービスの特徴

### Amazon SNS (Simple Notification System)

Amazon SNSはメッセージのPublishを行うサービスです。トピックのSubscriptionを定義することで、トピックへ発行されたメッセージを種々のサービスへ配信できます。配信先は2019年12月現在で

- Amazon SQS
- HTTP/S
- Eメール
- SMS
- Lambda

が用意されています。メッセージの配信者はAPIなどによりトピックへメッセージを発行（Publish）するだけでよく、その配信先を意識する必要がありません。

### Amazon SQS (Simple Queue Service)

Amazon SQSはメッセージのqueuingサービスです。キューへメッセージを送信することでメッセージが溜められ、メッセージを取得し削除するまでキューに残り続けます。また取得回数が一定数を超えたメッセージを [デッドレターキュー](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) に入れるなどの設定も可能です。


## 構築

今回は以下のようにしてシステムAがシステムBへ情報を届けることを想定します。SQSキューがSNSトピックをSubscribeしている状態を構築し、以下のようなフローで情報が流れていきます。

1. システムAがSNSトピックへメッセージを発行
2. SNSトピックがSQSキューへメッセージを配信
3. システムBがSQSキューをポーリングしメッセージを取得

![システム構成図](/images/pubsub-sqssns/systemimage.webp)

以下ではシステムAとSNSトピックを合わせて **Publisher（パブリッシャー）**、システムBとSQSキューを合わせて **Subscriber（サブスクライバー）** と呼びます。

本記事ではPublisherとSubscriberは **同じAWSアカウント** で **別のCloudFormationスタック** に構築します。同じスタックで定義する場合はOutputsやImportValueなどが不要となりますし、別のAWSアカウント間で構築したい場合はまた設定が異なりますので、そのような場合は以下の記事などをご参照ください。

https://aws.amazon.com/jp/premiumsupport/knowledge-center/sqs-sns-subscribe-cloudformation/


### Publisher側（SNSトピック）の構築

CloudFormationによりSNSトピックを作成します。またSubscriber側からSNSトピックのARNを参照するためにOutputsセクションを定義しています。今回は細かなパラメータは省略していますので、以下の公式ドキュメントを参考にカスタマイズしてください。

- [AWS::SNS::Topic](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-sns-topic.html)
- [Outputs](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html)

```yaml:system_a.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: System A

Resources:
  Topic:
    Type: AWS::SNS::Topic

Outputs:
  TopicArn:
    Value: !Ref Topic
    Export:
      Name: SystemASnsTopicArn
```

このテンプレートをCloudFormationに入力することでSNSトピックの構築が完了します。

```sh:create_system_a.sh
aws cloudformation create-stack \
  --stack-name system-a-stack \
  --template-body file://system_a.yaml
```

### Subscriber側（SQSキュー）の構築

CloudFormationによりSNSトピックをサブスクライブしてデータを格納するSQSキューを作成します。また今回はSQSキューへのアクセス権限（Queueポリシー）やSNSトピックへのサブスクリプションもこちらで定義しています。SNSトピックのARNをPublisher側で出力したものを参照するためImportValue関数を用いています。その他の細かい設定は以下を参照してください。

- [AWS::SQS:Queue](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-sqs-queues.html)
- [AWS::SQS::QueuePolicy](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-sqs-policy.html)
- [AWS::SNS::Subscription](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-sns-subscription.html)
- [Fn::ImportValue](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html)

```yaml:system_b.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: System B

Resources:
  Queue:
    Type: AWS::SQS::Queue

  SnsSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: sqs
      Endpoint: !GetAtt Queue.Arn
      TopicArn: !ImportValue SystemASnsTopicArn

  QueuePolycy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      PolicyDocument:
        Version: 2012-10-17
        Id: AllowSnsTopicToSendMessage
        Statement:
          - Sid: 1
            Effect: Allow
            Principal: "*"
            Action:
              - sqs:SendMessage
            Resource: "*"
            Condition:
              ArnEquals:
                aws:SourceArn:
                  !ImportValue SystemASnsTopicArn
      Queues:
        - !Ref Queue
```

このテンプレートをCloudFormationに入力することでSQSキューの作成やSNSトピックへのサブスクリプションの構築が完了します。

```sh:create_system_b.sh
aws cloudformation create-stack \
  --stack-name system-b-stack \
  --template-body file://system_b.yaml
```

### 動作確認

以上でSNSトピックとSQSキューの連携が完了しました。AWSコンソールなどからSNSトピックへメッセージを発行することで、SQSキューへメッセージが溜まることが確認できるかと思います。

CLIからの確認であれば以下の形式のコマンドを実行することでSNSトピックへのメッセージが発行できます。

```sh
aws sns publish \
  --topic-arn arn:aws:sns:${AWS_REGION}:${AWS_ACCOUNT_ID}:${SNS_TOPIC_NAME} \
  --message file://${MESSAGE_FILE_NAME}
```

また以下のコマンドによりSQSキューのメッセージが取得できます。

```sh
aws sqs receive-message --queue-url ${SQS_QUEUE_URL}
```

あとはシステムAからSNSへメッセージを発行する部分と、システムBからSQSキューへメッセージを取りに行く部分を作成することでシステム全体が完成します。この部分についてはシステムAおよびシステムBで用いる言語ごとのそれぞれのライブラリのドキュメントをご参照ください。

## まとめ

Amazon SNSとSQSを用いることで高速、高可用性、疎結合なPub/Subモデルのメッセージング機構を構築しました。また今回はCloudFormationによりAWSリソースを定義しました。実際の利用にあたっては今回触れていない細かなパラメーターの設定やデッドレターキューの利用などもぜひご検討ください。

## さいごに

株式会社オプティマインドでは「多様性が進んだ世の中でも、全ての人に物が届く世界を持続可能にする」という物流業界の壮大な社会課題を解決すべく、一緒に働く仲間を大募集中です。会社や業務について話を聞いてみたいという方はお気軽にカジュアル面談をお申し込みください！

https://recruit.optimind.tech/#join-us


