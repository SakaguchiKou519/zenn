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

本記事ではWSL2環境でk3sをハンズオン形式で動かしながら、Kubernetesの概念を学ぶことを目的としています。
ゴールは簡易アプリにHTTP接続をしてレスポンスが返ることとします。実際にコマンドを打ちながら進めていきましょう。

なお、以下は本記事では扱いません。

- WSL2自体の導入手順
- Kubernetesの全リソース（数が多いため、入門に必要なものだけを抜粋します）

## 対象読者

- Dockerは触ったことがあるが、Kubernetesは未経験の方
- Windows11+WSL2上のUbuntuが使える状態の方

## k3sとは

Kubernetesをエッジ／IoT向けに軽量化した、Kubernetesディストリビューションです。
1台のマシンでもクラスターを構築できるため、学習用途に向いています。

クラスターとは
複数台のコンピュータやサーバーをネットワークで接続し、1つのシステムのように連携させて処理を行う仕組みのこと。

名前の由来も面白いので、引用しておきます。

> Kubernetesの一般的な短縮形が「K8s」であることから、それをもじって「K3s」と命名されました。8K3sという言葉は、K3sの要件である「K8sから5つの要素を変更（削除）して軽量化（した」ことを表します。

https://www.sysdig.com/jp/learn-cloud-native/what-is-k3s

### k8sの構成について

ハンズオン前に、Kubernetes(以降k8s)について説明します。
k8sの最大の特徴は、**あるべき状態をマニフェストファイル（YAML）で宣言し、その状態を維持し続ける**ことです。
「このコンテナを2つ動かす」と宣言すれば、コンテナが1つでも落ちた場合に自動で作り直してくれます。

複数のコンピューターから成るクラスター構成で、**コントロールプレーン**（マスタノード）と**ワーカーノード**に分かれます。
スケーリングを前提とした仕組みのため、コントロールプレーンが複数のワーカーノードを管理します。

コントロールプレーンは複数のコンポーネントから成り立っていますが、はじめは「コントロールプレーンが指示を出し、ワーカーノードが実際にコンテナを動かす」という理解で十分です。
また、本記事ではコントロールプレーンとワーカーノードが1つのノードに収められるシングルノードで進めます。シングルノードを利用することで通常は複数のコンピューターが必要なk8sを1台のコンピューターで動作確認することが出来ます。1台でもVMを利用すれば可能ではありますが。

参考記事
https://qiita.com/tadashiro_ninomiya/items/6e6fea807b2a16732b5b#kubernetes%E3%81%A8%E3%81%AF

## k3sをインストール

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.35.5+k3s1" sh -
```

これだけで、Ubuntuがそのままk3sクラスターのノードになります。
コマンドの意味は以下の通りです。

- `curl -sfL https://get.k3s.io`：k3s公式のインストールスクリプトを取得する
- `INSTALL_K3S_VERSION="v1.35.5+k3s1"`：インストールするk3sのバージョンを固定する

バージョンを固定しているのは、記事の実行結果と出来る限り一致させるためです。

## 接続情報（kubeconfig）を設定する

