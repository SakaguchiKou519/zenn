---
title: "🚀 OpenAI Codexでリファクタリングしてみた"
emoji: "🦾"
type: "tech"
topics: ["OpenAI", "#zennfes2025ai"]
published: true
published_at: 2025-09-15 05:00
publication_name: "secondselection"
---

## はじめに
話題になっていたOpenAI Codex CLIを試しにリファクタリング利用してみました。
私自身claudecodeやgemini cliをチュートリアル程度に利用したことがありますが、しっかりと触れたことがなかったです。
理由としてはコマンドベースの操作に抵抗がありました。
ChatGPTのサブスクリプションで利用できることが、自分にとっては一番大きいなと感じています。
良い機会でしたので、どんなことが出来るのか、現状どのレベルなのかを試してみました。

## OpenAI Codexの概要
OpenAIが開発したコーディングエージェントです。似たようなサービスとしてClaudeCodeやGeminiCLIがあります。
特徴としてはコマンドベースで開発が出来ます。
https://developers.openai.com/codex/cli/?utm_source=chatgpt.com

## 利用の流れ
利用方法は簡単でパッケージマネージャを使える環境でコマンドを実行するだけです。

```
npm install -g @openai/codex
```

パッケージ追加後、`codex`とターミナルに入力することで利用が開始できます。

![利用方法選択](/images/login.png)

1. 課金しているアカウントでの利用
2. API利用

1を選択するとサインイン画面に遷移し、認証完了すると以下画面が表示されます。

![利用規約画面](/images/signin.png)

その後、操作モードが聞かれます。勝手にcommitなどをされると困るので、2番を選択しました。こちらは起動時に毎回聞かれるようです。
- **Yes, allow Codex to work in this folder without asking for approval**
→ はい、Codexがこのフォルダ内で作業するときに承認を求めないようにします。
    
- **No, ask me to approve edits and commands**
→ いいえ、Codexが編集やコマンドを実行する際には毎回承認を求めてください。

![操作モード](/images/mode.png)

これにて利用の準備は完了です。

## 基本的な利用方法
初回実行時には下記のような画面が表示されます。

![チュートリアル](/images/tutorial.png)

- **/init**
    `AGENTS.md` ファイルを作成し、Codexへの指示を書き込む
    
- **/status**
    現在のセッション設定やトークン使用状況を表示する
    
- **/approvals**
    Codexが承認なしでできる操作を選ぶ

- **/model**
    使用するモデルと推論のレベルを選ぶ

どれも代表的なコマンドになりますが、最初は`/init`を実行することをお勧めします。
`AGENTS.md`とは？
エージェント向けのREADMEです。ビルドの手順であったり、人間には不要なルール（例:日本語で出力すること）などを記載するファイルになっています。
特徴としてはエージェント間の互換性があることです。今回はOpenAIのエージェントですが、Geminiを使う際にも有効だということです。

参考
https://agents.md/

/statusでは画像のようなセッション設定やトークンの使用状況が確認できます。

![status実行結果](/images/codex.png)

推論のキャンセル
間違えて送信した際にESCを押すことで中断することが出来ます。また、Escを2回押してEnterを押すことで以前のプロンプトの編集も可能です。

![推論キャンセル](/images/cansel.png)

## 便利な機能
/compact
これまでの会話履歴のトークンを要約して圧縮してくれます。これまでの履歴を継続したいが、トークンの残りが少なくなってきたときなどに使えます。

/new
会話履歴をリセットして新しい会話を作成することが出来ます。タスクが完了したなど一区切りついた段階で使うことをお勧めします。
※コンソール上のテキストは残ったままです。

@
ファイルをメンションし、コンテキストとしてエージェントに与えることが出来ます。絶対パスを入力しなくてもファイル名を入力することであいまい検索がされるので便利です。

## リファクタリング

コードや画面を公開できないのですが、APIを記載したPythonファイルとvueのファイル、計2ファイルのリファクタリングを行いました。
この時、先ほど紹介した`@`コマンドを利用して@ファイルA @ファイルBのリファクタリングをしてと指示しました。
注意点として@コマンドは前に半角スペースを1つ開けておく必要があります。
リファクタリングの際のプロンプトですが、どのような観点で行うのかを記載する必要があると考えます。現在は試行錯誤中です。

コードを記載することが出来ないのでサンプルコードで様子をお見せします。

