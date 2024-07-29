---
title: "CloudFormationでLambdaの自動デプロイ環境を構築する"
emoji: "🦒"
type: "tech"
topics: [aws, lambda, codepipeline, cloudformation]
published: true
published_at: 2019-08-19
publication_name: optimind
---

## はじめに

近年CI/CDの重要性が各所で叫ばれています。AWS Lambdaを用いたサービスを開発する際にも、例えばGitHubにプッシュしたコードが自動でLambdaへデプロイされればCI/CDの実現に繋がります。本記事ではAWSのCloudFormationとCodePipelineを用いて、GitHubからLambda（＋DynamoDB）までの自動デプロイ環境の構築方法を紹介します。

以下の記事にてコンソールからCodePipelineを設定する方法が解説されています。本記事ではCodePipeline自体もCloudFormationで作成する方法をご紹介します。

- [CodePipeLineを使ってLambdaへの自動デプロイ - Qiita](https://qiita.com/RyujiKawazoe/items/38411271230f9e112253)
- [LambdaをCodePipeline(CodeCommit→CodeBuild→CloudFormation)でCDする方法 - Qiita](https://qiita.com/yasumo/items/75917d38b7e3ded9f806)

## 関連するコンポーネントの説明

### GitHub

GitHub自体の説明は他に任せます。今回はGitHub上でコードを管理し、そのリポジトリへのプッシュをトリガーに自動デプロイされる環境を構築します。

### CodePipeline

環境の構築やテスト、デプロイを自動で実行するマネージドサービスです。今回はコードを取得するSourceフェーズ、ビルドを行うBuildフェーズ、デプロイを行うDeployフェーズを利用します。デプロイにはソースコードだけではなくLambdaやDynamoDBなどのインフラに関する情報も必要ですので、ここには後述するCloudFormationを利用します。

またCodePipelineではデプロイするためのファイルやバイナリをアップロードするS3や、各種AWSリソースを操作するためのIAMも定義する必要があります。これらはすべてCloudFormationのテンプレートに記載します。

### CloudFormation

AWSのリソースをテンプレートと呼ばれるテキストで定義し、構築や更新ができるサービスです。AWSリソースをCLIやWebコンソールから手動で構築・更新を繰り返していると、気づいたら「今どんな設定がされているかわからない」「同じ環境を再現できない」「リソース同士の依存関係がわからない」などの問題が生じます。関連するリソース群をまとめてテンプレートとして保存しておくことでCloudFormation経由で簡単に全リソースをデプロイすることができます。さらにテンプレートはJSONまたはYAML形式であるため、ソースコードと同じようにGitなどで差分管理することも可能です。

また紛らわしいのですが、上記のCodePipelineもAWSリソースであり、CloudFormationテンプレートで定義可能です。以下ではCodePipelineのテンプレートとLambda+DynamoDBの２つのテンプレートを準備します。またLambda+DynamoDBのテンプレートはソースコードと同じリポジトリで管理することとします。

## 構成の概要

パイプラインは以下の流れで動作します。

1. GitHubをトリガーに処理を開始
2. GitHubからコードを取得
3. CodeBuildによりビルド
4. CloudFormationによりLambda関数やDynamoDBテーブルを作成

各フェーズ間のファイル（アーティファクト）のやりとりにはS3を用います。パイプラインの作成に伴いGitHubリポジトリとの紐付けが行われ、以降はGItHubへpushするだけでLambdaなどがデプロイされる環境が出来上がります。以下の図がパイプライン全体のイメージです。

![デプロイパイプラインのイメージ](/images/cfn-lambda-deploy-pipeline/pipeline.webp)

このパイプラインはWebコンソールからも作成できるのですが、パイプライン作成においても人的ミスや属人化などを防ぐために、CloudFormation経由で作成することとします。CloudFormationテンプレートにはパイプラインの定義だけでなく、アーティファクトの保存先となるS3バケットや、Lambdaなどをデプロイするために必要なIAMロールなども定義します。以下ではtemplate_pipeline.ymlとして記述しています。

![パイプラインを定義するCFnテンプレートのイメージ](/images/cfn-lambda-deploy-pipeline/template_pipeline.webp)

パイプラインのDeployフェーズではLambda関数やDynamoDBテーブルをCloudFormationにより作成／更新します。LambdaやDynamoDBをCloudFormationテンプレートで定義しておき、ソースコードと一緒にGitHubにて管理します。以下ではtemplate_deploy.ymlとして記述しています。

![デプロイフェーズで適用されるテンプレートのイメージ](/images/cfn-lambda-deploy-pipeline/template_deploy.webp)

## 構築

ここでは実際に自動デプロイをするための環境構築の準備をします。AWSの設定はすべてCloudFormationで行いますので、そのためのテンプレートファイルの準備をしていきます。

ここに載せるコードはGitHubでも公開しています。

https://github.com/ytaka95/blog-cfn-template

### GitHubリポジトリ

GitHubリポジトリにはLambdaへデプロイするコードの他に、Lambda関数やDynamoDBテーブルを定義するテンプレートファイル、パイプラインで参照するパラメータファイルなどを保存しておきます。以下がフォルダ構成です。各ファイルについての説明は[ファイル準備](#ファイル準備)にて記載します。

```
/
├── README.md
├── lambda_handler.py        # Lambda関数で動くコード
└── pipeline_settings
    ├── buildspec.yml        # Buildフェーズで動く内容
    ├── param.json           # パラメータ
    └── template_deploy.yml  # LambdaやDynamoDBを定義するテンプレート
```

### ファイル準備

#### template_pipeline.yml

template_pipeline.ymlではデプロイ用のパイプライン（Pipeline）を定義しています。それ以外にもS3バケットや、各種リソースに割り当てるIAMロールなどを定義しています。

|リソース名|概要|
|---|---|
|ArtifactStoreBucket|アーティファクト保存用S3バケット|
|BuildProject|Buildフェーズで行うビルド|
|PipelineDeployRole|Deployフェーズでtemplate_deploy.ymlをデプロイするための権限を定義したIAMロール|
|PipelineRole|パイプライン自体に与えるIAMロール|
|CodeBuildRole|BuildフェーズのCodeBuildに与えるIAMロール|
|Pipeline|デプロイ用のパイプライン|

Buildフェーズにおいては、実行する内容をbuildspec.ymlというファイルを参照するようにしています。これはGitHubリポジトリに含めており、SourceフェーズでダウンロードしたSourceOutputに含まれています。

CloudFormationテンプレートの具体的な書き方については[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-guide.html)に詳細にまとまっています。

:::details コード (250行)
```yaml:template_pipeline.yml
AWSTemplateFormatVersion: 2010-09-09
Description: CloudFormation Template of Pipeline

Parameters:
  Owner:
    Type: String
  Repo:
    Type: String
  OAuthToken:
    Type: String
    NoEcho: true
  Branch:
    Type: String
  ModuleName:
    Type: String
  ModuleStackName:
    Type: String
  PackagedTemplateFilePath:
    Type: String
    Default: packaged.yml
  DeployParamFile:
    Type: String
    Default: param.json
  BuildSpec:
    Type: String
    Default: pipeline_settings/buildspec.yml

Resources:
  ArtifactStoreBucket:
    Type: AWS::S3::Bucket

  BuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Ref ModuleName
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/ubuntu-base:14.04
        EnvironmentVariables:
          - Name: PACKAGED_TEMPLATE_FILE_PATH
            Value: !Ref PackagedTemplateFilePath
          - Name: S3_BUCKET
            Value: !Ref ArtifactStoreBucket
      Source:
        Type: CODEPIPELINE
        BuildSpec: !Ref BuildSpec

  PipelineDeployRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action: sts:AssumeRole
            Principal:
              Service: cloudformation.amazonaws.com
      Path: /
      Policies:
        - PolicyName: !Sub ${ModuleName}DeployPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:*
                Resource: !Sub arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/*
              - Effect: Allow
                Action:
                  - lambda:*
                Resource: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:*
              - Effect: Allow
                Action:
                  - iam:CreateRole
                  - iam:DeleteRole
                  - iam:GetRole
                  - iam:PassRole
                  - iam:DeleteRolePolicy
                  - iam:PutRolePolicy
                  - iam:GetRolePolicy
                Resource: !Sub arn:aws:iam::${AWS::AccountId}:role/*
              - Effect: Allow
                Action: s3:GetObject
                Resource:
                  - !Sub arn:aws:s3:::${ArtifactStoreBucket}
                  - !Sub arn:aws:s3:::${ArtifactStoreBucket}/*

  PipelineRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Action: sts:AssumeRole
            Effect: Allow
            Principal:
              Service: codepipeline.amazonaws.com
      Path: /
      Policies:
        - PolicyName: CodePipelineAccess
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Sid: S3GetObject
                Effect: Allow
                Action: s3:*
                Resource:
                  - !Sub arn:aws:s3:::${ArtifactStoreBucket}
                  - !Sub arn:aws:s3:::${ArtifactStoreBucket}/*
              - Sid: S3PutObject
                Effect: Allow
                Action: s3:*
                Resource:
                  - !Sub arn:aws:s3:::${ArtifactStoreBucket}
                  - !Sub arn:aws:s3:::${ArtifactStoreBucket}/*
              - Sid: CodeBuildStartBuild
                Effect: Allow
                Action:
                  - codebuild:StartBuild
                  - codebuild:BatchGetBuilds
                Resource: !Sub arn:aws:codebuild:${AWS::Region}:${AWS::AccountId}:project/${ModuleName}
              - Sid: CFnActions
                Effect: Allow
                Action:
                  - cloudformation:DescribeStacks
                  - cloudformation:DescribeChangeSet
                  - cloudformation:CreateChangeSet
                  - cloudformation:ExecuteChangeSet
                  - cloudformation:DeleteChangeSet
                Resource:
                  - !Sub arn:aws:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/${ModuleStackName}/*
              - Sid: PassRole
                Effect: Allow
                Action:
                  - iam:PassRole
                Resource: !GetAtt PipelineDeployRole.Arn

  CodeBuildRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Action: sts:AssumeRole
            Effect: Allow
            Principal:
              Service: codebuild.amazonaws.com
      Path: /
      Policies:
        - PolicyName: CodeBuildAccess
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Sid: CloudWatchLogsAccess
                Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource:
                  - !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/codebuild/*
              - Sid: S3Access
                Effect: Allow
                Action:
                  - s3:PutObject
                  - s3:GetObject
                  - s3:GetObjectVersion
                Resource:
                  - !Sub arn:aws:s3:::${ArtifactStoreBucket}
                  - !Sub arn:aws:s3:::${ArtifactStoreBucket}/*
              - Sid: CloudFormationAccess
                Effect: Allow
                Action: cloudformation:ValidateTemplate
                Resource: "*"

  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: !Sub pipeline-${ModuleName}
      RoleArn: !GetAtt PipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactStoreBucket
      Stages:
        - Name: Source
          Actions:
            - Name: DownloadSource
              ActionTypeId:
                Category: Source
                Owner: ThirdParty
                Version: 1
                Provider: GitHub
              Configuration:
                Owner: !Ref Owner
                Repo: !Ref Repo
                Branch: !Ref Branch
                OAuthToken: !Ref OAuthToken
              OutputArtifacts:
                - Name: SourceOutput
        - Name: Build
          Actions:
            - InputArtifacts:
                - Name: SourceOutput
              Name: Package
              ActionTypeId:
                Category: Build
                Provider: CodeBuild
                Owner: AWS
                Version: 1
              OutputArtifacts:
                - Name: BuildOutput
              Configuration:
                ProjectName: !Ref BuildProject

        - Name: Deploy
          Actions:
            - Name: CreateChangeSet
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: '1'
              InputArtifacts:
                - Name: BuildOutput
              Configuration:
                ActionMode: CHANGE_SET_REPLACE
                RoleArn: !GetAtt PipelineDeployRole.Arn
                StackName: !Ref ModuleStackName
                ChangeSetName: !Sub ${ModuleStackName}-changeset
                Capabilities: CAPABILITY_NAMED_IAM
                TemplatePath: !Sub BuildOutput::${PackagedTemplateFilePath}
                TemplateConfiguration: !Sub BuildOutput::${DeployParamFile}
              RunOrder: '1'
            - Name: ExecuteChangeSet
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: '1'
              InputArtifacts:
                - Name: BuildOutput
              Configuration:
                ActionMode: CHANGE_SET_EXECUTE
                ChangeSetName: !Sub ${ModuleStackName}-changeset
                StackName: !Ref ModuleStackName
              RunOrder: '2'
```
:::

#### buildspec.yml

パイプラインのBuildフェーズで行う内容を定義します。今回はCloudFormationでpackageコマンドを実施します。ここで参照している環境変数「PACKAGED_TEMPLATE_FILE_PATH」と「S3_BUCKET」はtemplate_pipeline.ymlの中のBuildProjectの`EnvironmentVariables`として定義しているものです。

```yaml:buildspec.yml
version: 0.2

phases:
  build:
    commands:
      - |
        aws cloudformation package \
          --template-file pipeline_settings/template_deploy.yml \
          --s3-bucket $S3_BUCKET \
          --output-template-file $PACKAGED_TEMPLATE_FILE_PATH

artifacts:
  files:
    - $PACKAGED_TEMPLATE_FILE_PATH
    - pipeline_settings/*
  discard-paths: yes
```

#### param.json

DeployフェーズにてLambda関数などをデプロイする際にtemplate_deploy.ymlを用いますが、それに対して入力するパラメータを定義したものです。デプロイ用のテンプレートを環境ごとに用意してメンテするのは効率的ではないためこうしています。

```json:param.json
{
  "Parameters": {
    "LambdaFunctionName": "TestFunction",
    "LambdaFunctionHandler": "lambda_handler.lambda_handler"
  }
}
```

#### template_deploy.yml

Lambda関数やDynamoDBテーブルを定義します。またそれらに与えるIAMロールなども定義します。Parametersで定義しているパラメータが、template_pipeline.ymlのDeployフェーズの`TemplateConfiguration`で指定したファイルから読み込まれます（ここでは上記のparam.jsonに相当します）。

:::details コード (83行)
```yaml:template_deploy.yml
AWSTemplateFormatVersion: '2010-09-09'
Description:  Service Infra Build Pipeline

Parameters:
  LambdaFunctionName:
    Type: String
  LambdaFunctionHandler:
    Type: String

Resources:
  LambdaTestFunction:
    Type: AWS::Lambda::Function
    Properties:
      Description: test function
      Environment:
        Variables:
          TABLE_ARN: !GetAtt DynamoDBTestTable.Arn
      FunctionName: !Ref LambdaFunctionName
      Handler: !Ref LambdaFunctionHandler
      MemorySize: 256
      Role: !GetAtt LambdaRole.Arn
      Runtime: python3.6
      Timeout: 10

  DynamoDBTestTable:
    Type: AWS::DynamoDB::Table
    Properties:
      AttributeDefinitions:
      - AttributeName: name
        AttributeType: S
      - AttributeName: key
        AttributeType: S
      - AttributeName: date
        AttributeType: S
      BillingMode: PAY_PER_REQUEST
      GlobalSecondaryIndexes:
        - IndexName: KeyDate
          KeySchema:
          - AttributeName: key
            KeyType: HASH
          - AttributeName: date
            KeyType: RANGE
          Projection:
            ProjectionType: ALL
      KeySchema:
      - AttributeName: name
        KeyType: HASH
      TimeToLiveSpecification:
        AttributeName: expireAt
        Enabled: true

  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal:
              Service:
                - "lambda.amazonaws.com"
            Action: "sts:AssumeRole"

      Policies:
        - PolicyName: !Sub ${LambdaFunctionName}-DynamoDB
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: "Allow"
                Action:
                  - "dynamodb:GetItem"
                  - "dynamodb:Query"
                  - "dynamodb:PutItem"
                  - "dynamodb:UpdateItem"
                Resource: !GetAtt DynamoDBTestTable.Arn
        - PolicyName: !Sub ${LambdaFunctionName}-CloudWatch
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: "Allow"
                Action: "logs:*"
                Resource: "arn:aws:logs:*:*:*"
```
:::

### CodePipelineのデプロイ

ファイルは準備できたので、ここからパイプラインをデプロイします。ここではデプロイにAWS CLIを用いる例を紹介します。

パイプラインからGitHubへアクセスが必要なため、リポジトリのオーナーやリポジトリ名を指定します。またプライベートリポジトリの場合は認証が必要ですので、GitHubのPersonal access tokenを取得しておきます（取得方法については[公式のヘルプページ](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line)を参照）。

```sh
aws cloudformation deploy \
    --stack-name auto-deploy-pipeline \
    --template-file ./template_pipeline.yml \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
    --parameter-overrides \
    OAuthToken=GitHubPersonalAccessToken \
    Owner=GitHubRepoOwnerName \
    Repo=GitHubRepoName \
    Branch=GitHubBranchName \
    ModuleName=deploy-test-module \
    ModuleStackName=test-module
```

### CodePipelineの実行

GitHubの対象のブランチにプッシュすることでパイプラインが動きます。

## まとめ

GitHubからLambdaデプロイまでの自動化を行いました。今回はシンプルな構成にしましたが、CodePipeline, CodeBuildは高機能で、例えばテストの自動化を組み込んだり、予め決めたメールアドレスにデプロイの承認を求めるといったことも実装可能です。このあたりは上記のテンプレートを公式ドキュメントに沿ってカスタマイズしていくことでどんどん実現することが可能です。そのあたりもぜひお試しください。

## さいごに

株式会社オプティマインドでは「多様性が進んだ世の中でも、全ての人に物が届く世界を持続可能にする」という物流業界の壮大な社会課題を解決すべく、一緒に働く仲間を大募集中です。会社や業務について話を聞いてみたいという方はお気軽にカジュアル面談をお申し込みください！

https://recruit.optimind.tech/#join-us