インストールが終わったら、クラスターへの接続情報をホームディレクトリにコピーしておきます。

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
```

k3sは接続情報（APIのリクエスト先）を`/etc/rancher/k3s/k3s.yaml`にroot専用（600）で作成します。
そのままだと一般ユーザーが操作するたびにsudoが必要になります。`~/.kube/config`は後述のkubectlコマンドが自動で読みに行く場所なので、ここにユーザー権限でコピーしておけばsudoなしで操作できるようになります。

パーミッションを緩めて共有する方法もありますが、このファイルは**クラスター管理者に相当する権限を持った接続情報**です。
複数人が触るサーバーや本番のクラスターでは、用途ごとに権限を制限した接続情報を配る、といった対応が必要になります。

情報を確認するとserverの値にリクエスト先が記載されていることを確認できます。

```bash
cat /etc/rancher/k3s/k3s.yaml
```

結果（一部抜粋）

```txt
server: https://127.0.0.1:6443
```

kubectlのAPIサーバーにリクエストを送ります。

## kubectlコマンドを打ってみる

kubectlは、k3sクラスターを操作するためのCLIツールです。k3sインストールスクリプトに内包されており、追加のインストールなしで`kubectl`単体で実行できます（`k3s kubectl`と書いても同じ動作です）。イメージとしては`docker ps`のような使い方になります。
まずはノードの一覧を見てみましょう。

```bash
kubectl get nodes
```

以下のような結果が出れば成功です。

```bash
NAME         STATUS   ROLES           AGE   VERSION
wsl-ubuntu   Ready    control-plane   1d   v1.35.5+k3s1
```

通常は2台以上で構成しますが、1台のノードがコントロールプレーンとワーカーノードを兼ねている状態です。

## namespaceを作成する

クラスター内部は、namespaceというリソースで仮想的に区切られています。
区切っておくと、以下のようなメリットがあります。

- **名前の衝突を避けられる**：Podやサービスの名前はnamespace内で一意なので、開発用と検証用で同じ名前のリソースを共存させられる
- **まとめて片付けられる**：namespaceごと削除すれば、中のリソースも一緒に消えるため、掃除が楽になる
- **権限やリソース量を分けられる**：チームや環境ごとに操作できる範囲を絞ったり、使えるCPU・メモリの上限を設定したりできる

今回のように1人で触る学習環境でも、ハンズオンで作ったものをまとめて消せるという使い方ができます。
ハンズオン用のnamespaceを作り、以降のkubectlコマンドがそこに向くよう切り替えておきます。

```bash
kubectl create namespace k3s-tutorial
kubectl config set-context --current --namespace=k3s-tutorial
```

## Podを作成する

KubernetesではPodが最小単位になります。Dockerではコンテナ単位で作成しますが、Podはコンテナを1つ以上内包したものになります。

![alt text](../images/k3s_manual.md/pod.png)

各PodにIPが割り振られ、内包するコンテナはこのIPを共有します。そのため、同一Pod内のコンテナ間はlocalhostで接続できます。

![alt text](../images/k3s_manual.md/pod-ip.png)

複数のコンテナをまとめて1セットとして扱うイメージですが、大きな特徴は**使い捨てである**ことです。

そのため、Podを削除して再作成するとIPは変わります。また今回の構成では発生しませんが、クラスター内のどのノードにPodが作成されるかはシステム側が制御するため、配置されるノードが変わる場合もあります。

![alt text](../images/k3s_manual.md/pod-rebuild.png)

実際にPodを1つ作成してみましょう。マニフェストファイルを作成し、コマンドで適用する流れになります。

今回利用する`traefik/whoami`は、リクエストを受け取ると自分のホスト名・IP・受け取ったHTTPヘッダーをそのまま返すだけの、動作確認用の軽量イメージです。

レスポンスに応答したPodのホスト名が含まれるため、どのPodが返したのかが分かります。

以下のコマンドをそのまま貼り付ければ、Podを作成するマニフェストが作成されます。

```bash
cat << 'EOF' > whoami-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: whoami-pod
  labels:
    app: whoami
spec:
  containers: # 作成するコンテナの設定
    - name: whoami
      image: traefik/whoami # 利用するイメージ名
      ports:
        - containerPort: 80 # コンテナの公開Port
EOF
```

`labels`は、後述のServiceがPodを探すための目印になります。
以下のコマンドでマニフェストを適用します。

```bash
kubectl apply -f whoami-pod.yaml
kubectl get pods
```

作成状況を確認します。

```bash
kubectl get pods -o wide
```

作成されたPod名とIPが確認できます。
```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ kubectl get pods -o wide
NAME         READY   STATUS    RESTARTS   AGE    IP            NODE         NOMINATED NODE   READINESS GATES
whoami-pod   1/1     Running   0          4m2s   10.42.0.150   wsl-ubuntu   <none>           <none>
```

Podにリクエストを送ってみましょう。IP欄に出ているIPにリクエストします。

```
curl -i http://<IPアドレス>:80
```

レスポンス

```
HTTP/1.1 200 OK
Date: Fri, 04 Sep 2026 08:39:21 GMT
Content-Length: 191
Content-Type: text/plain; charset=utf-8

