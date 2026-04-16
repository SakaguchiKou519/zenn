---
title: "Claude Code + Playwright MCPで始めるレスポンシブ確認の自動化"
emoji: "🤡"
type: "tech"
topics: ["claude", "playwright", "mcp", "devcontainer", "レスポンシブ"]
published: true
published_at: 2026-04-20 06:00
publication_name: "secondselection"
---

## はじめに

Playwrightを用いて、**複数ブラウザでのレスポンシブ対応の確認を自動化**したいと考えました。
複数ブラウザ・複数サイズでのレイアウト確認を手動で行うのは手間がかかります。今回はそういった課題をClaude Code + Playwright MCPを利用して解決していきます。
最終成果物は複数ブラウザで撮影をするスクリプトになります。

![Claude CodeとPlaywright MCPを組み合わせたシステム構成図](/images/responsible_playwright/system_image.png)

用語や、環境構築方法についてはこちらの記事をご参照ください。

https://zenn.dev/secondselection/articles/claude_playwright

## PlaywrightのRecorder機能を使わないのか？

PlaywrightにはRecorder機能があり、ブラウザで操作するだけで自動的にテストコードが生成されます。
Recorderを使わずにAIを利用する最大の利点としては、こちらの意図を汲み取ってくれることだと言えます。
後述のスクリプトでも紹介しているような繰り返しの作業も、意図を理解していい感じに処理してくれます。
サイトの構造が変わった際も、自然言語で修正できるので、構築、保守ともに便利だと考えます。

## 要件について

ブラウザおよびサイズ要件は以下とします。

### ブラウザ要件

- Chromium（Chrome/Edge）
- Firefox
- WebKit（Safari）

### サイズ要件

- PC:1920px*1080px
- タブレット:768px*1024px
- スマホ:390px*844px

## 環境構成

冒頭でご紹介した記事の構成を踏襲しています。変更点はブラウザの追加と、postCreateCommandのコマンドを別ファイルへ切り出した点です。
設定ファイルを再掲しますが、環境構築の方法は冒頭の記事を参照ください。

```json:devcontainer.json
{
  "name": "responsive",
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
  "postCreateCommand": "bash .devcontainer/postCreateCommand.sh",
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

```bash:postCreateCommand.sh
#!/bin/bash
set -e

curl -fsSL https://claude.ai/install.sh | bash
npx playwright install chromium firefox webkit
claude mcp add playwright npx @playwright/mcp@latest
```

## 手順

今回は弊社のHPにアクセスしてレイアウトを確認していきます。
最初のステップとしてChromiumでサイズ要件にある3サイズをスクショできるようにします。
画像はフルサイズで撮影できますが、長くなってしまいますので、掲載分は一部を切り取っています。

以下Claude Codeに対するプロンプトは🤵マークを付与します。

`🤵https://www.secondselection.com/に1920px*1080pxでアクセス、撮影して。`

![PC(1920×1080px)でのスクリーンショット撮影結果](/images/responsible_playwright/pc.png)

無事にアクセス出来ていることが確認できたので、スマホ、タブレットも続けていきます。

`🤵次にスマホサイズ390px*844pxでアクセス、撮影して。`

![スマホ(390×844px)でのスクリーンショット撮影結果](/images/responsible_playwright/mobile.png)

`🤵次にタブレットサイズ768px*1024pxでアクセス、撮影して。`

![タブレット(768×1024px)でのスクリーンショット撮影結果](/images/responsible_playwright/tablet.png)

`🤵ここまでの作業をPlaywrightを利用したコードに書き起こして`

### 生成されたスクリプト

※汎用的に利用できるよう一部修正しています。

