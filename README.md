# zenn-content

これは私のZenn ([zenn.dev/ytaka95](https://zenn.dev/ytaka95)) での記事を管理するリポジトリです。

## 使い方

### 前提

npmがインストールされている

### 環境構築

リポジトリをクローンする

```sh
git clone git@github.com:ytaka95/zenn-content.git
```

Zenn CLI のインストール (package.jsonおよびpackage-lock.jsonで管理している)

```sh
npm install
```

### 新しい記事を書く

対象のブランチを作成する（仕組み上必須ではない）

```sh
git switch main
git pull
git checkout -b article/xxx
```

CLIで記事のファイルを作成する

```sh
npx zenn new:article \
  --slug xxx \
  --title xxx \
  --type tech/idea \
  --emoji ⭐️ \
  --published true \
  --publication-name optimind
```

mainにマージすると公開される

### ローカルでプレビュー

```sh
npx zenn preview
```

### その他のZenn CLIの使い方

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)
