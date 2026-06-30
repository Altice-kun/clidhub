---
name: clidhub
description: GitHubリポジトリの作成・構築・設定・編集・公開をClaudeが代行するスキル。リポジトリ作成、ファイルのコミット/push、ブランチ操作、プルリクエスト、Issues、GitHub Actions(CI/CD)設定、GitHub Pagesでの公開など、GitHubに関するあらゆる操作が必要な場面で必ず使用すること。「GitHubに上げて」「リポジトリ作って」「PR出して」「Actionsを設定して」「公開して」「リポジトリを編集して」など、GitHub操作を示唆する依頼があれば、明示的に「Clidhubを使って」と言われていなくても積極的にこのスキルを参照する。
---

# Clidhub

ユーザーのGitHubアカウントを操作し、リポジトリの作成からコード構築、設定、編集、公開までを行うスキル。

## 0. 事前準備: gh CLIの確認

`gh`コマンドが入っているか確認し、無ければインストールする(初回のみ。root環境では`sudo`が無いこともあるため有無を確認して実行):
```bash
which gh || { (sudo apt-get update -qq && sudo apt-get install -y -qq gh) 2>/dev/null || (apt-get update -qq && apt-get install -y -qq gh); }
```

## 1. 最初に必ず行うこと: 認証

Clidhubは**永続的にトークンを保存しない**。GitHubのPersonal Access Token (PAT) はセッション(=この会話)の間だけ使う一時的な認証情報として扱う。

1. 会話内でユーザーがまだPATを提示していなければ、最初にこう尋ねる:
   > 「GitHubのPersonal Access Tokenを教えてください。このトークンは今回のセッション内でのみ使用し、保存はしません。」

   **必要な権限(scope)**: GitHubには2種類のPATがあり、必要な権限が異なる。
   - **Classic PAT**(`ghp_`で始まる): `repo`スコープ(リポジトリ全般)+ `workflow`スコープ(Actions操作)があれば本スキルの操作はほぼ全てカバーできる。
   - **Fine-grained PAT**(`github_pat_`で始まる): リポジトリ単位で個別に権限を選ぶ方式。以下の **Repository permissions** が必要:
     - `Contents: Read and write`(ファイルpush/編集)
     - `Pull requests: Read and write`(PR作成/マージ)
     - `Issues: Read and write`(Issue操作)
     - `Actions: Read and write`(Actions実行/確認)
     - `Pages: Read and write`(GitHub Pages公開)
     - `Metadata: Read-only`(自動で付与される基本権限)
     - **新規リポジトリを作成させたい場合は追加で `Account permissions` → `Administration: Read and write` が必須**(これが無いと`gh repo create`が`Resource not accessible by personal access token`エラーで失敗する)。既存リポジトリの操作のみなら不要。
   
   実行中に権限不足のエラーが出たら、上記のどの権限が足りないかを具体的に伝え、ユーザーにトークン設定の見直しを依頼すること。
2. ユーザーがPATを提示したら、bashで環境変数として設定する(チャット上にその後トークンの値を繰り返し表示しない):
   ```bash
   export GITHUB_TOKEN="ユーザーから受け取ったトークン"
   export GH_TOKEN="$GITHUB_TOKEN"   # gh CLIはこちらの変数名を見る
   ```
3. 認証確認:
   ```bash
   gh auth status 2>/dev/null || curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/user
   ```
   ユーザー名が取得できればOK。失敗したらトークンのスコープ/有効期限を疑い、ユーザーに再確認する。

**重要な安全ルール:**
- トークンの値を会話の出力やファイル、コミットメッセージ、コード中に直接書き出さない。常に環境変数経由で参照する。
- `git remote`のURLにトークンを埋め込まない(`https://<token>@github.com/...`形式は履歴やログに残るリスクがあるため避ける)。

**git pushの認証方式(重要・要注意):**

