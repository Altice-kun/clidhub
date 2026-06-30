# GitHub REST API リファレンス (Clidhub用)

gh CLIでカバーしきれない、または1コマンドで完結させたい操作のためのcurlサンプル集。
全リクエストで `$GITHUB_TOKEN` 環境変数が設定済みであることが前提。

> **注**: REST API (`curl`) は `Authorization: Bearer $GITHUB_TOKEN` で問題なく認証できる。
> ただし `git push` 等のgit操作はBearerでは通らないことがあるため、SKILL.mdの「git pushの認証方式」セクションに従い `x-access-token:<TOKEN>` のBasic認証を使うこと。両者は別物として扱う。

共通ヘッダー:
```bash
-H "Authorization: Bearer $GITHUB_TOKEN"
-H "Accept: application/vnd.github+json"
-H "X-GitHub-Api-Version: 2022-11-28"
```

## 単一ファイルの取得・作成・更新

### ファイル取得(SHAも必要なので更新前に必ず取得)
```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<owner>/<repo>/contents/<path>
```
レスポンスの `sha` フィールドを更新時に使う。

### ファイル作成・更新
```bash
CONTENT_B64=$(base64 -w0 < ./local_file.txt)
SHA=$(curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/contents/<path> | python3 -c "import sys,json;print(json.load(sys.stdin).get('sha',''))")

curl -s -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<owner>/<repo>/contents/<path> \
  -d "{\"message\":\"Update <path>\",\"content\":\"$CONTENT_B64\",\"sha\":\"$SHA\",\"branch\":\"main\"}"
```
新規作成の場合は `sha` を省略する。

### ファイル削除
```bash
curl -s -X DELETE \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/contents/<path> \
  -d "{\"message\":\"Delete <path>\",\"sha\":\"$SHA\",\"branch\":\"main\"}"
```

## ブランチ作成(REST API版)

```bash
# まずmainの最新コミットSHAを取得
BASE_SHA=$(curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/git/ref/heads/main | python3 -c "import sys,json;print(json.load(sys.stdin)['object']['sha'])")

curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/git/refs \
  -d "{\"ref\":\"refs/heads/feature/xxx\",\"sha\":\"$BASE_SHA\"}"
```

## ブランチ保護ルールの設定

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<owner>/<repo>/branches/main/protection \
  -d '{
    "required_status_checks": null,
    "enforce_admins": true,
    "required_pull_request_reviews": {
      "required_approving_review_count": 1
    },
    "restrictions": null
  }'
```

## GitHub Pages

### 有効化
```bash
curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<owner>/<repo>/pages \
  -d '{"source":{"branch":"main","path":"/docs"}}'
```

### 設定変更(既に有効な場合はPOSTではなくPUT)
```bash
curl -s -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<owner>/<repo>/pages \
  -d '{"source":{"branch":"main","path":"/docs"}}'
```

### 状態確認
```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/pages
```
`status: "built"` になれば公開完了。`html_url`に公開URLが入る。

## リポジトリ設定変更全般

```bash
curl -s -X PATCH \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo> \
  -d '{
    "description": "新しい説明文",
    "homepage": "https://example.com",
    "private": false,
    "has_issues": true,
    "has_wiki": false,
    "default_branch": "main"
  }'
```

## Secrets (Actions用の環境変数登録)

Actions内でAPIキー等を使う場合、リポジトリSecretsへの登録が必要。値は公開鍵で暗号化してから送信する必要があるため、`gh` CLIを使うのが圧倒的に簡単:

```bash
gh secret set MY_SECRET_NAME --body "secret-value" --repo <owner>/<repo>
```

REST APIで直接行う場合はlibsodiumでの暗号化が必須になるため、特別な理由がない限り`gh secret set`を使うこと。

## Projects (v2) GraphQL操作

Projects v2はREST APIではなくGraphQL APIで操作する。`gh project`でカバーできない細かい操作(カスタムフィールド値の更新等)向け。

### Project IDとフィールドIDの取得
```bash
curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  https://api.github.com/graphql \
  -d '{"query":"query{ user(login:\"<owner>\"){ projectV2(number: <project番号>){ id fields(first:20){ nodes{ ... on ProjectV2FieldCommon { id name } } } } } }"}'
```
Organization配下のプロジェクトの場合は `user(login:...)` を `organization(login:...)` に置き換える。

### アイテムのステータス(単一選択フィールド)を更新
```bash
curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  https://api.github.com/graphql \
  -d '{
    "query":"mutation($project:ID!, $item:ID!, $field:ID!, $value:String!){ updateProjectV2ItemFieldValue(input:{projectId:$project, itemId:$item, fieldId:$field, value:{singleSelectOptionId:$value}}){ projectV2Item{ id } } }",
    "variables":{"project":"<PROJECT_ID>","item":"<ITEM_ID>","field":"<FIELD_ID>","value":"<OPTION_ID>"}
  }'
```
各IDは事前にfields/itemsクエリで取得する必要がある。`gh project item-list <番号> --owner <owner> --format json` でアイテムIDは簡単に取得できる。

## プロフィール・フォロワー・メールアドレス・プランの読み取り(読み取り専用)

```bash
# 自分のプロフィール全体(login, name, company, location, bio, public_repos, followers, following, plan等)
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/user

# 他ユーザーの公開プロフィール(認証不要な情報のみ)
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/users/<username>

# フォロワー / フォロー中
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/user/followers
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/user/following

# メールアドレス一覧(primary/verifiedフラグ付き、Email addresses: Read-only権限が必要)
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/user/emails
```

`/user`のレスポンス中の`plan`オブジェクト(`name`: free/pro/team等、`space`: ストレージ容量、`private_repos`: プライベートリポジトリ上限)でプラン情報を確認できる。

403が返る場合は対応する権限(`Followers: Read-only`、`Email addresses: Read-only`等)がトークンに付与されていない可能性が高い。

## コラボレーター招待

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/collaborators/<username> \
  -d '{"permission":"push"}'
```
