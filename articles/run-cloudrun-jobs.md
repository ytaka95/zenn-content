---
title: "Cloud Run Jobs をPythonから起動する"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gcp, googlecloud, cloudrun, cloudrunjobs]
published: true
publication_name: optimind
---

本記事では Google Cloud の Cloud Run ジョブをPyhtonから実行する例を紹介します。実行の際に環境変数や引数を設定することも可能です。

## Cloud Run ジョブとは

Google Cloud には Cloud Run ジョブというサービスがあります。コンテナの実行基盤なのですが、従来の Cloud Run のようにWebサーバーを立ててリクエストを受け付けるような用途ではなく、バッチ処理のように非同期で処理時間が長いようなものの実行に向いているサービスです。AWSで言うとBatch (on Fargate) が近いサービスかと思います。

https://zenn.dev/google_cloud_jp/articles/cloudrun-jobs-basic

Cloud Run ジョブはジョブの作成と実行という概念があり、あるジョブに対して何度も実行をすることができます。各実行では複数のタスク（コンテナインスタンス）を立てることができ、タスクの並列実行数の制御や各タスクが失敗した場合の再試行回数なども指定できます。以下のようなイメージです。

![Cloud Run Jobsのイメージ](/images/run-cloudrun-jobs/cloud_run_concept.webp)

このあたりの概念は以下の記事にも詳しく書かれています。

https://medium.com/google-cloud-jp/cloud-run-jobs-c963a7143367

## やりたいこと

今までは Cloud Functions でそこそこ処理に時間のかかるアルゴリズムを動かしていたのですが、実行時間上限の問題があったため別のインフラを検討しました。Cloud Functions 第２世代（つまり Cloud Run サービス）も考えましたが、処理時間が伸びるので呼び出し元は同期的にレスポンスを待たないようにしたく、今回は Cloud Run ジョブを採用しようと考えました。

処理の開始はユーザーからのAPIリクエストをトリガーとしたいので「ジョブを開始する」という処理をプログラムで作る必要があります。さらにユーザーからの入力に対して処理を行いたいので「実行ごとに処理を変える」ということも必要です。

そこで、あらかじめDockerイメージに紐づいたジョブを作成した上で、ユーザーのリクエストがあるたびにそれに応じた引数を渡しながら実行する、というコードを書きました。ここではPythonを例にしています。

## 準備

ジョブを実行するには以下の準備が必要です。

- Dockerイメージの作成
- Artifact Registry への登録
- ジョブの作成

これらを終えた上でジョブを実行するコードを用意することで、好きなタイミングで任意の引数を与えたジョブを実行することができます。

### Dockerイメージの作成

ここではサンプルとして以下のようなDockerfileを使います。

```Dockerfile
FROM ubuntu:latest

ENTRYPOINT ["echo"]
```

ビルドします。

```sh
docker build -t echo .
```

### Artifact Registry への登録

事前にArtifact Registry にリポジトリを作っておきます（この手順は割愛します）。以下では `asia-northeast1-docker.pkg.dev` （東京リージョン）にリポジトリを作ったと仮定して書いていきます。

Webコンソールの「設定の手順」にある通り、gcloudコマンドの初期設定を済ませた上でDockerの構成を行う必要があります。

```sh
gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```

そしてタグ付けした上でpushします。

```sh
docker tag echo:latest ${REPOSITORY_NAME}/echo:latest
docker push ${REPOSITORY_NAME}/echo:latest
```

ちなみにハマりがちポイントとして、dockerコマンドをsudoをつけて実行していると内部のgcloudコマンドもsudo先のユーザーとして実行してしまうため、認証に失敗します。

```text
denied: Unauthenticated request. Unauthenticated requests do not have permission "artifactregistry.repositories.uploadArtifacts" on resource "projects/PROJECT/locations/LOCATION/repositories/REPOSITORY" (or it may not exist)
```

sudoなしで実行できるようdockerグループに入れておくか、`sudo -u [ユーザー名]` として実行するようにしましょう。

### Cloud Run ジョブの作成

以下の手順に沿ってジョブを作ります。

https://cloud.google.com/run/docs/create-jobs?hl=ja#job

手っ取り早いのはCLIでしょうか（`gcloud` コマンドのセットアップが済んでいるはずなので）。Terraformでコード管理してもよいですし、初期設定スクリプトとして各種言語のライブラリで作ってもよいと思います。運用を見越して適切な方法を選んでください。