`gh repo create --clone`でcloneしても、サンドボックス環境ではgit credential helperが正しく機能せず、素の`git push`が`could not read Username`エラーで失敗することがある。また`Authorization: Bearer $GITHUB_TOKEN`ヘッダーもgit pushでは通らないことがある。**確実に動作するのは以下の Basic認証形式**:

```bash
AUTH=$(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 -w0)
git -c http.extraHeader="Authorization: Basic $AUTH" push -u origin <branch>
```

毎回`-c http.extraHeader`を付けるのが面倒な場合、リポジトリ単位で設定してもよい(ただし設定ファイルにトークンが残るので、作業ディレクトリごと削除すれば問題ない一時的な作業領域に限る):
```bash
git config http.extraHeader "Authorization: Basic $AUTH"
```

一方、**REST API (`curl`) を直接叩く場合は `Authorization: Bearer $GITHUB_TOKEN` で問題なく動作する**。pushだけ方式が異なる点に注意。

- 各操作の前にscope(権限)が足りているか考え、足りなさそうならユーザーに伝える。

## 2. 操作手段の使い分け

Clidhubは `gh` CLI と GitHub REST API (`curl`) を**使い分ける**。判断基準:

| 操作 | 推奨手段 | 理由 |
|---|---|---|
| リポジトリ作成/削除/clone | `gh repo create` / `gh repo delete` | シンプルで対話的設定も楽 |
| ファイルの複数コミットpush | `git` (ローカルにclone→commit→push) | 通常のgitワークフローが最も確実 |
| 単一ファイルの軽量編集 | REST API (`PUT /repos/{owner}/{repo}/contents/{path}`) | cloneせず1ファイルだけ直すのに最適 |
| ブランチ作成 | `git checkout -b` または REST API (`POST /repos/{owner}/{repo}/git/refs`) | どちらでも可、gitの方が直感的 |
| PR作成/マージ | `gh pr create` / `gh pr merge` | gh CLIが圧倒的に楽 |
| Issue作成/編集 | `gh issue create` / `gh issue edit` | 同上 |
| GitHub Actions workflow追加 | `git` でyamlファイルをpush | ファイル操作なのでgitで十分 |
| Actions実行状況確認 | `gh run list` / `gh run view` | gh CLIが見やすい |
| リポジトリ設定変更(可視性、ブランチ保護等) | REST API (`PATCH /repos/{owner}/{repo}` 等) | gh CLIでカバーされない細かい設定が多い |
| GitHub Pages公開設定 | REST API (`POST /repos/{owner}/{repo}/pages`) | gh CLIに専用コマンドがない |

詳細なAPIエンドポイントやコマンド例は `references/api_reference.md` を参照。

## 3. 基本ワークフロー

> **注意**: 以下の例では簡潔さのため`git push`と書いているが、実際には前章の「git pushの認証方式」に従い、必ず以下のように`AUTH`変数を使ったコマンドに置き換えて実行すること:
> ```bash
> AUTH=$(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 -w0)
> git -c http.extraHeader="Authorization: Basic $AUTH" push ...
> ```

### 3.1 新規リポジトリを作って公開する

> **注意**: 環境のgit設定によっては、新規リポジトリのデフォルトブランチ名が`main`ではなく`master`になることがある。pushでエラーになったら`git branch -a`で確認し、必要なら`git branch -m master main`でリネームしてからpushする。

```bash
# 1. リポジトリ作成(privateで作成し、後で公開もできる)
gh repo create <repo-name> --public --description "説明文" --clone

# 2. 作業ディレクトリに移動してファイルを構築
cd <repo-name>
# ... ファイルを作成 ...

# 3. コミット & push
git add -A
git commit -m "Initial commit"
git push -u origin main
```

### 3.2 既存リポジトリにファイルを追加/編集する

```bash
gh repo clone <owner>/<repo-name> /home/claude/work_<repo-name>
cd /home/claude/work_<repo-name>
# 編集...
git add -A
git commit -m "変更内容を要約したメッセージ"
git push
```

