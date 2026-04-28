---
title: "DockerアプリをGitHub Actions × AWS Fargateで自動デプロイしてみた"
emoji: "🔰"
type: "tech"
topics: ["Docker", "AWS", "デプロイ", "CICD", "GithubActions"]
published: true
published_at: 2026-05-11 06:00
publication_name: "secondselection"
---

## はじめに

Dockerでアプリを開発し、CI/CDを利用してAWSデプロイまでを行いました。その過程で学んだこと、苦労したことをまとめます。
これからWebアプリ開発をしてみたい、CI/CDを用いたWebアプリ開発に興味がある方の参考になれば幸いです。
Dockerについては書くことが無数にあるので、私が抑えておくべきだと考えた点を抜粋しています。

:::details CI/CD

**CI（Continuous Integration）**
コードが共有リポジトリにプッシュされたタイミングで、自動的にビルドとテストを実行するプロセス。品質向上に繋がる。

**CD（Continuous Delivery/Deployment）**
CIでテストされたコードを、自動的に本番環境へ反映可能な状態にする、または実際に自動でデプロイするプロセス。

どちらも本来は手動で行っていた作業をコードによって自動化する手法です。

:::

開発にあたって、以下のUdemy講座を参考に学習を進めました。特にDockerの説明が非常に分かりやすかったです。

https://www.udemy.com/course/ok-docker/

## 想定読者

- CI/CDを用いたWebアプリ開発に興味がある
- Web開発におけるデプロイまでの流れを理解したい
- Dockerの基礎知識がある方

## システム開発の流れ

はじめにシステム開発の流れをおさらいします。設計フェーズは省略し、開発→CI→CDの順で解説します。

![開発フェーズ（環境構築・開発・コミット）からCI（GitHub Actionsでビルド・テスト）、CD（AWSへデプロイ）までの流れ](/images/docker_web/dev_flow.png)

## システム構成

今回デプロイするシステムは以下のような構成になります。

![ユーザーがインターネット経由でAWS FargateのnginxコンテナにアクセスしReactコンテナと連携するシステム構成図](/images/docker_web/prod.png)

## 使用したAWSサービス

今回はDockerを重点的に学んだので、AWSサービスについては軽くまとめます。

**ECR(Elastic Container Registry)**
コンテナイメージの「保管庫」になります。CIでテストを通してビルドしたイメージを保存します。

**ECS(Elastic Container Service)**
コンテナを管理するサービスです。

**Fargate**
コンテナを実際に動かすサーバーレスのエンジンになります。ECR上からイメージを取得してコンテナを実行します。

各サービスの詳細は以下記事を参照ください。

https://zenn.dev/secondselection/articles/beginner-aws-ecs#1.-%E3%81%AF%E3%81%98%E3%82%81%E3%81%AB

## コンテナ作成の流れ

Dockerコンテナを動かすまでの流れは、大きく以下の3ステップです。

1. **Dockerfileを作成する** — どんなコンテナを作るかを記述したテキストファイル
2. **`docker build`でimageを作成する** — Dockerfileを元に、コンテナの「型」となるimageをビルド
3. **`docker run`でコンテナを起動する** — imageから実際に動くコンテナを生成

![Dockerfileからdocker buildでDocker imageを作成し、docker runでContainerを起動する3ステップの図](/images/docker_web/docker_flow.png)

次節以降ではDockerfileから順に見ていきます。

## Dockerfileの作成

Dockerfileは**Docker image**を作成するためのファイルです。
Dockerを利用する一番のメリットは、環境が違っても同じように動作するアプリを作成できることです。
例えばWindows(WSL)上で作成したアプリをMac上でも同じように動作させることができます。自分の環境では動いたのに、他の人の環境で動かない状態を解消します。
通常は環境の違いを修正する必要がありますが、Dockerによってその手間が省けます。

### docker image

Docker imageはコンテナを作成するための型になります。
Docker imageを用意する方法は2種類あります。

1. Docker Hubのようなイメージレジストリからpull
2. Dockerfileから作成する

1で取得したイメージをDockerfileでカスタマイズする方法も良く利用されます。

:::details Docker Hubとは
イメージレジストリと呼ばれるimageを登録、取得できるサービスです。
Docker界隈のGitHubというイメージです。

