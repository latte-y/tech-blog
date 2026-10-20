---
title: S3＋CloudFrontで技術ブログを公開する（手作業編）
weight: 1
date: 2026-10-03
description: Hugoで作ったブログを、非公開のS3とCloudFront（OAC）で配信するまでの手順。まずはAWSのコンソールで手作業で作り、仕組みを理解する。
tags: [aws, s3, cloudfront, hugo]
---

このブログ自体を、Hugo でビルドして S3＋CloudFront で配信するまでの記録。
あとで Terraform と GitHub Actions に置き換える前提で、まずはコンソールで1つずつ作って仕組みを理解する。

{{< callout type="info" >}}
記事中の `<bucket>`、`<distribution-id>`、`dxxxx.cloudfront.net` は自分の環境の値に読み替える。
{{< /callout >}}

## 全体の構成

```text
手元の Mac ── hugo でビルド ── aws s3 sync ──→ S3（非公開）
                                                  ↑ OAC で読み取り
読者のブラウザ ── HTTPS ──→ CloudFront ───────────┘
                             └ CloudFront Functions（/xxx/ → /xxx/index.html）
```

| 部品 | 役割 |
|---|---|
| Hugo | Markdown を HTML・CSS・JS に変換して `public/` に書き出す |
| S3 | 変換したファイルの置き場所。インターネットからは直接見えないようにする |
| CloudFront | 読者への配信。HTTPS と世界中のキャッシュを提供する |
| OAC（Origin Access Control） | CloudFront だけが S3 を読めるようにする仕組み |
| CloudFront Functions | フォルダの URL を `index.html` に読みかえる |

### GitHub Pages や Amplify ではなく S3＋CloudFront にした理由

ブログを公開するだけなら GitHub Pages や Amplify Hosting のほうが簡単。
今回はあえて、それらが自動でやってくれる部分（OAC、IAM の最小権限、キャッシュの削除、OIDC）を自分で組んで理解するために、S3＋CloudFront を選んだ。

## 1. ブログ専用の AWS アカウントを作る

実験用のアカウントとは分けて、ブログ専用のアカウントを AWS Organizations で作った。
実験の後片付けで、ブログのリソースを消してしまわないようにするため。

1. 管理アカウントに SSO ユーザー（`AdministratorAccess`）でログインする
2. **AWS Organizations** →「AWS アカウントを追加」→「AWS アカウントを作成」
   - アカウント名：`tech-blog`
   - メールアドレス：Gmail の `+` を使い、アカウントごとに別のアドレスにする（例：`name+aws-tech-blog@gmail.com`）
3. **IAM Identity Center** →「AWS アカウント」→ 新しいアカウントを選ぶ →「ユーザーまたはグループを割り当て」
   - ユーザー：普段使っている SSO ユーザー
   - 権限セット：`AdministratorAccess`
4. AWS アクセスポータルから新しいアカウントに入れることを確認する

{{< callout >}}
「割り当て」は「この人が、このアカウントに、この権限で入れる」という登録。
割り当てると、アカウントの中に `AWSReservedSSO_<権限セット名>_<ランダムな文字列>` という IAM ロールが自動で作られ、ログインするとそのロールを一時的に借りた状態になる。
SSO ユーザー自体がアカウントの中にコピーされるわけではない。
{{< /callout >}}

ルートユーザーとの見分け方は、CloudShell で `aws sts get-caller-identity` を実行して `Arn` を見るのが確実。

| ログインしている人 | `Arn` の形 |
|---|---|
| ルートユーザー | `arn:aws:iam::<アカウントID>:root` |
| SSO ユーザー | `arn:aws:sts::<アカウントID>:assumed-role/AWSReservedSSO_.../<ユーザー名>` |

## 2. Hugo でビルドする

```bash
rm -rf public && hugo --minify
```

| 部品 | 意味 |
|---|---|
| `rm -rf public` | 前回のビルド結果を消す。Hugo は古いファイルを消さないので、毎回まっさらな状態から作る |
| `hugo` | Markdown を HTML に変換して `public/` に書き出す |
| `--minify` | 改行や空白を詰めてファイルを小さくする（見た目は変わらない） |

記事1つにつきフォルダ1つが作られる点がポイント。
たとえば `content/docs/sql/common.md` は `public/docs/sql/common/index.html` になり、URL は `/docs/sql/common/` になる。

