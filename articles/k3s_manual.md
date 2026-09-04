---
title: "WSL2でk3sに入門しよう！（サンプルコード付き）"
emoji: "🗄️"
type: "tech"
topics: ["k3s", "k8s","docker", "wsl2"]
published: true
published_at: 2026-09-21 06:00
publication_name: "secondselection"
---

## はじめに

本記事ではwsl環境でk3sをハンズオン形式で動かしながら概念を学べることを意識しています。
k3sを通してk8sの概念を掴むことを目的とします。
なお、wsl2の導入手順は扱いません。
HTTP接続をするアプリを題材として扱います。
k8sは多くのリソースから成り立っているため、本記事ではすべてのリソースを網羅していません。

## 対象読者

Dockerは触ったことがあるがKubernetesは未経験の人
Windows11+WSL2上のUbuntuが使える状態

### k3sとは

k8sをエッジコンピューター向けに軽量化したKubernetesディストリビューションです。

Kubernetesの一般的な短縮形が「K8s」であることから、それをもじって「K3s」と命名されました。8はkとsの間の文字数（k「ubernete」s）を表します。K3sという言葉は、K3sの要件である「K8sから5つの要素を変更（削除）して軽量化（5 less than K8s）した」ことを表すために、Kubernetesの文字の総数（10）を半分に割り、8を3に置き換えました。
引用元：https://www.sysdig.com/jp/learn-cloud-native/what-is-k3s

#### k8s(k3s)の構成について

k8sはあるべき状態をマニフェスト形式(YAML)で宣言します。このあるべき状態を維持し続けるのがメリットになります。
k8sはクラスター形式で、マスターノードとワーカーノードで構成されます。
スケーリングを前提としているため、マスターノードが複数のワーカーノードを管理します。
マスターノードはcontroleplaneとも呼ばれ複数のコンポーネントから成り立ちます。
まずはマスターノードが様々な指示を出してワーカーノードがコンテナの作成などをする理解でよいでしょう。

参考記事
https://qiita.com/tadashiro_ninomiya/items/6e6fea807b2a16732b5b#kubernetes%E3%81%A8%E3%81%AF

#### k3sをインストール

```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -
```

インストールすることでUbuntuがそのまま1台のノードになります。
curl -sfL https://get.k3s.io：「K3s」の公式インストールスクリプトを取得
K3S_KUBECONFIG_MODE="644"：`K3S_KUBECONFIG_MODE="644"` は、クラスタ管理者に相当する権限を持った接続情報を、そのマシンの全ユーザーが読める状態にする。1人で使う学習用の WSL 環境だから割り切れるもので、複数人が触るサーバーや本番のクラスタでは、ファイル権限を絞る、用途ごとに権限を制限した接続情報を配る、といった設定が要る。

#### kubectlコマンドを打ってみる

kubectlコマンドは先述したk8sクラスターを操作するためのCLIツールです。イメージとしては`docker ps`のような使い方になります。

```bash
kubectl get node
```

結果

```bash
ad-sakaguchi@n0075-pc:~/container_devkit$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
n0075-pc   Ready    control-plane   86d   v1.35.5+k3s1
```

#### naespaceを作成する

クラスター内部はnamespaceというリソースで仮想的に区別がされています。
```bash
kubectl create namespace k3s-tutorial
kubectl config set-context --current --namespace=k3s-tutorial
```

#### Podを作成する

k8sではPodが最小単位になります。Dockerではコンテナを最小単位として作成しますが、Podはコンテナを1つ以上内包したものになります。
各PodにIPが割り振られ、コンテナはこのIPを共有します。そのため同一Pod内のコンテナ間はlocalhostで接続できます。
イメージとしてはdockercomposeのようなもので、特徴としては使い捨てであることです。
そのためPodを削除し、再作成するとPodのIPは変わります。またクラスター内のどのノードに作成されるかもシステム側が制御するため、ホストが変わる場合もあります。


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: whoami-pod
spec:
  containers:
    - name: whoami
      image: traefik/whoami
      ports:
        - containerPort: 80
```

#### Serviceを定義する

Pod への接続を安定して提供する仕組みです。ユーザーのリクエストを適切なPodへルーティングする役割を担います。先述した通り、Podは使い捨ての概念であるため、再作成のたびにIPが変わってしまいます。特定のPodにIPで接続していると、再作成のたびにIPを特定する手間が発生します。
3種類のタイプから成り立ち、3種類の接続に対応しています。

以下2点は表にしたい

タイプ
LoadBalacer：クラスタのロードバランサ―
NodePort：各ノードのポート
ClusterIP：複数Podのエンドポイントを束ねる

LoadBalacerを作成するとNodePort,ClusterIPが作成され、NodePortを作成するとClusterIPが作成される依存？親子関係にあります。
ローカルでの動作確認であればNodePort、ClusterIPで事足ります。負荷分散をしたい場合はLoadBalacerを利用しましょう。

接続方式
TCP
UDP
SCTP

HTTP,HTTPS接続には後述します、Ingressというリソースを利用します。



#### Deploymentを定義する

Podをあるべき状態に保つための仕組みです。
システムエラーにより特定のPodが落ちてしまった場合に、あるべき状態に修正します。
またアクセス負荷が増えた場合にもスケーリングすることが出来ます。

#### 接続する

## おわりに

