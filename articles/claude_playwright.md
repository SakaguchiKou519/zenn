---
title: "Claude Code × Playwright MCP で NotebookLM を自動更新する"
emoji: "🎭"
type: "tech"
topics: ["claude", "playwright", "mcp", "devcontainer", "notebooklm"]
published: false
---

## 課題

**NotebookLMのソースに更新があった際に同期ボタンをクリックする必要がある。**
上記課題のため、Notebookの数が増えるとメンテナンス負荷が増えます。  
例えば、社内のマニュアルを読み込ませたチャットボットを作成している場合などに課題が出てくるでしょう。

![alt text](/images/claude_playwright/sync.png)

## やりたいこと

- NotebookLMにアクセスし、ソースの更新を自動で行ってほしい。

フロー図を作成予定。

## 解決手段

Claude CodeとPlaywright MCPを組み合わせて、自動でソース更新をする。
最終的には自動更新スクリプトを実行することで完結します。

## Playwright とは

Microsoftが開発したオープンソースのブラウザ自動化フレームワークです。

| 項目 | 内容 |
| --- | --- |
| 対応ブラウザ | Chromium（Chrome/Edge）、Firefox、WebKit（Safari） |
| 対応言語 | JavaScript/TypeScript、Python、Java、.NET |
| 主な用途 | E2E テスト、Web スクレイピング、業務自動化 |

**Auto-wait 機能**が最大の特徴で、操作対象の要素が準備完了になるまで自動的に待機します。
これにより、タイミング問題による操作の不安定さを大幅に削減できます。

## Playwright MCP とは

`@playwright/mcp`として提供されるMicrosoft公式のMCPサーバーです。
Claude CodeなどのAIエージェントがPlaywrightのブラウザ操作機能を直接呼び出せるようにします。

https://github.com/microsoft/playwright-mcp

## システムイメージ

ClaudeがMCPサーバーを介してクリック等のPlaywright機能を利用するイメージです。
MCPサーバーとして提供されていることで、AIはPlaywrightのAPIを直接書くことなく、ツール名とパラメータを指定するだけでブラウザ操作ができます。

![alt text](/images/claude_playwright/playwright_image.png)

## 環境構成

ブラウザ(Chrome)にて動作確認しながら進めたかったため、**コンテナ内に仮想デスクトップ（noVNC）を立てて、ブラウザから画面を確認する**方式を採用します。

```md
WSL
└── devcontainer
    ├── Playwright MCP サーバー（npx）
    ├── 仮想デスクトップ（Xvfb + noVNC）
    └── Claude Code
         ↓ ブラウザ操作
Windows ブラウザ → localhost:6080 で画面確認
```

### 構成のメリット

- Claudeと対話形式でブラウザ操作をさせる、操作状態も確認できる
- Windows側にNode.jsなどのインストールが不要
- コンテナ内で完結するため環境が汚れない
- `localhost:6080` をブラウザで開くだけで画面確認できる

## セットアップ手順

### Step 1: devcontainer.json を設定

仮想デスクトップにて確認をするため`desktop-lite`featureを追加し、ポート6080を転送します。

```json
{
  "name": "Node.js with AI Tools",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:1-20-bullseye",
  "features": {
    "ghcr.io/devcontainers/features/common-utils:2": {
      "installZsh": "true",
      "configureZshAsDefaultShell": "true",
      "installCurl": "true",
      "upgradePackages": "false"
    },
    "ghcr.io/devcontainers/features/desktop-lite:1": {}
  },
  "forwardPorts": [6080],
  "postCreateCommand": "curl -fsSL https://claude.ai/install.sh | bash && npx playwright install --with-deps chromium && claude mcp add playwright npx @playwright/mcp@latest",
  "customizations": {
    "vscode": {
      "settings": {},
      "extensions": [
        "dbaeumer.vscode-eslint"
      ]
    }
  },
  "remoteUser": "node"
}
```

### Step 2: 動作確認

devcontainer起動後、Claude Codeで確認します。

```bash
/mcp
```

`playwright` が `connected` で表示されれば設定完了です。

![alt text](/images/claude_playwright/mcp.png)

### Step 3: ブラウザで画面を開く

ブラウザで以下にアクセスします。

`http://localhost:6080`

仮想デスクトップが表示されます。Claude CodeからPlaywrightを操作すると、この画面でブラウザの動きを確認できます。接続をクリックしてください。

![alt text](/images/claude_playwright/novnc.png)

## ブラウザを操作させてみる

ここまでのセットが完了すると、あとはClaude Codeに自然言語で指示するだけでブラウザを操作できます。

`google.comを開いて`

![alt text](/images/claude_playwright/test.png)

## NotebookLMの自動更新

ドキュメントの自動同期を行います。

後ほどスクリプト作成用の実行手順を教えるためにスクショを取らせます。

`各操作をする度にスクショをして`

`NotebookLMを開いて`

サインインします。こちらは手動入力します。

![alt text](/images/claude_playwright/signin.png)

サインインが完了するとNotebookLMにログインできます。
サインイン後、ログインステータスを保存することで次回からのログインが省けますが、セキュリティにはご注意ください。

![alt text](/images/claude_playwright/notebook_top.png)

`「任意のノートブック名」のブックを開いて`

自動更新させたいブックを開かせます。

![alt text](/images/claude_playwright/notebook_test.png)

`ソースをクリックして`

自動更新させたいソースを開かせます。

![alt text](/images/claude_playwright/notebok_open.png)

`同期するをクリックして。`

![alt text](/images/claude_playwright/before.png)
![alt text](/images/claude_playwright/after.png)

`ここまでの作業をPlaywrightを用いて自動化して`

すると自動更新のスクリプトが作成されますので、スクリプトを定期実行することで更新の手間が省けます。

## まとめ

Claude CodeとPlaywright MCPを利用することでNotebookLMを自動更新できることを確認しました。
この構成でフロント操作の自動化が出来ないかを検証して、上手くいけばスクリプトを出力させる使い方が良いかと考えます。
検証する中で複数のnotebookを更新させましたが、10分程度エラーなしに動作していて、エージェントとしての能力が実用レベルだと感じました。
