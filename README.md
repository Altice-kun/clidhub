# Clidhub

Claude AI がGitHubリポジトリの作成・構築・設定・編集・公開を代行するためのスキル(SKILL.md)です。

- 公開ページ: https://altice-kun.github.io/clidhub/
- `.skill`ファイルは公開ページの「Download clidhub.skill」からダウンロードできます。
- 中身は `clidhub/SKILL.md` および `clidhub/references/` 以下のドキュメント一式です。

## できること

- リポジトリの作成・削除・clone・可視性変更
- ファイルのコミット/push、単一ファイルの軽量編集
- ブランチ作成・プルリクエストの作成/マージ
- Issue管理
- GitHub Actions (CI/CD) の設定
- GitHub Pagesでの公開
- Projects (v2) のボード/アイテム操作
- プロフィール・フォロワー・メールアドレス・プランの読み取り
- Gistsの読み取り
- ブランチ保護・コラボレーター招待などのリポジトリ設定

## 必要なもの

GitHub Personal Access Token(Classic: `repo` + `workflow`スコープ、または Fine-grained: Contents / Pull requests / Issues / Actions / Pages の Read and write)。トークンはセッションごとに入力する方式で、永続保存はされません。
