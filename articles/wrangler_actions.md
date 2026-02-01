---
title: "wrangler-actionsを使ってCloudflare Pagesを自動デプロイする" # 記事のタイトル
emoji: "😘" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [Cloudflare,CloudflarePages,wrangler,GitHubActions] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## はじめに
どうもK1mu21です。
今回はCloudflare Pagesに対してGitHub Actionsを使って自動デプロイを行う方法について紹介します。
最近GitLab CI/CDを利用してCloudflare pagesにデプロイをしていましたが、Github Actionsに移行してより簡単にCloudflare Pagesにデプロイできるようになったので、方法を共有したいと思いました。

個人開発ではCloudflareは無料枠が充実してるので利用している方は多いと思うので、wranglerを使って自動デプロイする方法を知っておくと便利かと思います。

## 手順
### 1. Cloudflare APIトークンの作成

まず、Cloudflareのダッシュボードにログインし、APIトークンを作成します。

### 2. GitHubリポジトリの設定

GitHubリポジトリのSettings > Secrets and variables > Actionsに移動し、以下のシークレットを追加します。

- `CLOUDFLARE_API_TOKEN`: 先ほど作成したCloudflareのAPIトークン
- `CLOUDFLARE_ACCOUNT_ID`: CloudflareアカウントID

1,2に関しては公式サイトを参考にしてそのまま進めれば設定できるかと思います
https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/

### 3. GitHub Actionsワークフローの作成

次に、GitHubリポジトリのルートにある`.github/workflows`ディレクトリに新しいYAMLファイルを作成します

デプロイにはwrangler-actionを使用します
https://github.com/cloudflare/wrangler-action

```yaml:deploy.yaml
...
deploy:
    name: Deploy to Cloudflare Pages
    runs-on: ubuntu-latest
    needs: build
    environment: 
        name: ${{ github.ref_name == 'refs/heads/main' && 'production' || 'preview' }}
        url: ${{ github.ref_name == 'refs/heads/main' && '<your_domain>' || format('https://{0}.{1}.pages.dev', steps.slug.outputs.short_slug, '<project-name>') }}
    steps:
        - uses: actions/checkout@v6
        - name: Generate short slug
          id: slug
          run: |
            BRANCH_NAME=${{github.head_ref }}
            SHORT_SLUG=$(echo "$BRANCH_NAME" | tr '[:upper:]' '[:lower:]' | cut -c1-28 | tr '/' '-')
            echo "short_slug=$SHORT_SLUG" >> $GITHUB_OUTPUT
        # Build stepから成果物を受け取る想定
        - name: Download Artifacts
            uses: actions/download-artifact@v7
            with:
              name: dist
              path: ./dist
        - name: Build & Deploy (production)
            if: github.ref_name == 'refs/heads/main'
            uses: cloudflare/wrangler-action@v3
            with:
              apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
              accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
              command: pages deploy --project-name=<project-name>
        - name: Build & Deploy (preview)
            if: github.ref_name != 'refs/heads/main'
            uses: cloudflare/wrangler-action@v3
            with:
              apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
              accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
              command: pages deploy --project-name=<project-name> --branch=${{ github.head_ref }}
...
```

wranglerはブランチ名を指定すれば`https://<brahch-name>.プロジェクト名.pages.dev/`のようにデプロイされるので、mainにmergeされたら本番環境、それ以外だとpreview環境にデプロイするようにActionsを設定することができます

また、preview環境のbranch-nameにはブランチ名が入りますが、最大28文字までしか入らずそれ以上の長さは自動で切り捨てられるという仕様もあります
他にブランチの大文字は小文字に変換されるなどのルールもあるので、注意が必要です

PR上からpreview環境にアクセスしたい場合は、URLはCloudflare仕様に合わせたものをenvironment変数に設定しておくとview deploymentボタンから遷移できるので設定しておくと便利です

#### 4.PRがcloseされたときにpreview環境を削除するワークフローの作成

PRがcloseされたときなどにpreview環境を削除するワークフローも作成しておいた方が安全です。

```yaml:delete-preview.yaml
...
on:
  pull_request:
    types: [closed]
env:
  PROJECT_NAME: <project-name>

jobs:
    delete-preview:
        name: Delete Cloudflare Pages Preview Deployment
        runs-on: ubuntu-latest
        steps:
            - name: Delete Preview Deployment
              uses: cloudflare/wrangler-action@v3
              with:
                ApiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
                AccountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
              run: |
                # 生成されたブランチ名を取得
                BRANCH_NAME=${{github.head_ref }}
                SHORT_SLUG=$(echo "$BRANCH_NAME" | tr '[:upper:]' '[:lower:]' | cut -c1-28 | tr '/' '-')
                # 対象ブランチのID取得
                DEPLOYMENTS_ID=$(curl --silent --request GET "https://api.cloudflare.com/client/v4/accounts/$AccountID/pages/projects/$PROJECT_NAME/deployments" \
                  --header "Authorization: Bearer $ApiToken" \
                  --header "Content-Type: application/json" | jq -r ".result[] | select(.deployment_trigger.metadata.branch==\"$SHORT_SLUG\") | .id")
                # デプロイ削除
                curl --request DELETE "https://api.cloudflare.com/client/v4/accounts/$AccountID/pages/projects/$PROJECT_NAME/deployments/$DEPLOYMENTS_ID?force=true" \
                  --header "Authorization: Bearer $ApiToken" \
                  --header "Content-Type: application/json"

```

実は、Cloudflare PagesはPRがMergeされてbranchが削除されたとしても、Preview環境に同期されているわけではないので削除されません
Cloudflareのコンソール上から削除を行うこともできますが、見た目から消えてるだけで実はPreview URLを直指定することでアクセスが可能になっています

https://developers.cloudflare.com/api/resources/pages/

参考になるissue
https://community.cloudflare.com/t/how-to-delete-aliased-preview-deployments/269292/25

完全に削除するにはCloudflareのAPIを叩かないといけないという罠も仕組まれているため、GitHub Actionsで自動化しておくと便利です

## おわりに

今回はCloudflare Pagesに対してGitHub Actionsを使って自動デプロイする方法について紹介しました
wrangler-actionを使うことで簡単にデプロイが可能になりますし、PRごとにpreview環境を立てることもできるので非常に便利です
個人開発でCloudflare Pagesを利用している方は多いと思うので、ぜひ試してみてください
また、静的サイトレベルなら業務利用でも十分Cloudflare Pagesは使えると思うので、興味がある方は検討してみてはいかがでしょうか