Hostname: whoami-pod
IP: 127.0.0.1
IP: ::1
IP: 10.42.0.150
IP: fe80::943c:6bff:feac:6d31
RemoteAddr: 10.42.0.1:39480
GET / HTTP/1.1
Host: 10.42.0.150
User-Agent: curl/8.5.0
Accept: */*
```

Podが起動しtraefik/whoamiのイメージでコンテナが正常に動作している事を確認できました。

それではPodを削除します。
```
kubectl delete pod whoami-pod
```

```
pod "whoami-pod" deleted from k3s-tutorial namespace
```

消えていることを確認します。
```
kubectl get pods
```

```
No resources found in k3s-tutorial namespace.
```

## Deploymentを定義する

Podをあるべき状態に保つための仕組みです。
Deployment自体はReplicaSetを作り、そのReplicaSetがPodの数を管理する、という二段構えになっています。

ReplicaSetとは、Podを複製して、指定した数を維持するリソースです。
Deploymentとは、ReplicaSetに加えて、ローリングアップデートやロールバックなどを実現するリソースです。

Podを直接作成した場合、削除されても作り直してくれません。一方DeploymentがあればシステムエラーでPodが落ちても、宣言した数になるよう自動で修正されます。
アクセス負荷が増えた場合に`replicas`の数を増やしてスケーリングしたり、イメージを差し替えてローリングアップデートしたりできるのもDeploymentの役割です。

実際の運用ではPodを直接作ることはほとんどなく、Deployment経由で作成します。

実際に体験してみましょう。先ほどと同じPodを、Deployment経由で作り直します。

```bash
cat << 'EOF' > whoami-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami-deploy
spec:
  replicas: 2 # 維持したいPodの数
  selector:
    matchLabels:
      app: whoami # 管理対象のPodを探す条件
  template: # 作成されるPodの定義
    metadata:
      labels:
        app: whoami # selectorと一致させる必要がある
    spec:
      containers: # コンテナの定義
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
EOF
```

`template`より下は、Podのマニフェストとほぼ同じ内容です。「このPodを2つ維持して」と宣言しています。
適用して、状態を確認します。

```bash
kubectl apply -f whoami-deploy.yaml
kubectl get deployments
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ kubectl get deployments
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
whoami-deploy   2/2     2            2           30s
```

`READY 2/2`は、宣言した2つが揃っていることを表します。Podも見てみましょう。

```bash
kubectl get pods -o wide
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
whoami-deploy-7c9f8b6d54-2xq7m   1/1     Running   0          32s   10.42.0.151   wsl-ubuntu   <none>           <none>
whoami-deploy-7c9f8b6d54-h4rjt   1/1     Running   0          32s   10.42.0.152   wsl-ubuntu   <none>           <none>
```

Pod名がランダムな文字列付きになっているのがポイントです。Deployment（とReplicaSet）が自動で名前を振るため、こちらでPod名を決めることはありません。

### 落としても復活することを確認する

せっかくなので、Deploymentのメリットを体験してみましょう。Podを1つ削除します。

```bash
kubectl delete pod whoami-deploy-7c9f8b6d54-2xq7m
kubectl get pods
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
whoami-deploy-7c9f8b6d54-h4rjt   1/1     Running   0          2m10s
whoami-deploy-7c9f8b6d54-w8kzd   1/1     Running   0          5s
```

消したはずなのに2つあります。名前が違う新しいPod（`w8kzd`）が作られ、AGEも数秒前になっていますね。IPが変わっていることも確認できます。
「2つ動かす」という宣言が守られるよう、システム側が自動で作り直した結果です。これにより、本番環境でPodが落ちた場合でも自動的に復活します。

### 数を増やしてみる

`replicas`を変えればスケーリングできます。コマンドで直接変更する方法もありますが、マニフェストが実態とずれてしまうため、ファイルを書き換えて再適用するのが基本です。

```bash
sed -i 's/replicas: 2/replicas: 3/' whoami-deploy.yaml
kubectl apply -f whoami-deploy.yaml
kubectl get pods
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
whoami-deploy-7c9f8b6d54-h4rjt   1/1     Running   0          4m30s
whoami-deploy-7c9f8b6d54-w8kzd   1/1     Running   0          2m25s
whoami-deploy-7c9f8b6d54-qv6tp   1/1     Running   0          4s
```

足りない1つだけが追加され、既存のPodはそのまま動き続けています。次のServiceで使うため、`replicas`は3のまま進めます。

## Serviceを定義する

Podへの接続を安定して提供する仕組みです。リクエストを適切なPodへルーティングする役割を担います。

先述の通りPodは使い捨てのため、再作成のたびにIPが変わります。IPを直接指定して接続していると、そのたびにIPを調べ直す手間が発生してしまいます。Serviceを挟むことで、接続先を固定できます。

Serviceには4種類のタイプがありますが、入門時は上3つを押さえておけば十分です。

| タイプ | 概要 | 主な用途 |
| --- | --- | --- |
| ClusterIP | クラスター内部だけで使える仮想IPを払い出し、複数Podのエンドポイントを束ねる | クラスター内のPod間通信（既定値） |
| NodePort | 各ノードのポート（30000〜32767）を開放する | ノード経由での動作確認 |
| LoadBalancer | ロードバランサー経由で公開する | 外部公開・負荷分散 |
| ExternalName | 外部のDNS名へ橋渡しする（Podは選択しない） | クラスター外のサービス参照 |

上の3つには親子関係があり、LoadBalancerを作成するとNodePortとClusterIPが、NodePortを作成するとClusterIPが内部的に作られます。

ローカルでの動作確認であればClusterIPやNodePortで事足ります。
なお、k3sには`ServiceLB`というロードバランサーが標準で組み込まれているため、クラウドでなくてもLoadBalancerがそのまま動きます。

また、Serviceは`protocol`で接続方式を指定できます。

| protocol | 補足 |
| --- | --- |
| TCP | 既定値。HTTP/HTTPSもこの上で動く |
| UDP | DNSなど |
| SCTP | 対応環境が限られる |

ここで注意したいのが、ServiceはあくまでTCP/UDPレベル（L4）の振り分けだという点です。
「`/api`はこのPodへ、それ以外はあのPodへ」といったURLベースの振り分け（L7）をしたい場合は、Ingressという別のリソースを利用します（本記事では扱いません）。

### ClusterIPのServiceを作ってみる

先ほどのDeploymentで作ったPodたちに、Service経由で接続してみましょう。

```bash
cat << 'EOF' > whoami-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami-svc
spec:
  type: ClusterIP
  selector:
    app: whoami # このラベルを持つPodへ振り分ける
  ports:
    - protocol: TCP
      port: 80 # Serviceが受け付けるPort
      targetPort: 80 # 転送先となるコンテナのPort
EOF
```

重要なのは`selector`です。Pod名ではなく**ラベル**で対象を探すため、Podが作り直されて名前やIPが変わっても、ラベルさえ付いていれば自動で振り分け先に加わります。
適用して確認します。

```bash
kubectl apply -f whoami-svc.yaml
kubectl get svc
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
whoami-svc   ClusterIP   10.43.180.42    <none>        80/TCP    8s
```

`CLUSTER-IP`に払い出された仮想IPが、固定の接続先になります。実際に振り分け先となっているPodは以下で確認できます。

```bash
kubectl get endpoints whoami-svc
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ kubectl get endpoints whoami-svc
NAME         ENDPOINTS                                      AGE
whoami-svc   10.42.0.151:80,10.42.0.153:80,10.42.0.154:80   25s
```

3つのPodのIPがぶら下がっていますね。

### 振り分けられていることを確認する

このIPに何度かリクエストを送ってみます。whoamiは自分のホスト名（＝Pod名）を返してくれるので、振り分けの様子がそのまま見えます。

```bash
for i in $(seq 1 6); do curl -s http://10.43.180.42 | grep Hostname; done
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ for i in $(seq 1 6); do curl -s http://10.43.180.42 | grep Hostname; done
Hostname: whoami-deploy-7c9f8b6d54-h4rjt
Hostname: whoami-deploy-7c9f8b6d54-qv6tp
Hostname: whoami-deploy-7c9f8b6d54-w8kzd
Hostname: whoami-deploy-7c9f8b6d54-h4rjt
Hostname: whoami-deploy-7c9f8b6d54-qv6tp
Hostname: whoami-deploy-7c9f8b6d54-w8kzd
```

同じIPを叩いているのに、応答するPodが入れ替わっています。正直、ここが一番「おお」となるところだと思います。

### Podを作り直しても接続先が変わらないことを確認する

Podを1つ消してから、もう一度同じIPにリクエストしてみます。

```bash
kubectl delete pod whoami-deploy-7c9f8b6d54-h4rjt
curl -s http://10.43.180.42 | grep Hostname
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ curl -s http://10.43.180.42 | grep Hostname
Hostname: whoami-deploy-7c9f8b6d54-n5jzc
```

Podは新しいものに入れ替わり、IPも変わっているはずですが、こちらが叩くIPは`10.43.180.42`のまま変わりません。Podを直接指定していた頃に発生していた「作り直すたびにIPを調べ直す」手間が、これで解消されます。

なお、ClusterIPはクラスター内部専用のIPです。ここまでのcurlがすべてUbuntu上から実行できていたのはそのためで、Windows側のブラウザからは届きません。最後に、外部から接続できるようにしてゴールを目指します。

## 外部から接続する

ClusterIPをNodePortに変更します。NodePortは、クラスター内の各ノードのPort（30000〜32767）を開放し、そこへ来た通信をServiceへ流す仕組みです。
先ほどのマニフェストを、2行だけ書き換えます。

```bash
cat << 'EOF' > whoami-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami-svc
spec:
  type: NodePort # ClusterIPから変更
  selector:
    app: whoami
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080 # ノード側で開放するPort
EOF
```

`nodePort`は省略すると範囲内から自動で割り当てられますが、今回は手順を固定したいので明示しています。
適用して確認します。

```bash
kubectl apply -f whoami-svc.yaml
kubectl get svc
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ kubectl get svc
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
whoami-svc   NodePort   10.43.180.42    <none>        80:30080/TCP   12m
```

`PORT(S)`が`80:30080/TCP`になりました。ClusterIPの`10.43.180.42`はそのまま残っている点にも注目してください。先述の親子関係の通り、NodePortの内部にはClusterIPが作られています。

まずはUbuntu上から、ノードのPort経由で叩いてみます。

```bash
curl -s http://localhost:30080 | grep Hostname
```

```
ubuntu@wsl-ubuntu:~/k3s-tutorial$ curl -s http://localhost:30080 | grep Hostname
Hostname: whoami-deploy-7c9f8b6d54-qv6tp
```

そのままWindows側のブラウザで`http://localhost:30080`を開いてみてください。WSL2はLinux側で待ち受けているPortをWindowsのlocalhostへ転送してくれるため、追加設定なしでwhoamiのレスポンスが表示されるはずです。

```
Hostname: whoami-deploy-7c9f8b6d54-qv6tp
IP: 127.0.0.1
IP: 10.42.0.153
RemoteAddr: 10.42.0.1:52134
GET / HTTP/1.1
Host: localhost:30080
User-Agent: Mozilla/5.0 ...
```

「簡易アプリにHTTP接続をしてレスポンスが返る」という当初のゴールに到達しました。
リロードするたびに`Hostname`が入れ替わるので、ブラウザからでも振り分けの様子が確認できます。

## 後片付け

namespaceごと削除すれば、この記事で作ったリソースはまとめて消えます。namespaceを切っておいたご利益がここで効いてきます。

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace k3s-tutorial
```

## おわりに

todo
