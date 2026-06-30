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

## コラボレーター招待

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/collaborators/<username> \
  -d '{"permission":"push"}'
```