```
# 数字のリストを作って合計を求める冗長なコード

# 空のリストを用意
numbers = []

# 1から5までをリストに追加
numbers.append(1)
numbers.append(2)
numbers.append(3)
numbers.append(4)
numbers.append(5)

# 合計を求める変数を初期化
total = 0

# 1つずつ取り出して加算
for n in numbers:
    if isinstance(n, int):
        if n > 0:
            total = total + n
        else:
            total = total + 0
    else:
        total = total + 0

# 結果を文字列に変換して結合
result = "合計は " + str(total) + " です"

# 出力
print(result)
```

画像のようにGit管理されているファイルは差分を表示しながら変更の確認をしてくれます。不必要な変更が心配な人はこのタイミングで確認することをお勧めします。

![alt text](/images/diff1.png)

![alt text](/images/diff2.png)

変更を承認すると変更概要も出力してくれます。

![alt text](/images/overview.png)

リファクタリングのコードがこちら。
```
"""
正の整数のみ合計して結果を出力するシンプルなスクリプト。

振る舞いは元コード（int かつ正の値のみ加算）を維持。
"""

from typing import Any, Iterable


def sum_positive_ints(numbers: Iterable[Any]) -> int:
    """int 型かつ正の値のみを合計して返す。"""
    return sum(n for n in numbers if isinstance(n, int) and n > 0)


def format_total(total: int) -> str:
    """日本語の結果文を生成する。"""
    return f"合計は {total} です"


def main() -> None:
    numbers = [1, 2, 3, 4, 5]
    total = sum_positive_ints(numbers)
    print(format_total(total))


if __name__ == "__main__":
    main()

```

コードゴルフさせてみた
コードゴルフはソースコードの文字数の少なさを競います。

```
print("合計は %d です"%sum(range(6)))
```

コーディング規約などがある場合などは、それらを記載したファイルをプロジェクトに配置し、それらにそってリファクタリングさせるのも良いと考えます。
```codingstandards.md
- インラインコメントは使用しない
- 関数・変数名は snake_caseを使用すること
```

Before
インラインコメントとキャメルケースを追加しました。
指示：sum.pyをコーディング規約codingstandards.mdに則って、処理フローは保ったままリファクタリングしてください。

```
numbers = []  # 空のリストを用意

# 1から5までをリストに追加
numbers.append(1)
numbers.append(2)
numbers.append(3)
numbers.append(4)
numbers.append(5)


TotalNumbers = 0  # 合計を求める変数を初期化

# 1つずつ取り出して加算
for n in numbers:
    if isinstance(n, int):
        if n > 0:
            TotalNumbers = TotalNumbers + n
        else:
            TotalNumbers = TotalNumbers + 0
    else:
        TotalNumbers = TotalNumbers + 0


totalResult = "合計は " + str(TotalNumbers) + " です"  # 結果を文字列に変換して結合


print(totalResult)  # 出力
```

after
インラインコメントは削除され、キャメルケースも改善されていますね。
今回はスネークケースが不要とはいえ、コメントは単一行などに移行してほしかったです。そういったことも規則ファイルに書く必要がありますね。
```
numbers = []
numbers.append(1)
numbers.append(2)
numbers.append(3)
numbers.append(4)
numbers.append(5)

total = 0
for n in numbers:
    if isinstance(n, int):
        if n > 0:
            total = total + n
        else:
            total = total + 0
    else:
        total = total + 0

result = "合計は " + str(total) + " です"
print(result)
```

今回はサンプルコードで実装しましたが、業務でも十分活用できるレベルでした。また、単純にリファクタリングしてと指示すると、定数化であったり、関数化する傾向があると感じました。リファクタリングでは元のコードをごっそりと変えることもあるので、注意が必要です。

## おまけ
ネット接続が可能。

## 今後したいこと
- MCPとの連携
- 作業終了時の通知（承認時にも）
- config.tomlの有効化

## おわりに
今回Codexを活用してみた純粋な感想はなぜ今まで使っていなかったんだろうでした。コマンドベースですが、自然言語で開発が出来るため、難しいことがほとんどなかったです。開発でWebアプリを使っていたことを反省中です。WebのChatGPTを利用する場合と比べ、情報を与えるのがかなり楽で、精度も高く感じました。業務利用となると活用できる場面は限られますが、使える場面で積極的に使っていきます。また、アプリと行き来する必要がないので開発の集中がしやすいと感じました。
一日利用してみてUIの改善であったり、リファクタリングなどを効率的に行うことが出来ました。中でもGit管理しておくことで差分が出せるので、これはマストです。
最終的なアウトプットの責任は自分にあるので、差分を見て判断がしっかりとできるよう、今後もコーディング力を磨いていきます。