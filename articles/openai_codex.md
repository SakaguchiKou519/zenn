---
title: "【入門】OpenAI Codex CLIでリファクタリングしてみた"
emoji: "🥋"
type: "tech"
topics: ["vibecoding", "OpenAI", "codexcli", "zennfes2025ai", "agent"]
published: true
published_at: 2025-09-16 06:00
publication_name: "secondselection"
---

## はじめに

巷で話題になっていた**OpenAI Codex CLI**をリファクタリングで利用してみました。
クラウドモデル、ローカルモデルの両方に対応しているので、どちらとも検証しました。
これまで、Claude CodeやGemini CLIをチュートリアル程度に利用したことがありますが、
しっかりと触れたことがなかったです。
コマンド操作に慣れておらず、Web上のChatGPTを使用していたこともあって、コマンドベースの操作に抵抗がありました。
個人的にはChatGPTのサブスクリプションで利用できることが一番大きいなと感じています。
良い機会でしたので、どんなことが出来るのか、現状どのレベルなのかを試しました。

## OpenAI Codexの概要

OpenAIが開発した**コーディングエージェント**です。公開自体は2025年4月にされています。
似たようなサービスとしてClaude CodeやGemini CLIがあります。

:::details `コーディングエージェントとは`
自然言語で指示をすることで、コーディングやテスト、デバッグといった作業を自律的に行ってくれる生成AIのことを指します。

:::

https://developers.openai.com/codex/cli/?utm_source=chatgpt.com

## 利用の流れ

### 開発環境

#### 必須

- ChatGPTアカウント(有料プラン利用orAPI利用)
- Node.js
- npm

#### 任意

- Docker(linux)

利用方法は簡単で、Nodeのパッケージマネージャが使える環境でコマンドを実行するだけです。macの場合Homebrewでのインストールに対応しています。

```bash
npm install -g @openai/codex
```

パッケージ追加後、`codex`とターミナルに入力することで利用が開始できます。

-----

![利用方法選択](/images/login.png)

1. ChatGPTアカウントでのサインイン(ChatGPT Plusなどを契約している方はこちらがお勧めです。)
2. API利用

-----

![利用規約画面](/images/signin.png)

サインインが完了すると利用規約画面が表示されます。

-----

![操作モード](/images/mode.png)

1. **Yes, allow Codex to work in this folder without asking for approval**
→ はい、Codexがこのフォルダ内で作業するときに承認を求めないようにします。
2. **No, ask me to approve edits and commands**
→ いいえ、Codexが編集やコマンドを実行する際には毎回承認を求めてください。

エージェントの操作モードを選択します。
勝手にcommitやsudoなどをされると困るので、2番を選択しました。
これにて利用の準備は完了です。

## CodexでローカルLLMを利用する場合

CodexでローカルLLMを利用する場合は`~/.codex/config.toml`にモデル設定を書き込む必要があります。

:::details config.tomlとは
エージェントの設定ファイルで、MCPとの連携やデスクトップ通知、推論モードの設定などが可能になります。

:::

躓きポイントとして、Docker環境では`cd ~`でユーザーのホームディレクトリに移動してから
設定ファイルを作成する必要がある点です。
クラウドモデルを利用する場合と違うのは、トークンの利用状況が不明なところです。
詳しい設定方法については以下記事を参照ください。

https://www.taneyats.com/entry/use-gpt-oss-in-codex

## 基本的な利用方法

初回実行時には下記のような画面が表示されます。

![チュートリアル](/images/tutorial.png)

| コマンド           | 機能                                       |
| -------------- | ---------------------------------------- |
| **/init**      | `AGENTS.md` ファイルを新規作成し、Codex への初期指示を書き込む |
| **/status**    | 現在のセッション設定やトークン使用状況を表示する                 |
| **/approvals** | Codex が事前承認なしで実行できる操作を選択・設定する            |
| **/model**     | 使用するモデルと推論レベルを選択する                       |

### /init

どれも代表的なコマンドになりますが、最初は`/init`を実行することをお勧めします。

:::details AGENTS.mdとは？

エージェント向けのREADMEです。
ビルドの手順であったり、人間には不要なルール（例:日本語で出力すること）などを
記載するファイルになっています。
特徴としてはエージェント間の互換性があることです。
今回はCodexでの利用ですが、Gemini CLIを使う際にも有効だということです。

:::

https://agents.md/

-----

### /status

![status実行結果](/images/codex.png)

画像のようにセッション設定やトークンの使用状況が確認できます。

## 便利な機能

| コマンド                                | 機能                                                                |
| ----------------------------------- | ----------------------------------------------------------------- |
| `/compact`                          | これまでの会話履歴のトークンを要約・圧縮します。履歴を継続したいがトークン残量が少ないときに有効です。               |
| `/new`                              | 会話履歴をリセットして新しい会話を開始します。タスク完了など一区切りついた段階での利用がお勧めです。※コンソール上にテキストは残ります。   |
| `@`（ファイルメンション機能）                    | `@`でファイルをメンションしてコンテキストとしてエージェントに渡せます。絶対パスは不要で、ファイル名であいまい検索されます。 |
| `Esc` / `Esc`×2 → `Enter`（推論のキャンセル） | 推論を中断できます。誤送信時は`Esc`で即中断、`Esc`を2回押して`Enter`で直前のプロンプトを編集できます。      |

