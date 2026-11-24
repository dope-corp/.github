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

呼び出し側の workflow には trigger・`concurrency`・job の `permissions` を書き、本体を commit SHA で固定して呼びます。版はコメントで併記し、以後は Renovate が更新します。版の tag が指す commit SHA は `gh api repos/dope-corp/.github/commits/<tag> --jq .sha` で得られます。

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

| workflow                     | 呼び出し側の trigger                                               | 呼び出し側の job の `permissions`                  | 呼び出し側の `mise.toml` の `[tools]` に固定するもの |
| ---------------------------- | ------------------------------------------------------------------ | -------------------------------------------------- | ---------------------------------------------------- |
| `reusable-gitleaks.yaml`     | `pull_request` (必要なら `schedule`)                               | `contents: read`                                   | `gitleaks`                                           |
| `reusable-pr-title.yaml`     | `pull_request` の `types: [opened, edited, synchronize, reopened]` | `contents: read`                                   | `prek`                                               |
| `reusable-claude-sweep.yaml` | `schedule` と `workflow_dispatch`                                  | `contents: read`・`actions: read`・`issues: write` | -                                                    |

本体は呼び出し側の `mise.toml` と `mise.lock` でツールを入れるので、上の列のツールを `[tools]` に固定して `mise.lock` に含めます (dope-corp/template から作ったリポジトリでは固定済み)。

共通の入力です。

| 入力           | 内容                                                                                                                                                                                                                                          |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `runs-on`      | job の `runs-on` に渡す値の JSON。既定は `"ubuntu-latest"`。例: `'["self-hosted", "Linux", "X64"]'`。`self-hosted` の label を含むと mise の cache を使わない (ディスクが残るので要らず、大きなツールがあると save / restore で timeout する) |
| `mise-version` | mise 本体の版 (claude-sweep を除く)。空なら、runner や cache に残った版か、mise-action が選ぶ公開から 24 時間たった最新版を使う。self-hosted runner では版がずれないよう指定する                                                              |

self-hosted runner はディスクと状態が job の後も残るので、PR のコード (`mise.toml` や hook) を動かした跡が、同じ runner の後の job に影響しえます。self-hosted runner は、外部の人の PR で workflow が動かず、その runner を使うリポジトリのすべてに同じ人たちが書き込めるときだけ使います。それ以外は GitHub-hosted runner を使います。

check の名前は `<呼び出し側の job の name> / <本体の job の name>` (例: `gitleaks / gitleaks`) になるので、ruleset の required checks にはこの名前で書きます。`.github/jactionlint.yaml` では、`forbidden-uses` の `allow` に `dope-corp/.github/.github/workflows/*` を足し (リポジトリ名だけでは通らない)、`required-checks` には書きません。jactionlint は別リポジトリの reusable workflow の中を確かめられず、書くと `unreachable-required-check` で失敗するためです (jactionlint 2.6.0 で確認)。

`reusable-pr-title.yaml` は、呼び出し側の `.pre-commit-config.yaml` にある commitlint の `commit-msg` hook (`id: commitlint`) で、PR タイトルと、squash merge で付く ` (#<番号>)` を足したタイトルを検査します。hook は dope-corp/template と同じく `language: node` で commitlint を入れる形にします。

`reusable-claude-sweep.yaml` は、`secrets` で `CLAUDE_CODE_OAUTH_TOKEN` を受け取ります。レビューの起点は呼び出し側の workflow の前回成功した run なので、ほかの job を持たない専用の workflow から呼びます。default branch 以外の手動実行ではレビューしません。Claude が使えるのは差分と issue の読み書きに要るコマンドだけで、public リポジトリでは、悪用できる脆弱性の中身を issue に書かず、非公開での確認が要ることだけを起票します。self-hosted runner で動かすときは、runner に `gh` が要ります。

## 変更を公開する

`renovate-preset.json` か `.github/workflows/reusable-*.yaml` を変えた PR を main に merge したら、手元で次を実行して、次の版の tag と GitHub Release を作ります。tag を作るまで、変更は各リポジトリに届きません。

```sh
mise run release
```

タスクは、手元の clone ではなく dope-corp/.github の main と公開済みの tag から次の版を計算し、前の版からの commit を表示して確認を求めます。確認を待つ間に main や最新の版が変わっていたら、何も作らずに止まります。tag だけがあって GitHub Release が無い版 (前回の Release の作成の失敗) があれば、tag を確かめて Release を作るコマンドを表示して止まります。shallow clone では使えません。版は、前の tag 以降に上の 2 種類のファイルを変えた commit の件名と本文 (squash merge なので PR タイトル) のうち、一番大きい変更で決まります。上の 2 種類のファイルが変わっていなければ tag を作りません。

| commit の件名・本文                                 | 版    |
| --------------------------------------------------- | ----- |
| `<type>!: ...` か、本文に `BREAKING CHANGE:` がある | major |
| `feat: ...`                                         | minor |
| それ以外 (`fix: ...` 等)                            | patch |

入力・secret の削除や改名、必須化のように呼び出し側の変更が要るものは、PR タイトルに `!` を付けて major にします。そのため、このリポジトリは squash merge の commit title を PR タイトルにする設定 (`squash_merge_commit_title: PR_TITLE`) にし、merge の画面で件名を書き換えません。issue・PR の雛形と `SECURITY.md` は版を持たず、main の内容がそのまま各リポジトリに効きます。

tag を CI で作らないのは、`GITHUB_TOKEN` が `workflows` 権限を持てず、workflow ファイルを変えた commit への tag の作成を拒否されることがあるためです。手元の `gh` の認証も、`workflows` 権限が無いとして拒否されたときは `workflow` scope (fine-grained PAT では Workflows の権限) を足します。

各リポジトリでは Renovate が新しい版への更新 PR を作り、minor / patch は CI が通れば自動で merge します (このリポジトリの更新は公開から 7 日待たない)。ただし、同じ PR にまとめられた更新に pin・digest の更新が 1 つでも入ると、PR 全体が人手の merge になります。

このリポジトリの workflow は本体を `$/` で呼ぶので、PR の CI が変更後の本体で動きます。

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

## `release`

- **Usage:** `release`

preset と reusable workflow の変更を merge した後に、main に次の版の tag と GitHub Release を作る

## `template-diff`

- **Usage:** `template-diff`

共通ファイルを dope-corp/template の main と比べて差分を表示する
<!-- /mise-tasks -->
<!-- dprint-ignore-end -->