cloneせず1ファイルだけ直したい場合はREST APIで直接更新できる(`references/api_reference.md`の「単一ファイル編集」参照)。

### 3.3 ブランチを切ってPRを出す

```bash
cd <repo-name>
git checkout -b feature/xxx
# 変更...
git add -A && git commit -m "feat: xxx"
git push -u origin feature/xxx
gh pr create --title "xxx" --body "変更内容の説明" --base main
```

マージまで頼まれたら:
```bash
gh pr merge <PR番号> --squash --delete-branch
```

### 3.4 GitHub Pagesで公開する

静的サイトを公開する場合、`gh-pages`ブランチまたは`main`ブランチの`/docs`配下を使う方式がある。シンプルなのは後者:

```bash
mkdir -p docs
mv index.html docs/   # 公開したいファイルをdocs/に配置
git add -A && git commit -m "Add GitHub Pages content"
git push
```

その後、REST APIでPages設定を有効化:
```bash
curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<owner>/<repo>/pages \
  -d '{"source":{"branch":"main","path":"/docs"}}'
```

公開URLは `https://<owner>.github.io/<repo>/` になる(ユーザー名リポジトリ`<owner>.github.io`の場合はルート直下)。

### 3.5 GitHub Actions (CI/CD) を設定する

`.github/workflows/`配下にYAMLを作成してpushするだけで有効になる。代表的なテンプレートは `references/actions_templates.md` を参照(Node.js / Python / 静的サイトデプロイなど)。

```bash
mkdir -p .github/workflows
# references/actions_templates.md から適切なテンプレートを選び .github/workflows/ci.yml として保存
git add -A && git commit -m "Add CI workflow"
git push
```

push後の実行状況確認:
```bash
gh run list --limit 5
gh run watch   # 直近の実行をリアルタイムで追う
```

### 3.6 Issue管理

```bash
gh issue create --title "タイトル" --body "本文"
gh issue list --state open
gh issue edit <番号> --add-label "bug"
gh issue close <番号>
```

### 3.7 リポジトリ設定(可視性、保護ブランチ等)

```bash
# 可視性変更
gh repo edit <owner>/<repo> --visibility public --accept-visibility-change-consequences

# ブランチ保護(REST API、gh CLIに直接コマンドなし)
curl -s -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<owner>/<repo>/branches/main/protection \
  -d @branch_protection.json
```

`branch_protection.json`の例は`references/api_reference.md`参照。

## 4. 作業ディレクトリの方針

- リポジトリのcloneやファイル構築は `/home/claude/work_<repo-name>/` 配下で行う(ユーザーには見えないスクラッチ領域)。
- ユーザーが成果物としてローカルにもファイルを残したい場合のみ、`/mnt/user-data/outputs/`にコピーし`present_files`で共有する。基本的にはGitHub上に存在すれば足りるため、明示的に頼まれない限り出力ディレクトリへのコピーは不要。

## 5. 進め方の原則

- **多段階の操作はTodoとして整理してから実行する**(例: 「リポジトリ作成→ファイル構築→CI設定→push→Pages公開」のように)。
- 各ステップ実行後、コマンドの結果(成功/失敗、URL、PR番号など)を簡潔にユーザーに報告する。
- 破壊的操作(リポジトリ削除、force push、ブランチ保護解除、可視性をpublicに変更など)は実行前に必ずユーザーに確認する。
- 操作完了後は、関連するURL(リポジトリURL、PR URL、公開URLなど)を明示してユーザーがすぐ開けるようにする。
- エラーが出た場合、トークンのスコープ不足・リポジトリ名の重複・権限不足などよくある原因をまず疑い、ユーザーに分かりやすく伝える。

## 6. 参考資料

- `references/api_reference.md` — REST APIエンドポイント一覧と curl サンプル(単一ファイル編集、ブランチ保護、Pages設定など gh CLIでカバーしきれない操作)
- `references/actions_templates.md` — GitHub Actions workflow のテンプレート集(Node.js、Python、静的サイトデプロイ等)
