---
title: "FTPサーバを立てて、Pythonで接続する方法"
emoji: "🦾"
type: "tech"
topics: ["FTP", "Docker", "Linux", "Python"]
published: true
published_at: 2025-12-01 06:00
publication_name: "secondselection"
---

## はじめに

FTPサーバーをコンテナで起動し、Pythonで接続する方法を習得しました。
匿名ユーザーでの利用を想定し、環境構築をする中でいくつか躓きポイントがあったので
共有します。本記事が皆さんの参考になりますと幸いです。

## 想定読者🧑‍🦲

- FTPサーバーを簡単に立てて、Pythonで利用したい方
- FTPを匿名ユーザで利用する方法が知りたい方

## FTPとは

「File Transfer Protocol」の名前通り、ファイルを転送するプロトコルです。
あるコンピュータから別のコンピュータへ**ファイルを送る・受け取る**際に利用します。
FTPクライアントはサーバーへ要求をし、FTPサーバーは要求に対して答えます。
例としてクライアントがAファイルの内容を知りたいと要求したら、サーバーは内容を答えます。

![alt text](/images/ftp/ftp_flow.png)

このクライアントとサーバーの2点を押さえておけば、理解が進みます。
今回のサンプルでは通信が暗号化されないため、ローカル環境での実行が前提ということを
ご留意ください。暗号化が必要な場合はFTPS/SFTPなどの利用を検討してください。

FTPの概要については下記記事が理解しやすかったです。

https://wa3.i-3-i.info/word1137.html

### システム構成

今回は図の環境で学習をしました。

![alt text](/images/ftp/ftp_system.png)

### FTPサーバーの立て方

https://dev.classmethod.jp/articles/docker-ftp-ssm-python-practice/

記事を参考にdocker-composeファイルを作成し、コンテナを起動します。
今回使用したFTPサーバーのイメージでは匿名ログインを許可するか、しないかのオプションが
ありました。

下記に匿名モードのサンプルコードを記載します。

```docker-compose
ftpd_server:
    image: stilliard/pure-ftpd
    ports:
      - "21:21"
      - "30000-30009:30000-30009"
    volumes: 
      - "./data:/home/username/"
      - "./passwd:/etc/pure-ftpd/passwd"
    environment:
      PUBLICHOST: "localhost"
      FTP_USER_NAME: testuser
      FTP_USER_PASS: test123
      FTP_USER_HOME: /home/testuser
      ADDED_FLAGS: -e # 匿名ユーザーを許可するオプション
```

コンテナ起動後、ftpユーザーを追加し、ホームフォルダを作成します。この手順が抜けているとサーバーにログインが出来ません。

```bash
useradd -d /var/ftp -s /sbin/nologin ftp
mkdir /var/ftp
```

https://github.com/stilliard/docker-pure-ftpd/issues/43

### FTPサーバーへの接続方法

FTPサーバーへ接続する場合、以下情報が必要です。

- 接続先のホスト名（IPアドレス）
- ポート番号
- ログインユーザ名
- パスワード

これらの情報を使ってクライアントからアクセスします。
FTPクライアントはいくつか選択肢がありますが、本記事ではPython標準モジュールのftplibを
利用します。

同一ネットワークならlocalhostで接続可能です。

接続確認用のコードは以下です。

```python
import ftplib

ftp = ftplib.FTP()
ftp.connect('localhost', port=21, timeout=60)
ftp.login()
# 一行でログインまで済ませたい場合
# ftp = ftplib.FTP('localhost','testuser','test123')
print("サーバーメッセージ:", ftp.getwelcome())
ftp.quit()
```

レスポンスが来たら接続成功です。

```text
# 匿名ログインの場合
サーバーメッセージ: 220---------- Welcome to Pure-FTPd [privsep] [TLS] ----------
220-You are user number 1 of 5 allowed.
220-Local time is now 06:16. Server port: 21.
220-Only anonymous FTP is allowed here
220-IPv6 connections are also welcome on this server.
220 You will be disconnected after 15 minutes of inactivity.

# ユーザーログインの場合
サーバーメッセージ: 220---------- Welcome to Pure-FTPd [privsep] [TLS] ----------
220-You are user number 1 of 5 allowed.
220-Local time is now 06:22. Server port: 21.
220-This is a private system - No anonymous login # 差分
220-IPv6 connections are also welcome on this server.
220 You will be disconnected after 15 minutes of inactivity.
```

ftpユーザー、フォルダがない場合は次のようなエラーが発生します。
ユーザー追加や、フォルダ構成を見直してください。
`ftplib.error_temp: 421 Unable to set up secure anonymous FTP`

### ファイルのアップロード、ダウンロード、削除

サンプルコードを記載します。
ファイルの操作権限エラーが発生する場合は、サーバー側の権限を見直してみてください。

```python
# upload
with open('tsxt.txt', "rb") as f:
    print(ftp.storbinary('STOR test.txt', f))

# download
with open('test.txt', 'wb') as f:
    ftp.retrbinary('RETR text.txt', f.write)

# delete
ftp.delete('test.txt')
```

その他コードは下記ソースをご参照ください。

https://zenn.dev/furimura/articles/a2fb4e91522f2b#python-ftplib%E3%81%AEftp%E6%93%8D%E4%BD%9C
https://docs.python.org/ja/3/library/ftplib.html

## おわりに

今回はFTPについて学習しました。FTPを意識せずにクライアントソフト（FileZillaなど）で利用していたので良い機会になりました。
サーバーの設定で苦労しましたが、ネットワークやDocker、Linuxについての知識が合わせて
身に付きました。
ネット上に匿名ユーザー環境についての情報が少なかったので参考になれば幸いです。