https://hub.docker.com/r/akogut/docker-pyftpdlib

誰でも登録できるため、可能な限りpull数やスター数を確認して、Officialのイメージを利用しましょう。
![imageのイメージ](/images/docker_web/official.png)

実際にマルウェアが組み込まれたケースもあります。

:::

### レイヤー構造

Dockerにはレイヤー構造という概念があります。イメージレイヤーとコンテナレイヤーがありますが、イメージレイヤーに着目します。

#### イメージレイヤー

イメージをビルドする際に作成されるレイヤーです。
Dockerfileのコマンドごとにレイヤーが作成され、キャッシュとして蓄積されます。キャッシュによって再ビルド時に時間を削減できます。

![RUN命令ごとにLayer1〜5が積み重なるイメージレイヤー構造の図](/images/docker_web/layer.png)

#### イメージ容量を削減する

以下の2つのコマンドを比べてみましょう。
一方は複数RUNを記載して、もう一方は&&で繋げています。

```dockerfile
FROM ubuntu:22.04
RUN apt update
RUN apt install -y curl
RUN apt install -y vim
CMD ["bash"]
```

```dockerfile
FROM ubuntu:22.04
RUN apt update && \
    apt install -y curl vim
CMD ["bash"]
```

後者のほうがイメージも小さく、初回ビルドも高速です。一方で、コマンドの一部を変更するとレイヤー全体が再作成されること、ひとまとまりになることでデバッグが困難になることがデメリットです。
デバッグ時は前者を利用して原因を特定する手法が推奨されます。

![RUN命令を&&にまとめることでレイヤーが3層に削減された比較図](/images/docker_web/layer2.png)

Dockerfileのベストプラクティスについては書ききれませんので、適宜以下を参照ください。

https://docs.docker.com/build/building/best-practices/

## Docker Composeの作成

通常nginxベースのコンテナとReactベースのコンテナを作成する場合、それぞれを管理する必要があります。

Docker Composeを利用すると、複数コンテナをまとめて管理できます。

例として、APIサーバー（api）とフロントエンド（web）の2コンテナを同時に起動する`compose.yml`を示します。

```yaml
services:
  api:
    container_name: api
    build:
      context: ./api
      target: base
    ports:
      - 8080:8080
    tty: true
    volumes:
      - ./api:/workspace:cached

  web:
    container_name: web
    build:
      context: ./web
      target: base
    ports:
      - 3000:3000
    environment:
      - REACT_APP_API_SERVER=http://localhost:8080/api
    tty: true
    volumes:
      - ./web:/workspace:cached
    depends_on:
      - api
```

`docker compose up`の一発で両方のコンテナが立ち上がり、`depends_on`によりapiが起動した後にwebが起動します。webからapiへのアクセス先は`environment`で渡しているため、コードに環境依存の値を埋め込まずに済みます。

今回の構成では利用していませんが、DBとWebアプリのコンテナがある場合、このファイルにDBのユーザーとパスワードを登録しておくことで、両方のコンテナで同一の値を利用できます。
なお、ユーザー名やパスワードは.envから取得するなどの対策が必要です。
`depends_on`の注意点としてコンテナの起動順を保証するだけでDBへの接続を保証しないことです。DBへの接続を保証したい場合は、`healthcheck`やアプリから接続失敗時にリトライする処理を入れることが有効です。

### Docker network

コンテナは環境を隔離できるメリットがある一方で、コンテナ間の通信が難しいです。

Docker networkを利用することでコンテナ間の通信を実現します。

Docker Composeで作成したコンテナが期待通りに動作することを確認できたら、GitHub Actionsを利用してデプロイします。

## CI/CDを設定する

今回はGitHub Actionsを利用しましたが、GitLabやAWS CodePipeline / CodeBuildなど多様なサービスが対応しています。
CI/CDを設定することでコードがプッシュされた際にテストやビルド、デプロイが行われます。
今回のような自動デプロイのほか、ベースイメージに変更がないか確認するために定期的にビルドする、といった活用方法もあります。

簡単な例として、`main`ブランチへのpushをトリガーに、DockerイメージをビルドしてECRへプッシュするGitHub Actionsのworkflowを示します。

