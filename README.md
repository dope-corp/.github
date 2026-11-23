# .github

[![CI](https://github.com/dope-corp/.github/actions/workflows/ci.yaml/badge.svg?event=pull_request)](https://github.com/dope-corp/.github/actions/workflows/ci.yaml)

dope-corp のリポジトリが共通で使う設定です。各リポジトリは、ここの Renovate preset と reusable workflow を版を固定して参照し、Renovate の更新 PR で新しい版を取り込みます。issue・PR の雛形と `SECURITY.md` は、organization の既定として自動で適用されます。

| ファイル                                                                               | 内容                                                                                                |
| -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `renovate-preset.json`                                                                 | Renovate の共通設定                                                                                 |
| `.github/workflows/reusable-gitleaks.yaml`                                             | 履歴全体の secret スキャン                                                                          |
| `.github/workflows/reusable-pr-title.yaml`                                             | PR タイトル (と squash merge の commit title) を commitlint で検査する                              |
| `.github/workflows/reusable-claude-sweep.yaml`                                         | 前回成功した run 以降の差分を Claude Code がレビューし、指摘を `claude-sweep` label の issue にする |
| `.github/ISSUE_TEMPLATE/` / `.github/PULL_REQUEST_TEMPLATE.md` / `.github/SECURITY.md` | 自前のファイルを持たないリポジトリ (private を含む) に適用される organization の既定                |

新しいリポジトリは [dope-corp/template](https://github.com/dope-corp/template) から作り、下の「使い方」に従ってここを参照します。

## 使い方

### Renovate preset

`renovate.json` で preset を tag で固定して参照し、リポジトリ固有の設定だけを足します。Renovate が tag の更新 PR を作ります。

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>dope-corp/.github:renovate-preset#v1.0.0"],
  "packageRules": []
}
```

### reusable workflow

呼び出し側の workflow に trigger・`concurrency`・job の `permissions` を書き、本体を commit SHA で固定して呼びます (版はコメントで併記し、Renovate が更新します)。check の名前は `<呼び出し側の job の name> / <本体の job の name>` になるので、ruleset の required checks と `.github/jactionlint.yaml` の `required-checks` はこの名前で書きます。`.github/jactionlint.yaml` の `forbidden-uses` の `allow` には `dope-corp/.github` を足します。

共通の入力です。

| 入力           | 内容                                                                                                                                                                                                                                 |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `runs-on`      | job の `runs-on` に渡す値の JSON。既定は `"ubuntu-latest"`。例: `'["self-hosted", "Linux", "X64"]'`。`self-hosted` を含むと mise の cache を使わない (ディスクが残るので要らず、大きなツールがあると save / restore で timeout する) |
| `mise-version` | mise 本体の版 (claude-sweep を除く)。空なら最新版。self-hosted runner では版がずれないよう指定する                                                                                                                                   |

```yaml
# .github/workflows/gitleaks.yaml
name: gitleaks
on:
  pull_request:
    types: [opened, synchronize, reopened]
permissions: {}
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  gitleaks:
    name: gitleaks
    permissions:
      contents: read
    uses: dope-corp/.github/.github/workflows/reusable-gitleaks.yaml@<commit SHA> # v1.0.0
    with:
      runs-on: '["self-hosted", "Linux", "X64"]'
      # renovate: depName=jdx/mise
      mise-version: 2026.11.8
```

`reusable-pr-title.yaml` は、`pull_request` の `types: [opened, edited, synchronize, reopened]` で呼びます。呼び出し側の `.pre-commit-config.yaml` にある commitlint の `commit-msg` hook (`id: commitlint`) で検査するので、hook は `language: node` で commitlint を入れる形にします (テンプレートと同じ)。

`reusable-claude-sweep.yaml` は `schedule` と `workflow_dispatch` で呼びます。job の `permissions` に `contents: read`・`actions: read`・`issues: write` を許可し、`secrets` で `CLAUDE_CODE_OAUTH_TOKEN` を渡します。レビューの起点は、呼び出し側の workflow の前回成功した run です。

## 変更を公開する

`renovate-preset.json` か `.github/workflows/reusable-*.yaml` を変えた PR を main に merge すると、`release.yaml` が次の版の tag と GitHub Release を作ります。版は main の commit の件名 (squash merge なので PR タイトル) で決まります。

| PR タイトル・commit の本文                          | 版    |
| --------------------------------------------------- | ----- |
| `<type>!: ...` か、本文に `BREAKING CHANGE:` がある | major |
| `feat: ...`                                         | minor |
| それ以外 (`fix: ...` 等)                            | patch |

入力・secret の削除や改名、必須化のように呼び出し側の変更が要るものは、PR タイトルに `!` を付けて major にします。上の 2 種類以外のファイル (README 等) だけを変えた merge では tag を作りません。issue・PR の雛形と `SECURITY.md` は版を持たず、main の内容がそのまま各リポジトリに効きます。

各リポジトリでは Renovate が新しい版への更新 PR を作り、minor / patch は CI が通れば自動で merge します (このリポジトリの更新は公開から 7 日待たない)。

このリポジトリの workflow は本体を `$/` で呼ぶので、PR の CI が変更後の本体で動きます。

### release の GitHub App (初回だけ)

`GITHUB_TOKEN` は `workflows` 権限を持てず、workflow ファイルを変えた commit への tag の作成を拒否されることがあるので、tag は GitHub App の token で作ります。

1. organization の **Settings → Developer settings → GitHub Apps → New GitHub App** で App を作る。Webhook は無効にし、Repository permissions は **Contents: Read and write** と **Workflows: Read and write** だけにする。
2. App を dope-corp/.github だけにインストールし、private key を発行する。
3. このリポジトリに environment `release` を作り、Deployment branches を `main` に限る。environment の variable `RELEASE_APP_CLIENT_ID` に App の Client ID を、secret `RELEASE_APP_PRIVATE_KEY` に private key を設定する。

## このリポジトリの開発

ツール・git hook・CI の構成はテンプレートと同じです。セットアップと検査は [dope-corp/template の README](https://github.com/dope-corp/template#readme) を参照してください。

## タスク

`mise run <task>` で実行します。一覧は `mise.toml` の `[tasks]` から hook が生成します。

<!-- dprint-ignore-start -->
<!-- mise-tasks -->
## `docs`

- **Usage:** `docs`

README.md のタスク一覧を mise.toml から生成し直す

## `pre-commit`

- **Usage:** `pre-commit`

全ファイルに pre-commit の hook を実行する (CI の prek job のうち online の検査を除く部分)

## `template-diff`

- **Usage:** `template-diff`

共通ファイルを dope-corp/template の main と比べて差分を表示する
<!-- /mise-tasks -->
<!-- dprint-ignore-end -->
