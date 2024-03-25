---
title: "【10分で簡単】VitePress を Azure Static Web Apps でホストしてみる"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, vue, vite, staticwebapps, githubactions]
published: false
publication_name: "microsoft"
---

# はじめに

VitePress は Vue.js の静的サイトジェネレーターで、Vue 2系時代の VuePress の後継としてビルド速度の向上や TypeScript のサポートなどが強化されています。

今回は、VitePress で作成したドキュメントを Azure Static Web Apps でホストする手順をご紹介します。

また、GitHub Actions を使って、main ブランチへのプッシュ時に自動的にビルド・デプロイする方法も合わせてご紹介します。

https://vitepress.dev/

# VitePress のセットアップ

ディレクトリを作成して、VitePress をインストールします。

```bash
mkdir my-vitepress && cd my-vitepress
npm i -D vitepress
```

VitePress のセットアップウィザードを使用することで、簡単に必要な設定ファイルやディレクトリ構造を作成できます。

```bash
npx vitepress init
```

初期化すると、以下のようなプロジェクト構造が作成されます。

```
.
├─ docs
│  ├─ .vitepress
│  │  └─ config.js # 設定ファイル
│  ├─ api-examples.md # VitePress API の使用例
│  ├─ markdown-examples.md # Markdown 記法の使用例
│  └─ index.md # トップページ
└─ package.json
```

また、`package.json` には以下のようなスクリプトを追加することで、簡単に開発サーバーを起動したりビルドできるようになります。

```json
{
  ...
  "scripts": {
    "dev": "vitepress dev docs",
    "build": "vitepress build docs",
    "preview": "vitepress preview docs"
  },
  ...
}
```

開発サーバーを起動して、ドキュメントを確認します。

```bash
npm run dev
```

# Azure Static Web Apps へのデプロイ

Azure Static Web Apps は、SPA や静的ウェブサイトをホストするためのサービスです。
GitHub Actions との連携が簡単で、GitHub にコードをプッシュするだけでビルド・デプロイが自動的に行うことができるため、静的サイトのホスティングに最適です。

https://learn.microsoft.com/ja-jp/azure/static-web-apps/overview

1. Azure Portal にログインし、検索バーから `静的 web アプリ` を検索します。
2. `[+作成]` をクリックして、新しい Static Web Apps を作成します。
3. サブスクリプション、リソースグループ、アプリ名、リージョン、プランの種類を選択します。

::: message
Azure Static Web Apps は、`Free`, `Standard` の2種類から選択することが可能です。
趣味で使用する分は `Free` プランで十分使用できますが、カスタム認証や SLA が必要な場合は `Standard` プランを検討してください。

![Azure Static Web Apps](/images/20240325_vitepress_static_web_apps/azure-static-web-apps-plan.png)
:::

4. `デプロイの種類` で `GitHub` を選択し、デプロイ対象の GitHub のリポジトリ、ブランチを選択します。

5. `ビルドのプリセット` で `vuepress` を選択します。また、ここでは `Gatsby`, `Hugo`, `Jekyll` などの静的サイトジェネレーターに加えて、 `Vue.js` や `Next.js` などのフレームワークも選択できます。

![ビルドのプリセット](/images/20240325_vitepress_static_web_apps/build-preset.png)

6. `アプリの場所`, `出力先` を以下のように設定します。

アプリの場所: `/`
出力先: `docs/.vitepress/dist`

![アプリの場所](/images/20240325_vitepress_static_web_apps/app-location.png)

::: message
静的ファイルのビルドコマンドをカスタマイズする場合は、GitHub Actions のワークフローファイルに  `app_build_command` プロパティを追加してください。

```diff yml
jobs:
  build_and_deploy_job:
  ... // 省略
      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          
          app_location: "/" # App source code path
          api_location: "" # Api source code path - optional
          output_location: "docs/.vitepress/dist  " # Built app content directory - optional
+         app_build_command: "カスタムビルドコマンド"
```
:::

7. プレビュー ワークフローファイルを選択し、GitHub Actions のワークフローが問題ないことを確認します。
8. `[確認および作成]` をクリックして、Static Web Apps インスタンスを作成します。

これで、GitHub にコードをプッシュすると、GitHub Actions がビルド・デプロイを自動的に行い、Azure Static Web Apps に VitePress のドキュメントがホストされます。

# まとめ

今回は、VitePress で作成したドキュメントを Azure Static Web Apps でホストする手順をご紹介しました。
Azure Static Web Apps は、GitHub Actions との連携が簡単で、静的サイトのホスティングに最適なサービスです。
是非 Azure Static Web Apps を使って、独自のブログやドキュメントを公開してみてください。

# 参考文献

https://vitepress.dev/

https://learn.microsoft.com/ja-jp/azure/static-web-apps/overview
