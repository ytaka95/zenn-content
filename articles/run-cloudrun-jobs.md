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

Google Cloud には Cloud Run ジョブというサービスがあります。コンテナの実行基盤なのですが、従来の Cloud Run のようにWebサーバーを立ててリクエストを受け付けるような用途ではなく、バッチ処理のように非同期で処理時間が長いようなものの実行に向いているサービスです。

https://zenn.dev/google_cloud_jp/articles/cloudrun-jobs-basic

Cloud Run ジョブはジョブの作成と実行という概念があります。あるジョブに対して何度も実行をすることができ、各実行では複数のタスク（コンテナインスタンス）を立てることができます。タスクの並列実行数の制御や、各タスクが失敗した場合の再試行回数なども指定できます。以下のようなイメージです。

![Cloud Run Jobsのイメージ](/images/run-cloudrun-jobs/cloud_run_concept.webp)

このあたりの概念は以下の記事にも詳しく書かれています。

https://medium.com/google-cloud-jp/cloud-run-jobs-c963a7143367

## やりたいこと

そこそこ処理に時間のかかるアルゴリズムを動かすために、今までは Cloud Functions を使っていたのですが、実行時間上限の問題があり別のインフラを検討しました。さらに

## Dockerイメージの作成

サンプルとして以下のようなDockerfileを使います。

```Dockerfile
FROM ubuntu:latest

ENTRYPOINT ["echo"]
```

ビルドします。

```sh
docker build -t echo .
```

## Artifact Registry への登録

Artifact Registry にリポジトリを作っておきます。この手順は割愛。

Artifact Registry にpushします。

Webコンソールの「設定の手順」にある通り、gcloudコマンドの初期設定を済ませた後にDockerの構成を行う必要があります。

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
denied: Unauthenticated request. Unauthenticated requests do not have permission "artifactregistry.repositories.uploadArtifacts" on resource "projects/cloud-run-jobs-423021/locations/asia-northeast1/repositories/sample" (or it may not exist)
```

sudoなしで実行できるようdockerグループに入るか、`sudo -u [ユーザー名]` として実行するようにしましょう。

## Cloud Run ジョブの作成

以下の手順に沿ってジョブを作ります。

https://cloud.google.com/run/docs/create-jobs?hl=ja#job

手っ取り早いのはCLIでしょうか（`gcloud` コマンドのセットアップが済んでいるはずなので）。Terraformでコード管理してもよいですし、初期設定スクリプトとして各種言語のライブラリで作ってもよいと思います。運用を見越して適切な方法を選んでください。

ということでここでは `gcloud` コマンドで作ってみます（以下は東京リージョンでジョブを作成する例です）。

```sh
gcloud run jobs create echo --image asia-northeast1-docker.pkg.dev/PROJECT_ID/REPO_NAME/echo:latest --region asia-northeast1
```

## Cloud Run ジョブの実行

ジョブの実行時に引数を与えることができます。

```sh
gcloud run jobs execute echo --args="hello"
```
今回の例では `ENTRYPOINT` が `echo` なので、argsとして渡したものがコンテナ内で `echo` の引数となります。つまり `echo hello` が実行され、標準出力に `hello` が出力されます。

引数以外にも環境変数などいろいろな設定を書き換えることができ、一度作成したジョブに対して柔軟に実行をすることができます。

Pythonではまずライブラリをインストールし、run_v2.JobsClientのrun_job関数を実行します。

```sh
pip install google-cloud-run

# timeoutを変更する場合は以下も必要
pip install protobuf
```

```py
from google.cloud import run_v2
# from google.protobuf.duration_pb2 import Duration

client = run_v2.JobsClient()


def run(job_name: str, args: list[str]):
    # 参考: https://cloud.google.com/python/docs/reference/run/latest/google.cloud.run_v2.types.RunJobRequest.Overrides.ContainerOverride
    container_overrides = [
        run_v2.types.RunJobRequest.Overrides.ContainerOverride(
            args=args
        )
    ]

    # 参考: https://cloud.google.com/python/docs/reference/run/latest/google.cloud.run_v2.types.RunJobRequest.Overrides
    overrides = run_v2.types.RunJobRequest.Overrides(
        container_overrides=container_overrides
    )

    # 参考: https://cloud.google.com/python/docs/reference/run/latest/google.cloud.run_v2.types.RunJobRequest
    request = run_v2.RunJobRequest(
        name=job_name,
        overrides=overrides
    )

    client.run_job(request=request)

if __name__ == "__main__":
    run("projects/cloud-run-jobs-423021/locations/asia-northeast1/jobs/echo", ["hello", "python"])
```

参考: [run_job | Class JobsClient](https://cloud.google.com/python/docs/reference/run/latest/google.cloud.run_v2.services.jobs.JobsClient#google_cloud_run_v2_services_jobs_JobsClient_run_job)

