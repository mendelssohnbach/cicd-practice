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