## 3. S3 バケットを作る

リージョンは東京（`ap-northeast-1`）。

| 項目 | 設定 | 理由 |
|---|---|---|
| バケットタイプ | 汎用 | |
| バケット名 | `tech-blog-<アカウントID>` | バケット名は世界中で重複できないので、アカウント ID を付ける |
| オブジェクト所有者 | ACL 無効 | アクセスはバケットポリシーだけで管理する |
| パブリックアクセスのブロック | 4項目すべてオン | 読者は CloudFront 経由でだけ見る |
| バージョニング | 無効 | 元の記事は Git にあるので不要 |
| 暗号化 | SSE-S3（初期値） | 追加料金なしで保存時に暗号化される |
| 静的ウェブサイトホスティング | 無効 | S3 自体を公開する機能なので使わない |
| タグ | `Project = tech-blog` | 費用の確認と、あとで Terraform に取り込むときの目印 |

### ファイルのアップロード

`public/` の **中身** を選択して、アップロード画面にドラッグ＆ドロップする。
`public` フォルダごと入れると、URL が `/public/index.html` になってしまう。

- ストレージクラスはスタンダード（小さいファイルを何度も読むサイトには一番安くて速い）
- アップロード後、`index.html` のメタデータの `Content-Type` が `text/html` になっていることを確認する

{{< callout type="warning" >}}
Finder からドラッグすると、Mac が作る隠しファイル `.DS_Store` も一緒にアップロードされることがある。
あとで `aws s3 sync --delete` を使えば消える。
{{< /callout >}}

## 4. CloudFront ディストリビューションを作る

「ディストリビューションを作成」のウィザードで進める。

| 画面 | 項目 | 設定 |
|---|---|---|
| 料金プラン | プラン | Free（選べなければ従量課金） |
| 始める | ディストリビューション名 | `tech-blog`（一覧で見分けるための名前。URL には影響しない） |
| | 種類 | 単一のウェブサイトまたはアプリ |
| ドメイン | 独自ドメイン | 使わない |
| オリジン | オリジンタイプ | Amazon S3 |
| | S3 オリジン | 作ったバケットを選ぶ（`s3-website` の付いた方ではない） |
| | 設定 | 推奨のオリジン設定を使う（OAC の作成と圧縮の設定が入る） |
| セキュリティ | WAF | 従量課金なら有効にしない（月額がかかる） |

作成すると、CloudFront がバケットポリシーを自動で更新してくれる。
作成後、「一般」→「設定」の「デフォルトルートオブジェクト」に `index.html` を設定する。

### この時点での確認

| 開く URL | 結果 | 理由 |
|---|---|---|
| `https://dxxxx.cloudfront.net/` | 表示される | デフォルトルートオブジェクトが効いている |
| `https://dxxxx.cloudfront.net/docs/sql/common/index.html` | 表示される | ファイルは置けている |
| `https://dxxxx.cloudfront.net/docs/sql/common/` | AccessDenied | `docs/sql/common/` というファイルは存在しない |
| `https://<bucket>.s3.ap-northeast-1.amazonaws.com/index.html` | AccessDenied | S3 は直接読めない（正しい状態） |

ファイルがないのに 404 ではなく 403（AccessDenied）になるのは、CloudFront に `s3:GetObject` だけを許可し、`s3:ListBucket` を許可していないから。
この場合 S3 は「ファイルがない」ことも教えずに一律で拒否する。

{{< callout type="info" >}}
S3 の画面に出る `s3://<bucket>/index.html` は「S3 URI」で、CLI などのツール用。ブラウザでは開けない。
ブラウザで開くのは `https://` で始まる「オブジェクト URL」の方。
{{< /callout >}}

## 5. CloudFront Functions で URL を補完する

S3 を非公開にすると、S3 のウェブサイト機能（フォルダの URL に `index.html` を補う機能）は使えない。
代わりに CloudFront Functions で、リクエストの URL を書き換える。

1. CloudFront →「関数」→「関数を作成」（ランタイム：`cloudfront-js-2.0`）
2. 次のコードを貼って保存する

