---
title: GitHub Actions＋OIDCで自動デプロイする
weight: 2
date: 2026-10-20
description: アクセスキーを使わず、OIDC で IAM ロールを一時的に借りて、push するだけで S3＋CloudFront にデプロイされるようにする。信頼ポリシーと許可ポリシーの違いも整理する。
tags: [aws, iam, oidc, github-actions, cloudfront]
---

[S3＋CloudFrontで技術ブログを公開する（手作業編）](../s3-cloudfront-hosting/) では、手元の Mac から3つのコマンドでデプロイしていた。
これを GitHub Actions に移し、`main` に push するだけで公開されるようにする。

{{< callout type="info" >}}
記事中の `<account-id>`、`<bucket>`、`<distribution-id>`、`<owner>`、`<repo>` は自分の環境の値に読み替える。
{{< /callout >}}

## 手動デプロイの何が困るか

| 困ること | 自動デプロイでどうなるか |
|---|---|
| コマンドの打ち間違いや、順番の間違いが起こりうる | 毎回同じ手順で動く |
| 強い権限（`AdministratorAccess`）でデプロイしている | デプロイ専用の最小権限のロールで動く |
| Git に入れていない記事でも公開できてしまう | `main` にあるものだけが公開される |

## なぜ OIDC を使うのか

GitHub Actions から AWS を操作する方法は大きく2つある。

| 方法 | GitHub に保存するもの | 漏れたときの影響 |
|---|---|---|
| IAM ユーザーのアクセスキー | 長期間有効なアクセスキー（Secrets に保存） | キーを知った人が、どこからでも AWS を操作できる。自分で無効化するまで使える |
| OIDC | 何もない | 盗むものがない。借りられる権限も最大1時間で切れる |

OIDC では「誰か」ではなく「どのリポジトリの、どのブランチで動いているか」で本人確認をする。
GitHub が署名した身分証明書（OIDC トークン）を AWS に見せて、IAM ロールを一時的に借りる。

```text
GitHub Actions ── ① 身分証明書を発行してもらう ──→ GitHub（token.actions.githubusercontent.com）
     │
     └── ② 身分証明書を見せてロールを借りる ──→ AWS STS
                                                  │ 信頼ポリシーの条件と照合
                                                  ↓
          ③ 一時的な鍵（最大1時間）を受け取る ←──┘
     │
     └── ④ その鍵で S3 に同期、キャッシュを削除
```

`aws sso login` でコンソールや CLI に入るのも、仕組みは同じ。
「ブラウザでのログイン」の代わりに「GitHub が発行する身分証明書」を使っている。

## 信頼ポリシーと許可ポリシー

IAM ロールには、性格の違う2つのポリシーが付く。

| | 信頼ポリシー | 許可ポリシー |
|---|---|---|
| 決めること | **誰が** このロールを借りられるか | 借りた人が **何を** できるか |
| 画面の場所 | ロールの「信頼関係」タブ | ロールの「許可」タブ |
| 今回の中身 | `<owner>/<repo>` の `main` で動く GitHub Actions だけ | このバケットへの同期と、このディストリビューションのキャッシュ削除だけ |

レンタカーでたとえると、信頼ポリシーは「誰に車を貸すか」、許可ポリシーは「貸した車で行ってよい範囲」。
どちらか片方だけでは成り立たない。

同じアカウントにある、ほかのロールの信頼ポリシーと比べると分かりやすい。

| ロール | 信頼ポリシー（誰が借りられるか） |
|---|---|
| `AWSReservedSSO_AdministratorAccess_...` | IAM Identity Center でログインした人 |
| `OrganizationAccountAccessRole` | 管理アカウントの管理者 |
| `github-actions-tech-blog-deploy` | `<owner>/<repo>` の `main` で動く GitHub Actions |

## 1. ID プロバイダーを追加する

「GitHub Actions が発行する身分証明書を、このアカウントは信用する」という登録。

IAM →「ID プロバイダー」→「プロバイダーを追加」

| 項目 | 値 |
|---|---|
| プロバイダーのタイプ | OpenID Connect |
| プロバイダーの URL | `https://token.actions.githubusercontent.com` |
| 対象者（Audience） | `sts.amazonaws.com` |

