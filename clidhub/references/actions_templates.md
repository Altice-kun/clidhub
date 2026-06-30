# GitHub Actions ワークフローテンプレート集 (Clidhub用)

`.github/workflows/<name>.yml` として保存する。プレースホルダー(`<...>`)は実際の値に置き換える。

## Node.js プロジェクトのCI

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [20.x]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm run build --if-present
      - run: npm test --if-present
```

## Python プロジェクトのCI

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt
      - run: pytest
```

## 静的サイトをGitHub Pagesに自動デプロイ(Actions経由)

`/docs`方式ではなく、ビルド成果物をActionsでデプロイしたい場合に使う。
事前にリポジトリ設定でPagesのソースを「GitHub Actions」に変更する必要がある:
```bash
curl -s -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/pages \
  -d '{"build_type":"workflow"}'
```

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [ main ]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: '.'   # 公開したいディレクトリ(ビルド成果物があるなら ./dist 等に変更)
      - id: deployment
        uses: actions/deploy-pages@v4
```

## Dockerイメージのビルド & GHCRへのpush

```yaml
name: Build and Push Docker Image
on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

## Lint専用ワークフロー(汎用)

```yaml
name: Lint
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npx eslint . --max-warnings=0
```

## 定期実行(スケジュール)タスク

```yaml
name: Scheduled Task
on:
  schedule:
    - cron: '0 0 * * *'   # 毎日0時(UTC)に実行
  workflow_dispatch: {}    # 手動実行も可能にする

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "ここに実行したい処理を書く"
```

## カスタムSecretsを使う場合

ワークフロー内で `${{ secrets.MY_SECRET_NAME }}` のように参照する。登録は事前に:
```bash
gh secret set MY_SECRET_NAME --body "value" --repo <owner>/<repo>
```