```yaml
# .github/workflows/deploy.yml
name: DockerイメージをビルドしてECRへプッシュ

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      # リポジトリのコードを取得
      - name: コードをチェックアウト
        uses: actions/checkout@v4

      # GitHubのSecretsに登録したAWSキーで認証
      - name: AWS認証情報を設定
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1

      - name: ECRへログイン
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # イメージをビルドしてECRへプッシュ
      - name: イメージをビルド・プッシュ
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: my-app
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

リポジトリのSecretsにAWSのアクセスキーを登録しておけば、pushのたびに自動でビルドとプッシュが走ります。デプロイ（CD）まで含める場合は、続けて`aws-actions/amazon-ecs-deploy-task-definition`などを使ってECSのタスク定義を更新します。

## デプロイされたアプリを確認する

デプロイされたアプリを確認します。
無事にアクセスして動作確認が出来ました。

![デプロイ後のアプリ画面。「Hello App」タイトルとCall APIボタン、Hello World!!テキストが表示されている](/images/docker_web/app.png)

## 苦労した点

苦労した点は主にデプロイ(CI/CD)でした。

### 1. imageがpullできない

DockerfileでECR Public Galleryからベースのイメージを利用していたのですが、以下のようなエラーが発生しました。

```txt
CannotPullContainerError: pull image manifest has been retried 7 time(s): failed to resolve ref public.ecr.aws/nginx/nginx:latest: failed to do request: Head "https://public.ecr.aws/v2/nginx/nginx/manifests/latest": dial tcp 75.2.101.78:443: i/o timeout
```

原因としてはセキュリティグループのアウトバウンドルールでした。
HTTPS(443番ポート)を0.0.0.0/0に対して全開放することで解決しました。

https://gallery.ecr.aws/

### 2. パブリックipにアクセスできない→インバウンドルールを変更

デプロイ後のパブリックIPにアクセスできない問題が発生しました。

こちらも1と同様に、インバウンドルールのHTTP(80番ポート)を0.0.0.0/0に対して全開放することで解決しました。

### 3. 指定したimageがない

利用したかったイメージをDocker Hubからpullしようとしましたが、メンテナンスが停止しており実質的に利用できず、ビルドに失敗しました。ここで代替イメージをAIに質問して鵜呑みにしたのが私の失敗で、提案されたイメージでもビルドに失敗。最終的にイメージを調べ直して解決しました。教訓はイメージレジストリのサイトにてイメージの説明をよく読み、既存のイメージと差分がないことを確認することです。

### 4. リージョンの間違い

続いてはECR上で作成したリポジトリにイメージをプッシュする際のエラーです。

```txt
name unknown: The repository with name '***' does not exist in the registry with id

Error: Process completed with exit code 1.
```

原因としてはリポジトリを東京`ap-northeast-1`で作成したのにバージニア北部`us-east-1`でプッシュしていたことでした。
作成したリージョンとイメージをプッシュするリージョンは統一する必要があります。

## 学んだこと

### 1. 原因の切り分けが最重要

DockerやCI/CD、AWSなど複数のサービスを利用していることで原因の特定が難しかったです。エラーメッセージから推測することで多くは解決しましたが、コンテナのデプロイ時にエラーメッセージが表示されない事象に直面しました。その際にまずローカルでコンテナが正しく動作しているか確認しなおすことで、イメージの問題でビルドに失敗していると特定できました。この経験から、原因の切り分けのためにログやエラーメッセージは必須だと痛感しました。

### 2. イメージは古くなる・消える

イメージレジストリから取得している場合、コードが古くてイメージが存在しない場合もあります。またイメージ自体が非公開になる場合もあり、コード自体は正しくても動かない事態になりうると学びました。

### 3. タイプミス対策はダブルチェック

CI/CDの設定にあたって環境変数を多く設定しました。またAWS上のサービス名などタイプミスが発生しうる状況も多くありました。当然ですが、コピー&ペーストを積極的に活用しダブルチェックすることが重要です。

## おわりに

今回はDockerからCI/CD、デプロイまでを学びました。各工程で設定値が多く、インフラ構築の難しさを体感しました。今回使用したGitHub ActionsはCI/CDの一手法にすぎません。AWSもGUIで構築しましたが、IaC（Infrastructure as Code）という方法でも実現が可能です。改めてDockerはじめ技術の奥深さを痛感しました。