## 2. デプロイ用のロールを作る

IAM →「ロール」→「ロールを作成」

| 項目 | 値 |
|---|---|
| 信頼されたエンティティタイプ | ウェブアイデンティティ |
| ID プロバイダー | `token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |
| GitHub 組織 / リポジトリ / ブランチ | `<owner>` / `<repo>` / `main` |
| 許可ポリシー | 何も選ばない（あとでインラインポリシーを付ける） |
| ロール名 | `github-actions-tech-blog-deploy` |

### 信頼ポリシー

```json {filename="trust-policy.json"}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<owner>@<owner-id>/<repo>@<repo-id>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

| 部分 | 意味 |
|---|---|
| `Principal` | 借りる人。GitHub の ID プロバイダー |
| `Action` | 借り方。外部の身分証明書を見せて借りる `AssumeRoleWithWebIdentity` |
| `aud` の条件 | 身分証明書が AWS 向けに発行されたものであること |
| `sub` の条件 | 身分証明書が「このリポジトリの `main` ブランチ」のものであること |

{{< callout type="warning" >}}
`Principal` だけだと「GitHub Actions なら誰でも」借りられてしまう。`sub` の条件で、自分のリポジトリとブランチに必ず絞る。
{{< /callout >}}

### 許可ポリシー

ロールを開き、「許可を追加」→「インラインポリシーを作成」→「JSON」

```json {filename="deploy-policy.json"}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucketForSync",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::<bucket>"
    },
    {
      "Sid": "UploadAndDeleteObjects",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::<bucket>/*"
    },
    {
      "Sid": "InvalidateCache",
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::<account-id>:distribution/<distribution-id>"
    }
  ]
}
```

| 許可 | 何のため |
|---|---|
| `s3:ListBucket`（バケットに対して） | `sync` が S3 にあるファイルと比べるため |
| `s3:PutObject`、`s3:DeleteObject`（バケットの中身に対して） | 送信と、`--delete` での削除のため |
| `cloudfront:CreateInvalidation`（このディストリビューションだけ） | キャッシュの削除のため |

`ListBucket` はバケット（`arn:aws:s3:::<bucket>`）、`PutObject` などは中身（`arn:aws:s3:::<bucket>/*`）が対象なので、`Resource` を分けて書く。

## 3. GitHub に変数を登録する

リポジトリの「Settings」→「Secrets and variables」→「Actions」→「Variables」

| 変数名 | 値 |
|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::<account-id>:role/github-actions-tech-blog-deploy` |
| `S3_BUCKET` | `<bucket>` |
| `CLOUDFRONT_DISTRIBUTION_ID` | `<distribution-id>` |
| `SITE_URL` | `https://dxxxx.cloudfront.net/`（末尾の `/` まで） |

### Secrets ではなく Variables にした理由

| | Secrets | Variables |
|---|---|---|
| 保存のされ方 | 暗号化され、登録後は誰も値を見られない | そのまま保存される |
| ログでの表示 | `***` に伏せられる | そのまま表示される |
| 入れるもの | 知られたら悪用されるもの | 知られても困らない設定値 |

4つとも、知っただけでは何もできない値なので Variables にした。

- ロールの ARN：借りるには「このリポジトリの `main` で動いている」という GitHub の署名入りの身分証明書が必要
- バケット名：直接アクセスしても 403。アカウントの外から来て拒否されたリクエストには課金されない
- ディストリビューション ID：AWS の権限がなければ何も操作できない
- サイトの URL：もともと公開している

公開リポジトリでは Actions のログも公開されるので、Variables の値は実質的に公開される前提で考える。
秘密情報が1つもないのは OIDC のおかげ。アクセスキーを使っていたら、必ず Secrets に入れる必要があった。

## 4. ワークフローを書く

