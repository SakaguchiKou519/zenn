---
title: "FTPサーバを立ててPythonで接続する環境構築してみた（仮タイトル）"
emoji: "🦾"
type: "tech"
topics: ["FTP", "Docker", "Linux", "DockerCompose", "Python"]
published: true
published_at: 2025-12-01 06:00
publication_name: "secondselection"
---

## はじめに

FTPサーバーをコンテナで起動し、Pythonから接続する方法を習得しました。
ネットワークやディレクトリ作成などいくつか躓きポイントがありました。
今回は匿名ユーザーでの利用を想定しているので、参考になれば幸いです。

## FTPとは

File Transfer Protocolの名前通りファイルを転送するプロトコルです。
FTPクライアントとFTPサーバーの2つを押さえておけば比較的簡単です。
通信が暗号化されないため、利用方法によってはリスクがあります。
その場合はFTPS/SFTPを利用してください。

FTPの概要については下記記事が理解しやすかったです。

https://wa3.i-3-i.info/word1137.html

### システム構成

今回は図の環境で学習をしました。

![alt text](/images/ftp/ftp_system.png)

### FTPサーバーの立て方

https://dev.classmethod.jp/articles/docker-ftp-ssm-python-practice/

記事を参考にコンテナを起動します。
FTPサーバーにはユーザー専用モードと匿名モードがあるようです。
匿名ログインを許可するオプションが分からずに苦労した。
匿名ユーザーのディレクトリも事前に作っておく必要がある。

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
      ADDED_FLAGS: -e # 匿名ユーザーモードで起動するオプション
```

コンテナ起動後、ftpユーザーを追加し、ホームフォルダを作成します。この手順が抜けているとサーバーにログインが出来ません。

```bash
useradd -d /var/ftp -s /sbin/nologin ftp
mkdir /var/ftp
```

https://github.com/stilliard/docker-pure-ftpd/issues/43

### FTPサーバーへの接続方法

FTPクライアントはいくつか選択肢がありますが、Pythonの標準モジュールftplibを利用します。

同一ネットワークならlocalhostで可能です。

接続確認用のコードは以下です。

```python
import ftplib

ftp = ftplib.FTP()
ftp.connect('172.23.0.2', port=21, timeout=60)
ftp.login()
print("サーバーメッセージ:", ftp.getwelcome())
ftp.quit()
```

サーバーからレスポンスが来たら接続成功です。
サーバーの作成等は下記記事をご参照ください。

https://zenn.dev/furimura/articles/a2fb4e91522f2b#python-ftplib%E3%81%AEftp%E6%93%8D%E4%BD%9C
https://docs.python.org/ja/3/library/ftplib.html

## おわりに

今回は気づかぬうちに利用しているFTPについて学習しました。環境構築を進めるうえで、ネットワークやDocker、Linuxについての知識も身に付きました。

## 参考