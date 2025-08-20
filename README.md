# astro-notion-blog

Notionで書いたコンテンツをAstroでブログ化しLolipopにデプロイする

## セットアップ

### 必要なもの

- Notion
- ロリポップレンタルサーバの契約

### ステップ

1. [ブログテンプレート](https://otoyo.notion.site/e2c5fa2e8660452988d6137ba57fd974?v=abe305cd8b3d467285e91a2a85f4d8de) を自分のNotionへ複製
2. 複製したページ(データベース)のアイコン、タイトル、説明を任意に変更
3. 複製したページ(データベース)のURL `https://notion.so/your-account/<ここ>?v=xxxx` を `DATABASE_ID` としてメモ
4. [Create an integration](https://developers.notion.com/docs/create-a-notion-integration#step-1-create-an-integration) からコネクトを新規作成し "Internal Integration Token" を `NOTION_API_SECRET` としてメモ
5. 複製したページを再度開き [Share a database with your integration](https://developers.notion.com/docs/create-a-notion-integration#step-2-share-a-database-with-your-integration) の手順でデータベースに上記のコネクト接続
6. [ロリポップのユーザー設定](https://user.lolipop.jp/?mode=account)を開き、ＦＴＰサーバーを`FTP_HOST`としてメモ、ＦＴＰ・WebDAVアカウントを`FTP_USERNAME`としてメモ、ＦＴＰ・WebDAVパスワードを`FTP_PASSWORD`としてメモしておく
7. GitHub の Settings > 左メニューのSecrets and variables > Actions を開く
8. New repository secret で、`DATABASE_ID`とかをNameに、めもした内容をSecretに入力し、以下のとおり合計7つのSecretを作成

- DATABASE_ID
- NOTION_API_SECRET
- FTP_HOST
- FTP_USERNAME
- FTP_PASSWORD
- FTP_REMOTE_ROOT
- PUBLIC_GA_TRACKING_ID（Google AnalyticsのトラッキングID。例：G-XXXXXXXXXX）

※FTP_REMOTE_ROOTは自分の契約したレンタルサーバーのどこにデプロイするかの設定。直下でいいならSecretの内容は半角スラッシュ「/」だけで作成(契約したレンタルサーバのアドレスにアクセスするとブログが見られるようになる)

9. GitHub の Actionsを開き、workflowを作成

```
   name: deploy

on:
  workflow_dispatch:
  schedule:
    - cron:  '00 10 * * *'
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: install packages
        run: yarn install
      - name: build
        run: yarn build
        env:
          NOTION_API_SECRET: ${{ secrets.NOTION_API_SECRET }}
          DATABASE_ID: ${{ secrets.DATABASE_ID }}
      - name: Deploy via FTP
        uses: SamKirkland/FTP-Deploy-Action@4.3.0
        with:
          server: ${{ secrets.FTP_HOST }}
          username: ${{ secrets.FTP_USERNAME }}
          password: ${{ secrets.FTP_PASSWORD }}
          server-dir: ${{ secrets.FTP_REMOTE_ROOT }}
          local-dir: ./dist/
```

※astro-notion-blog では新しい記事や変更を公開したいとき毎回デプロイが必要。

## :hammer_and_pick: カスタマイズするには

### 追加の必要要件

- Node.js v18.14.1 かそれ以上
- Git

### ステップ

1. 下記コマンドを実行して秘密情報を環境変数に設定

```sh
export NOTION_API_SECRET=<YOUR_NOTION_API_SECRET>
export DATABASE_ID=<YOUR_DATABASE_ID>
export PUBLIC_GA_TRACKING_ID=<YOUR_GA_TRACKING_ID>
```

2. 依存関係をインストールしローカルサーバーを起動

```sh
npm install
npm run dev
```

3. ブラウザで [http://localhost:4321](http://localhost:4321) を開く
4. 開発サーバーを停止するにはターミナルで `Ctrl+C`

### その他の情報

[wiki](https://github.com/otoyo/astro-notion-blog/wiki)
[初心者がastro-notion-blogをカスタマイズしてみた【具体例多数】](https://rakuraku-engineer.com/posts/anb-custom/)
[【astro-notion-blog】右メニューに目次を追加してみた](https://varubogu.com/posts/astro-notion-blog-add-headline/)
[astro-notion-blogカスタマイズメモ](https://suzu-mono-gram.com/blog/astro-notion-blog-memo/)

### Google Analytics の設定

1. [Google Analytics](https://analytics.google.com/) でプロパティを作成し、測定IDを取得（G-XXXXXXXXXX形式）
2. 取得した測定IDを`PUBLIC_GA_TRACKING_ID`としてGitHub Secretsに追加
3. この設定により、Google Analytics と Google Search Console の両方が利用可能になります

## 📝 注意事項とトラブルシューティング

このプロジェクトの運用に関する注意事項や、発生しうる問題とその解決策をまとめます。

### 1. フォークの同期と競合解決について

このリポジトリは `otoyo/astro-notion-blog` からフォークされています。元のリポジトリに更新があってフォークを同期する際には、**競合 (Conflict)** が発生する可能性があります。

### 2. GitHub Actionsの権限エラー (ワークフローファイルの更新拒否)

`stefanzweifel/git-auto-commit-action` を使用した際に、以下のようなエラーでプッシュが拒否されることがあります。これは、**GitHub Actionsがワークフローファイルを更新する権限がない**ために発生します。

**解決策:**

1.  **リポジトリのGitHub Actions設定を確認:**
    - GitHubリポジリトの「Settings」タブを開く。
    - 左側のサイドバーで「Actions」を展開し、「General」をクリック。
    - 「Workflow permissions」セクションで**「Read and write permissions」**が選択されていることを確認し、保存する。

2.  **`format.yml` (または自動コミットを使用しているワークフロー) の確認:**
    - `jobs.<job_name>` の直下に `permissions: contents: write` があることを確認。
    - `actions/checkout@v4` ステップで `with: token: ${{ secrets.GITHUB_TOKEN }}` が設定されていることを確認。

3.  **`.prettierignore` の確認:**
    - `npm run format` がワークフローファイル（`.github/workflows/*.yml`）を整形しないよう、`.prettierignore` ファイルに `*.yml` の除外設定が追加されていることを確認する。

### 3. 環境変数が定義されていないエラー (`PUBLIC_GA_TRACKING_ID is not defined`)

デプロイビルド時に、環境変数が定義されていないというエラーが発生した場合。

**解決策:**

1.  GitHubリポジトリのSecretsに環境変数が正しく設定されていることを確認:
    - 「Settings」→「Secrets and variables」→「Actions」で、必要な環境変数（例: `PUBLIC_GA_TRACKING_ID`）が正確な名前と値で登録されていることを確認する。

2.  デプロイワークフロー (`lolipop.yml`など) で環境変数が正しく渡されていることを確認:
    - ビルドステップの `env:` ブロックに、`PUBLIC_GA_TRACKING_ID: ${{ secrets.PUBLIC_GA_TRACKING_ID }}` のように記載されていることを確認する。

3.  **`astro.config.mjs` での参照方法を確認:**
    - `astro.config.mjs` 内で環境変数を参照する際には、`process.env.PUBLIC_GA_TRACKING_ID` のように `process.env.` を使用していることを確認する。
      例: `'import.meta.env.PUBLIC_GA_TRACKING_ID': JSON.stringify(process.env.PUBLIC_GA_TRACKING_ID)`

## :two_hearts:

astro-notion-blogの開発者さまを支援するには[![GitHub sponsors](https://img.shields.io/static/v1?label=Sponsor&message=%E2%9D%A4&logo=GitHub&color=%23fe8e86)](https://github.com/sponsors/otoyo)
