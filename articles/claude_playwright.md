---
title: "DevContainerで完結！Claude Code + Playwright MCPを使ったブラウザ操作自動化の構築手順"
emoji: "🎭"
type: "tech"
topics: ["claude", "playwright", "mcp", "devcontainer", "notebooklm"]
published: false
---

## はじめに

Claude Code + Playwright MCPを使うと、自然言語でブラウザを操作し、その操作をそのままRPAスクリプトとして自動生成できます。
イメージとしては、RPAのコードをClaude Codeに作らせるイメージです。

本記事では、この環境をDevContainer内で完結させる構築手順をまとめます。
動作確認として、NotebookLMのソース同期を自動化した例も紹介します。

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

![ClaudeがMCPサーバーを介してPlaywrightを操作するシステムイメージ](/images/claude_playwright/playwright_system_image.png)

## 環境構成

ブラウザ(Chrome)にて動作確認しながら進めたかったため、**コンテナ内に仮想デスクトップ（Xvfb + noVNC）を立てて、ブラウザから画面を確認する**方式を採用します。
※今回はDevContainerを利用しましたが、通常のDockerコンテナでも同じ環境は作れます。

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

VSCode上でdevcontainer起動後、Claude Codeを用いて確認します。

```bash
/mcp
```

`playwright` が `connected` で表示されれば設定完了です。

![Claude CodeでPlaywright MCPがconnected状態で表示されている様子](/images/claude_playwright/mcp.png)

### 活用例: NotebookLMのソース自動同期

ここからは実際にどんな操作を自動化するかを説明します。

NotebookLMはGoogleが提供するAIを活用したナレッジ管理ツールです。
ソース(アップロードした資料)に更新があった際に手動で同期ボタンをクリックする必要があり、Notebookの数が増えるとメンテナンス負荷が増えます。

![NotebookLMのソース同期ボタン](/images/claude_playwright/sync.png)

この同期作業をClaude Codeに指示しながら進め、最後にスクリプトとして書き出させます。

![スクリプトフロー](/images/claude_playwright/script_flow.png)

### Step 3: ブラウザで画面を開く

ブラウザで以下にアクセスします。

`http://localhost:6080`

仮想デスクトップが表示されます。Claude CodeからPlaywrightを操作すると、この画面でブラウザの動きを確認できます。接続をクリックしてください。

![noVNCの仮想デスクトップ画面（localhost:6080）](/images/claude_playwright/novnc.png)

## ブラウザを操作させてみる

ここまでのセットアップが完了すると、あとはClaude Codeに自然言語で指示するだけでブラウザを操作できます。

以降プロンプトは🧑‍🦲マークを付与します。

`🧑‍🦲google.comを開いて`

![Claude CodeがPlaywrightでgoogle.comを開いた様子](/images/claude_playwright/test.png)

## NotebookLMの自動更新

実際にClaude Codeに指示して操作させます。

後ほどスクリプト作成用の実行手順を教えるためにスクショを取らせます。

`🧑‍🦲各操作をする度にスクショをして`

`🧑‍🦲NotebookLMを開いて`

サインインします。こちらは手動入力します。

![NotebookLMのサインイン画面](/images/claude_playwright/signin.png)

サインインが完了するとNotebookLMにログインできます。
サインイン後、ログインステータスを保存することで次回からのログインが省けますが、セキュリティにはご注意ください。

![サインイン後のNotebookLMトップ画面](/images/claude_playwright/notebook_top.png)

自動更新させたいブックを開かせます。

`🧑‍🦲「The Japanese Test Document」のブックを開いて`

![「The Japanese Test Document」のNotebookを開いた画面](/images/claude_playwright/notebook_test.png)

自動更新させたいソースを開かせます。

`🧑‍🦲ソース(テスト)をクリックして`

![ソース詳細を開いた画面](/images/claude_playwright/notebok_open.png)

`🧑‍🦲同期するをクリックして。`

![同期前のソース状態](/images/claude_playwright/before.png)
![同期後のソース状態](/images/claude_playwright/after.png)

`🧑‍🦲ここまでの作業をPlaywrightを用いて自動化して`

すると自動更新のスクリプトが作成されますので、スクリプトを定期実行することで更新の手間が省けます。

## 生成されたスクリプト

```javascript
const { chromium } = require('playwright');

async function syncNotebookLM() {
  console.log('ブラウザを起動中...');

  const browser = await chromium.launch({
    channel: 'chrome',
    headless: false,
    args: ['--no-sandbox'],
  });

  const page = await browser.newPage();

  try {
    // 1. NotebookLM を開く
    console.log('NotebookLM を開いています...');
    await page.goto('https://notebooklm.google.com');

    // ログインが必要な場合は手動で待機
    if (page.url().includes('accounts.google.com')) {
      console.log('⚠️  ブラウザでGoogleアカウントにログインしてください。');
      console.log('ログイン完了後、自動的に続行します...');
      await page.waitForURL('https://notebooklm.google.com/**', { timeout: 120000 });
    }

    console.log('✅ NotebookLM を開きました');

    // 2. "The Japanese Test Document" をクリック
    console.log('"The Japanese Test Document" を開いています...');
    await page.goto('https://notebooklm.google.com/notebook/xxxx');
    await page.waitForLoadState('load');
    console.log('✅ ノートブックを開きました');

    // 3. 左側の「テスト」ソースをクリック
    console.log('「テスト」ソースを選択しています...');
    await page.getByRole('button', { name: 'テスト', exact: true }).waitFor({ timeout: 30000 });
    await page.getByRole('button', { name: 'テスト', exact: true }).click();
    await page.waitForTimeout(1000);
    console.log('✅ ソースを選択しました');

    // 4. 「クリックして Google ドライブと同期」をクリック
    console.log('Google ドライブと同期しています...');
    await page.waitForSelector('text=クリックして Google ドライブと同期', { timeout: 10000 });
    await page.click('text=クリックして Google ドライブと同期');

    // 同期完了を待機
    await page.waitForSelector('text=同期が完了しました', { timeout: 30000 });
    console.log('✅ 同期が完了しました！');

  } catch (err) {
    console.error('❌ エラーが発生しました:', err.message);
    console.log('現在のURL:', page.url());
    await page.screenshot({ path: 'debug_screenshot.png' });
    console.log('スクリーンショットを debug_screenshot.png に保存しました');
    process.exitCode = 1;
  } finally {
    await browser.close();
    console.log('ブラウザを閉じました。');
  }
}

syncNotebookLM();
```

## まとめ

Claude CodeとPlaywright MCPを利用することでNotebookLMを自動更新できることを確認しました。
この構成の活用方法として、**まずClaudeに自然言語で操作させてRPAスクリプトを自動生成し、それを定期実行する**という流れが有効です。Playwrightの知識がなくても、操作を見せるだけでスクリプトを作れる点が最大のメリットです。

![同期後のソース状態](/images/claude_playwright/pdca.png)
