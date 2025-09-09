# cicd-practice

I will learn from the book "GitHub CI/CD Practical Guide."

## 出典元

[GitHub CI/CD 実践ガイド](https://gihyo.jp/book/2024/978-4-297-14173-8)

[サンプルコード](https://github.com/tmknom/example-github-cicd)

## act

[act](https://github.com/nektos/act)

```workflow-error.yml
name: Workflow error

on: push

jobs:
  run:
    run-on: ubuntu-latest # 正しくは runs-on
    steps:
      - run: date
```

```terminal
$ act -j run -W .github/workflows/workflow-error.yml
INFO[0000] Using docker host 'unix:///var/run/docker.sock', and daemon socket 'unix:///var/run/docker.sock'
Error: workflow is not valid. 'workflow-error.yml': Line: 8 Column 5: Failed to match job-factory: Line: 8 Column 5: Unknown Property run-on
Line: 8 Column 5: Failed to match workflow-job: Line: 8 Column 5: Unknown Property run-on
Line: 9 Column 5: Unknown Property steps
Actions YAML Schema Validation Error detected:
For more information, see: https://nektosact.com/usage/schema.html
```

```terminal
act --help

Usage:
  act [event name to run] [flags]
-j, --job string
--workflows'/'-W' flag
```

## actionlint

[actionlint](https://github.com/rhysd/actionlint)

```terminal
actionlint .github/workflows/workflow-error.yml
.github/workflows/workflow-error.yml:6:3: "runs-on" section is missing in job "run" [syntax-check]
  |
6 |   run:
  |   ^~~~
.github/workflows/workflow-error.yml:8:5: unexpected key "run-on" for "job" section. expected one of "concurrency", "container", "continue-on-error", "defaults", "env", "environment", "if", "name", "needs", "outputs", "permissions", "runs-on", "secrets", "services", "steps", "strategy", "timeout-minutes", "uses", "with" [syntax-check]
  |
8 |     run-on: ubuntu-latest
  |     ^~~~~~~
```

## ベストプラクティス

- GitHub Actions のワークフローは常に **ダブルクォート** を用いるべき
  - `- run: echo "ACTOR - ${{ github.actor }}"`
- ハードコードされたブランチはセキュリティリスクがあるため使用しない
- GitHub コンテキストを使うこと
  - デフォルト環境変数の使用は避ける
- GitHub コンテキストを直接シェルコマンドへ埋め込んではいけない
  - 環境変数を用いてコンテキストをクォートすること
- アクションバージョンを **コミット SHA** に固定する
- **Secrets** をログ出力してはならない
- わかりやすいワークフロー実行名/ジョブ名/ステップ名を付与する
  - `run-name` でワークフロー実行名指定する
- `permissions` ジョブを宣言した後、 `timeout-minutes` ジョブを宣言する
  　- 複数の `permissions` が必要な場合は、ジョブを分割する
- 簡単なワークフローの `permissions` は `{}` を用いる
- 各ステップは `- name` レベルキー内のブロック内に配置する
- `GITHUB_OUTPUT` 環境変数を積極的に使う
  - 可読性が高い: ステップ ID を利用するから
  - `GITHUB_ENV` 環境変数はグローバル変数扱いなので、使用を避ける
- `read-all` と `write-all` の一括指定はトラブルシューティング以外では使わない
- タイムアウトを必ず設定する `tiemout-minites: 5`

```yml
# Bad Code
env:
  BRANCH: main # ハードコードされたブランチはセキュリティリスクがあるため使用しない
```

```yml
# Best Practice
- run: echo "${{ github.actor }}" # githubコンテキストの参照
```

```yml
# Best Practice
env:
  ACTOR: ${{ github.actor }} # コンテキストの値を環境変数に代入
steps:
  - run: echo "${ACTOR}" # envコンテキストを参照
```

```yml
# Best Practice
name: Naming # ワークフロー名の指定

on: push

jobs:
  print:
    name: Super Useful Job # ジョブ名の指定
    runs-on: ubuntu-latest
    permissions:
      contents: read
    timeout-minutes: 5
    steps:
      - name: Amazing Script #ステップ名の指定
        run: echo "Hello Naming"
```

```yam
# Best Practice
name: GITHUB_OUTPUT
run-name: GITHUB_OUTPUT sample

on: push

# `permissions` ジョブを先頭で宣言し、
# 続いて `timeout-minutes` ジョブを宣言する
# `- name` レベルキーをトップに宣言し、各ステップをブロックに配置する
jobs:
  share:
    permissions: {}
    timeout-minutes: 5
    runs-on: ubuntu-latest
    steps:
      - name: Initialize Workflow
        run: echo "STEP1 is running."
      - name: Generate Output Value
        id: source # ステップIDを設定

        run: echo "result=Hello" >> "${GITHUB_OUTPUT}" # `GITHUB_OUTPUT` へ書き込み
      - name: Use Output Value
        env:
          RESULT: ${{ steps.source.outputs.result }} # stepsコンテキストを参照
        run: echo "${RESULT}" # 参照値を出力
```

```yml
# ジョブ分割の例
jobs:
  read-only-job:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      - name: Do something with code

  read-write-job:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    needs: read-only-job # 依存関係を設定
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      - name: Commit and push changes
```

## Tips

**GITHUB_TOKEN** の特徴

- ワークフロー開始時に自動生成される
- 終了時自動で破棄される
- 取得方法 `${{ secrets.GITHUB_TOKEN }}`

**アクティビティタイプ** に挙動

- `types` キーで指定する
  - 省略時は全アクティビティを指定すると解釈される
- プルリクエストのみ省略時の解釈がことなる
- `types: [opened, synchronize, reopened]` と解釈される

```yml
on:
  issues: # Issue イベント
    types: [opened, edited] # 作成時と編集時のみ
```

`shell` キーの有無で起動オプションが変わる  
パイプエラーを拾うために `shell: bash` と記述する

```yml
defaults:
  run:
    shell: bash # ワークフローレベルで指定
```

```terminal
// 記述結果の起動オプション
shell: /usr/bin/bash --noprofile --norc -e -o pipefail {0}
```

## CLI

**GitHub CLI** コマンドの使い方

```terminal
# `Secrets` の登録
$ gh secret se PASSWORD --body 'ILoveSecrets!'
```

## ツール

- GitHub Actions ワークフローを TypeScript で記述
  [ghat](https://github.com/koki-develop/ghats)
- ワークフローのバージョンをコミット SHA に置換
  [pinact](https://github.com/suzuki-shunsuke/pinact)
- ワークフロー構文チェック
  [actionlint](https://github.com/rhysd/actionlint)
- ワークフローセキュリティチェック
  [ghalint](https://github.com/suzuki-shunsuke/ghalint/tree/main?tab=readme-ov-file#policies)
- ワークフローセキュリティチェック
  [zizmor](https://github.com/zizmorcore/zizmor)

### ツールの利用

```terminal
$ actionlint
$ ghalint act
$ ghalint run
$ zizmor .github/workflows/TARGET_FAIL
$ pinact run .github/workflows/TARGET_FAIL
$ act --validate
$ act -j JOBS_NAME -W .github/workflows/TARGET_FAIL
```

```terminal
# セキュリティやベストプラクティスに準拠しているかチェック
$ ghalint run
# 記述内容のチェック
$ ghalint act
```