ここでは `gcloud` コマンドで作ってみます（以下は東京リージョンでジョブを作成する例です）。

```sh
gcloud run jobs create echo --image asia-northeast1-docker.pkg.dev/PROJECT_ID/REPO_NAME/echo:latest --region asia-northeast1
```

## Cloud Run ジョブの実行

これで準備は整いました。ここからジョブの実行をするコードを書いて行きます。

ジョブの実行時には引数を与えることができます。一旦わかりやすく `gcloud` コマンドを例に出しますが以下のように実行できます。

```sh
# echoというジョブを "hello" という引数で実行する
gcloud run jobs execute echo --args="hello"
```

今回の例では `ENTRYPOINT` が `echo` なので、argsとして渡したものがコンテナ内で `echo` の引数となります。つまり `echo hello` が実行され、標準出力に `hello` が出力されます。

引数以外にも環境変数などいろいろな設定を書き換えることができ、一度作成したジョブに対して柔軟に実行をすることができます。

Pythonで実行するにはまずライブラリをインストールします。

```sh
pip install google-cloud-run

# timeoutを変更する場合は以下も必要
pip install protobuf
```

そしてrun_v2.JobsClientのrun_job関数を実行します。以下は `"hello python"` という引数を与えるだけのシンプルな例です。

```py
from google.cloud import run_v2

def run(job_name: str, args: list[str]):
    request = run_v2.RunJobRequest(
        name=job_name,
        overrides=run_v2.types.RunJobRequest.Overrides(
            container_overrides=[
                run_v2.types.RunJobRequest.Overrides.ContainerOverride(
                    args=args
                )
            ]
        )
    )
    client.run_job(request=request)

if __name__ == "__main__":
    run(job_name="projects/PROJECT/locations/asia-northeast1/jobs/echo",
        args=["hello", "python"])
```

ここでは以下の３つのクラスが出てきます。それぞれでいろいろな設定が可能ですので詳しくは各ドキュメントをご参照ください。

- [run_v2.types.RunJobRequest](https://cloud.google.com/python/docs/reference/run/latest/google.cloud.run_v2.types.RunJobRequest)
- [run_v2.types.RunJobRequest.Overrides](https://cloud.google.com/python/docs/reference/run/latest/google.cloud.run_v2.types.RunJobRequest.Overrides)
- [run_v2.types.RunJobRequest.Overrides.ContainerOverride](https://cloud.google.com/python/docs/reference/run/latest/google.cloud.run_v2.types.RunJobRequest.Overrides.ContainerOverride)

以下は環境変数、実行するタスク数、各タスクのタイムアウト値を変更する例です（タイムアウト値の設定には [google.protobuf.duration_pb2.Durationクラス](https://googleapis.dev/python/protobuf/latest/google/protobuf/duration_pb2.html) を使います）。

```py
from google.cloud import run_v2
from google.protobuf.duration_pb2 import Duration

def run(job_name: str, args: list[str]):
    request = run_v2.RunJobRequest(
        name=job_name,
        overrides=run_v2.types.RunJobRequest.Overrides(
            container_overrides=[
                run_v2.types.RunJobRequest.Overrides.ContainerOverride(
                    args=args,
                    env=[run_v2.types.EnvVar(name="env1", value="value1")]
                )
            ],
            task_count=1,
            timeout=Duration(seconds=1)
        )
    )
    client.run_job(request=request)
```

参考: [run_job | Class JobsClient](https://cloud.google.com/python/docs/reference/run/latest/google.cloud.run_v2.services.jobs.JobsClient#google_cloud_run_v2_services_jobs_JobsClient_run_job)

ちなみに実行されたタスクの設定が想定どおり上書きされているかどうかはWebコンソールの実行履歴の中の「構成」から確認可能です。

![Webコンソールから設定内容を見たイメージ](/images/run-cloudrun-jobs/task_config.webp)


## まとめ

ユーザーリクエストを非同期に長時間実行するために、PyhtonからCloud Run ジョブを引数を渡して実行するコードを書きました。任意の引数や環境変数を与えられるのでリクエスト内容に応じた処理を柔軟に実行可能です。さらに Cloud Storage を介するなどすればファイルの受け渡しも可能です。Google Cloud 上での処理の可能性が広がります。