| 項目 | 説明 |
| ----------------------------------- | ----------------------------------------------------------------- |
|Web検索|Codex は Web 検索が可能です。コーディング規約等を検索してきて、情報をもとにコーディング等も可能です。 |
|禁止コマンド設定 |sudoやcommitなどの禁止コマンド設定は、調査時点では情報が見つからなかったです。今後のアップデートでの対応が期待されます。|

## リファクタリング

サンプルコードで様子をお見せします。リファクタリングには記載がない限りクラウドモデル(GPT5)を使用しています。
この時、先ほど紹介した`@`コマンドを利用して`@ファイルAのリファクタリングをして`と指示します。
リファクタリングの際のプロンプトですが、どのような観点で行うのかを記載する必要があると考えます。効果的な方法を試行錯誤中です。

Before

```python
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

After

```python
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

After(gpt-oss-20b)

```python
def compute_total(numbers):
    return sum(n for n in numbers if isinstance(n, int) and n > 0)


def format_result(total):
    return f"合計は {total} です"


def main():
    numbers = [1, 2, 3, 4, 5]
    total = compute_total(numbers)
    print(format_result(total))


if __name__ == "__main__":
    main()

```

いかがでしょうか?
今回は簡単なプログラムでしたので、ここまで細かく関数化する必要はないと思いました。
単純にリファクタリングをお願いしただけなので、当然の結果とも言えます。

下記にリファクタリング時の様子を添付します。

![差分1](/images/diff1.png)

![差分2](/images/diff2.png)

画像のようにGit管理されているファイルは**差分を表示しながら変更の確認**をしてくれます。
不必要な変更が心配な人はこのタイミングで確認することをお勧めします。

![概要](/images/overview.png)

変更を承認すると変更概要も出力してくれます。

-----

コーディング規約などがある場合などは、それらを記載したファイルをプロジェクトに配置し、リファクタリングさせるのも良いと考えます。

```markdown
### サンプル規約
- インラインコメントは使用しない
- 関数・変数名は snake_caseを使用すること
```

それではコーディング規約に沿ってリファクタリングをお願いしていきます。
今回は、コードにインラインコメントとパスカルケース、キャメルケースを追加しました。
規約通りに、インラインコメントの撤廃とスネークケースの使用を出力として期待します。

プロンプト：`sum.pyをコーディング規約に則って、処理フローは保ったままリファクタリングしてください。`

Before

```python
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

After

```python
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

After(gpt-oss-20b)

```python
numbers = [1, 2, 3, 4, 5]
total = 0
for n in numbers:
    if isinstance(n, int) and n > 0:
        total = total + n
result = "合計は " + str(total) + " です"
print(result)
```

インラインコメントは削除され、キャメルケースも改善されていますね。
今回はスネークケースが不要とはいえ、コメントは単一行などに移行してほしかったです。
そういったことも規則ファイルに書く必要がありますね。
リファクタリングのコードを見比べてみると、クラウドモデルのほうが指示に忠実で、コードが読みやすいと感じました。

今回はサンプルコードをお見せしましたが、GPT5を業務で1日利用してみたところ、十分活用できるレベルでした。UIの改善であったり、リファクタリングなどを効率的に行うことが出来ました。
また、単純にリファクタリングしてと指示すると定数化であったり、関数化する傾向があると感じました。
リファクタリングでは元のコードをごっそりと変えることもあるので、注意が必要です。

## 今後したいこと

- MCPとの連携
- 作業終了時の通知（承認時にも）

これらが実現すると、回答を適宜確認する必要がなくなり、より機密な情報を扱うことが可能になります。MCP連携によってエージェントが取る行動の選択肢も増えて、結果として開発の効率が上がると考えられます。

## おわりに

今回Codexを活用してみた純粋な感想はなぜ今まで使っていなかったんだろうでした。
コマンドベースですが自然言語で開発が出来るため、難しいことがほとんどなかったです。
開発でChatGPTのWebアプリだけを使っていたことを反省中で、今後は併用します。
箇条書きで感想を記載します。

- WebのChatGPTと比べ、情報を与えるのが楽で、精度も高く感じました。
- アプリと行き来する必要がないので、開発の集中がしやすいと感じました。
- 実装面では**Git管理**しておくことで差分が出せるので、マストだなと思いました。

業務利用となると活用できる場面は限られますが、使える場面で積極的に使っていきます。
現状だと社内システムをクラウドモデルで作成する、機密な情報はローカルモデルで利用するといった使い分けが出来そうです。
とても便利なAIコーディングですが、最終的な成果物の責任は自分にあるので
差分を見て判断ができるよう、今後もコーディング力を磨いていきます。

## 参考

https://zenn.dev/dely_jp/articles/codex-cli-matome#%E3%81%AF%E3%81%98%E3%82%81%E3%81%AB