```js {filename="tech-blog-index-rewrite.js"}
function handler(event) {
  var request = event.request;
  var uri = request.uri;

  if (uri.endsWith('/')) {
    // /docs/sql/common/ → /docs/sql/common/index.html
    request.uri += 'index.html';
  } else if (!uri.includes('.')) {
    // /docs/sql/common（末尾の / なし）→ /docs/sql/common/index.html
    request.uri += '/index.html';
  }
  // .css や .js など拡張子付きのファイルはそのまま
  return request;
}
```

3. 「テスト」タブで、URL パス `/docs/sql/common/` の結果が `/docs/sql/common/index.html` になることを確認する
4. 「発行」タブで発行する（保存しただけでは使えない）
5. ディストリビューションに関連付ける
   - イベントタイプ：ビューワーリクエスト（キャッシュを探す前に書き換えるため）
   - キャッシュビヘイビア：デフォルト（`*`）

## 6. 404 ページを設定する

存在しないページで S3 の AccessDenied の XML が出ないように、Hugo が作る `404.html` を返す。

ディストリビューション →「エラーページ」→「カスタムエラーレスポンスを作成」

| 項目 | 設定 |
|---|---|
| HTTP エラーコード | 403 |
| エラーキャッシュ最小 TTL | 10 秒 |
| レスポンスページのパス | `/404.html` |
| HTTP レスポンスコード | 404 |

S3 は「ファイルがない」ときも 403 を返すので、403 を受けて 404 ページを返し、状態コードも 404 に直す。

## 7. コマンドでデプロイする

### AWS CLI のログイン設定（SSO）

```bash
aws configure sso
```

- SSO start URL：AWS アクセスポータルの URL
- アカウント：`tech-blog`、ロール：`AdministratorAccess`
- プロファイル名：`tech-blog`

```bash
aws sts get-caller-identity --profile tech-blog
```

長期間有効なアクセスキーは作らない。CLI もコンソールと同じく、ロールを一時的に借りてログインする。
期限が切れたら `aws sso login --profile tech-blog` で入り直す。

### デプロイの3ステップ

```bash
rm -rf public && hugo --minify --baseURL https://dxxxx.cloudfront.net/
```

```bash
aws s3 sync public/ s3://<bucket>/ --delete --profile tech-blog
```

```bash
aws cloudfront create-invalidation --distribution-id <distribution-id> --paths "/*" --profile tech-blog
```

| コマンド | 役割 |
|---|---|
| `hugo --baseURL ...` | 本番の URL を埋め込んでビルドする（サイトマップや canonical に使われる） |
| `aws s3 sync --delete` | `public/` と S3 を同じ状態にする。変わったファイルだけ送り、手元にないファイルは S3 から消す |
| `create-invalidation --paths "/*"` | エッジのキャッシュを捨てさせる。`/*` は1パスと数えるので無料枠（月1,000パス）に収まる |

{{< callout type="warning" >}}
`--delete` は送り先のバケットの中身を消せる強力なオプション。バケット名と `--profile` が正しいか必ず確認する。
{{< /callout >}}

### 順番が大事

**ビルド → 同期 → キャッシュの削除** の順でないといけない。
キャッシュを先に削除すると、同期が終わる前に来たアクセスで S3 の古いファイルが再びキャッシュされ、最大24時間（キャッシュの既定の保持時間）古いページが配られてしまう。

### 確認

| 確認すること | 期待する結果 |
|---|---|
| `aws s3 sync ... --dryrun` | 何も出ない（差分なし） |
| `aws cloudfront list-invalidations` | `Completed` |
| `curl -I https://dxxxx.cloudfront.net/docs/sql/common/` | `200`、`x-cache: Miss from cloudfront`（2回目以降は `Hit`） |
| `curl -I https://dxxxx.cloudfront.net/abc/` | `404` |

## 費用の目安

個人ブログの規模なら月1円にも届かない見込み。

- S3：保存料は 1GB あたり月約0.025ドル、このブログは約1MB
- S3 → CloudFront の転送：無料
- CloudFront：従量課金でも毎月の無料枠（1TB の転送、1,000万リクエスト）に収まる。Free プランなら月0ドルで、使いすぎても追加料金なし

## 次にやること

1. GitHub にリポジトリを作る
2. OIDC と GitHub Actions で、push したら自動でデプロイされるようにする → [GitHub Actions＋OIDCで自動デプロイする](../github-actions-oidc-deploy/)
3. ここまでコンソールで作ったものを Terraform に取り込む（`import` ブロック）