```js:fullscreenshot.js
const { chromium } = require('@playwright/test');
const fs = require('fs');
const path = require('path');

const URL = process.argv[2];
if (!URL) {
  console.error('Usage: node fullscreenshot.js <url>');
  process.exit(1);
}

const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
const hostname = new globalThis.URL(URL).hostname.replace(/\./g, '_');
const archiveDir = path.join('archive', `${timestamp}_${hostname}`);
fs.mkdirSync(archiveDir, { recursive: true });

(async () => {
  const browser = await chromium.launch();

  // デスクトップ (1920x1080)
  const desktopPage = await browser.newPage();
  await desktopPage.setViewportSize({ width: 1920, height: 1080 });
  await desktopPage.goto(URL, { waitUntil: 'networkidle' });
  await desktopPage.screenshot({ path: path.join(archiveDir, 'desktop.png'), fullPage: true });
  await desktopPage.close();

  // スマホ (390x844)
  const mobileContext = await browser.newContext({
    viewport: { width: 390, height: 844 },
    isMobile: true,
    hasTouch: true,
  });
  const mobilePage = await mobileContext.newPage();
  await mobilePage.goto(URL, { waitUntil: 'networkidle' });
  await mobilePage.screenshot({ path: path.join(archiveDir, 'mobile.png'), fullPage: true });
  await mobileContext.close();

  // タブレット (768x1024)
  const tabletPage = await browser.newPage();
  await tabletPage.setViewportSize({ width: 768, height: 1024 });
  await tabletPage.goto(URL, { waitUntil: 'networkidle' });
  await tabletPage.screenshot({ path: path.join(archiveDir, 'tablet.png'), fullPage: true });
  await tabletPage.close();

  await browser.close();

  console.log(`Saved to: ${archiveDir}`);
})();

```

### 結果

`node fullscreenshot.js <url>`のようにURLを入力するだけでchromiumで3サイズの撮影が出来ることを確認しました。

### 注意点

動的なサイトは適宜待機する設定が必要です。
今回はスクリプト実行時に真っ白な画面が撮影されたため、ネットワークリクエストが500ms以上発生しなくなるまで待機するwaitUntil:'networkidle'を各gotoに追加しました。サイトによっては特定の要素の表示を待つ設定が別途必要になる場合があります。
また、スクリプトを実行する際は対象サイトの利用規約を確認してください。

### 複数ブラウザ対応

単一ブラウザで動作確認が出来ましたので、複数ブラウザ対応に進みます。

`🤵Chromiumで3サイズ撮影するスクリプトをFirefox,Webkitを追加して、3ブラウザで撮影するように対応させて`

Chromium、Firefoxは順調にいきましたがSafariで失敗しました。WSL2環境ではSafari(WebKit)の起動に必要なライブラリが多く、環境構築の難易度を考慮して今回はスキップする仕様にしました。

```js:capture.js
const { chromium, firefox, webkit } = require('@playwright/test');
const fs = require('fs');
const path = require('path');

const URL = process.argv[2];
if (!URL) {
  console.error('Usage: node capture.js <url>');
  process.exit(1);
}

const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
const hostname = new globalThis.URL(URL).hostname.replace(/\./g, '_');
const archiveDir = path.join('archive', `${timestamp}_${hostname}`);
fs.mkdirSync(archiveDir, { recursive: true });

const viewports = [
  { name: 'desktop', width: 1920, height: 1080, isMobile: false },
  { name: 'tablet',  width: 768,  height: 1024, isMobile: false },
  { name: 'mobile',  width: 390,  height: 844,  isMobile: true  },
];

const browsers = [
  { name: 'chromium', launcher: chromium },
  { name: 'firefox',  launcher: firefox  },
  { name: 'webkit',   launcher: webkit   },
];

(async () => {
  for (const { name: browserName, launcher } of browsers) {
    let browser;
    try {
      browser = await launcher.launch();
    } catch (e) {
      console.warn(`Skipping ${browserName}: ${e.message.split('\n')[0]}`);
      continue;
    }

    for (const { name: vpName, width, height, isMobile } of viewports) {
      const contextOptions = { viewport: { width, height } };
      if (browserName !== 'firefox') {
        contextOptions.isMobile = isMobile;
        contextOptions.hasTouch = isMobile;
      }
      const context = await browser.newContext(contextOptions);
      const page = await context.newPage();
      await page.goto(URL, { waitUntil: 'networkidle' });
      await page.screenshot({ path: path.join(archiveDir, `${browserName}-${vpName}.png`), fullPage: true });
      await context.close();
      console.log(`Captured: ${browserName}-${vpName}`);
    }

    await browser.close();
  }

  console.log(`Saved to: ${archiveDir}`);
})();

```

## おわりに

最終的にはChromium、FirefoxのブラウザでPC、スマホ、タブレットの計6枚のスクショ撮影を自動化できました。URLを引数に渡すだけで動く汎用スクリプトになっているので、ぜひご活用ください。
ClaudeCode + Playwright MCPの組み合わせはブラウザ操作全般に応用できるので、環境構築から試してみることをお勧めします。