```yaml {filename=".github/workflows/deploy.yml"}
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

# OIDC で AWS のロールを借りるために id-token: write が必要
permissions:
  id-token: write
  contents: read

# デプロイが同時に2つ走らないようにする（後から来たものは前の完了を待つ）
concurrency:
  group: deploy
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.166.0
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          # 記事の最終更新日を git の履歴から取るため、履歴を全部取得する
          fetch-depth: 0

      # Hextra テーマを Hugo Modules で取得するために Go が必要
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.23"
          cache: false

      - name: Install Hugo
        run: |
          wget -q -O hugo.deb "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb"
          sudo dpkg -i hugo.deb
          hugo version

      - name: Build
        run: hugo --minify --baseURL "${{ vars.SITE_URL }}"

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1

      # 順番が大事：同期してからキャッシュを削除する
      - name: Sync to S3
        run: aws s3 sync public/ "s3://${{ vars.S3_BUCKET }}/" --delete

      - name: Invalidate CloudFront cache
        run: aws cloudfront create-invalidation --distribution-id "${{ vars.CLOUDFRONT_DISTRIBUTION_ID }}" --paths "/*"
```

| 設定 | 意味 |
|---|---|
| `on.push.branches: [main]` | `main` への push で動く。`workflow_dispatch` で画面から手動実行もできる |
| `permissions.id-token: write` | GitHub に身分証明書を発行させる許可。これがないとロールを借りられない |
| `concurrency` | push が続いても、デプロイが同時に2つ走らない |
| `fetch-depth: 0` | `enableGitInfo` で記事の更新日を出すため、git の履歴を全部取る |
| `HUGO_VERSION` | 手元と同じバージョンに固定し、環境による差をなくす |

手動デプロイの3つのコマンド（ビルド → 同期 → キャッシュの削除）を、そのまま書き写しただけになっている。

## つまずいたところ：`Not authorized to perform sts:AssumeRoleWithWebIdentity`

最初の実行で、ロールを借りる手順が次のエラーで失敗した。

```text
Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

変数とロールの ARN は正しかったので、身分証明書の中身（`sub`）が信頼ポリシーの条件と合っていないと考えた。
リポジトリの OIDC の設定を確認すると、次のようになっていた。

```bash
gh api repos/<owner>/<repo>/actions/oidc/customization/sub
```

```json
{"use_default":true,"use_immutable_subject":true,"sub_claim_prefix":"repo:<owner>@<owner-id>/<repo>@<repo-id>"}
```

`use_immutable_subject: true` のリポジトリでは、`sub` にユーザーとリポジトリの **数字の ID** が入る。

| | `sub` の値 |
|---|---|
| ロール作成画面が作った条件 | `repo:<owner>/<repo>:ref:refs/heads/main` |
| GitHub が実際に送ってくる値 | `repo:<owner>@<owner-id>/<repo>@<repo-id>:ref:refs/heads/main` |

名前だけだと、リポジトリを消したあとに別の人が同じ名前で作り直すと、その人の Actions がロールを借りられてしまう。
数字の ID は作り直すと変わるので、その心配がない。

信頼ポリシーの `sub` を実際の値に直し、値が完全に決まっているので `StringLike` ではなく `StringEquals`（完全一致）にした。

{{< callout type="info" >}}
借りるのに失敗した記録は CloudTrail の `AssumeRoleWithWebIdentity` イベントでも確認できる。ただし記録が出るまで15分ほどかかることがある。
{{< /callout >}}

## 5. 確認する

```bash
gh run watch
```

| 確認すること | 期待する結果 |
|---|---|
| Actions の実行 | すべての手順に緑のチェック |
| 「Configure AWS credentials」のログ | ロールを借りられている |
| サイト | 記事の変更が反映されている |
| `curl -I https://dxxxx.cloudfront.net/` | `200` |

## セキュリティで気をつけること

- 悪用の余地が残るのは「`main` ブランチにワークフローを書き込まれる」場合だけ。書き込める人を増やすときは注意する
- フォークからのプルリクエストでは `sub` が `main` にならないうえ、このワークフローは `main` への push でしか動かないので、ロールは借りられない
- 許可ポリシーは、デプロイに必要な3種類の操作だけ。バケットやディストリビューションの設定変更はできない

## 次にやること

ここまでコンソールで作ったもの（S3、CloudFront、CloudFront Functions、ID プロバイダー、ロール）を Terraform に取り込む。
