---
title: "GitHub ActionsでLightdashのダッシュボード検証をPR単位で自動化する"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Lightdash", "dbt", "GitHubActions", "BIasCode"]
published: true
---

## やりたいこと

Lightdash で Dashboards as Code を運用していると、dbt モデルの変更やダッシュボード YAML の変更でチャートが壊れることがあります。これを main マージ後のデプロイ時ではなく、PR の段階で検知したい。

`lightdash validate` コマンドを使えばチャート・ダッシュボードの整合性を検証できますが、そのまま実行すると本番プロジェクトに対する検証になり、PR で追加した新規ダッシュボードは検証対象に含まれません。

そこで `lightdash start-preview` でプレビュー環境を作り、そこに PR の YAML を upload してから validate することで、**dbt モデルの変更もダッシュボード YAML の変更も含めた完全な検証**を PR 単位で実行できるようにしました。

## 環境

- Lightdash: セルフホスト（Cloud Run）
- Lightdash CLI: v0.2520.2
- dbt Core: GitHub Actions 上で実行
- データウェアハウス: BigQuery
- 認証: IAP + Google OAuth

## 仕組み

PR で dbt 関連のファイルが変更されると、以下のフローが自動実行されます。

1. `dbt compile` で manifest.json を生成
2. `lightdash start-preview --skip-copy-content` で空のプレビュー環境を作成
3. `lightdash upload --project <preview UUID>` で PR ブランチのダッシュボード YAML をプレビューに反映
4. `lightdash validate --preview` でプレビュー環境に対して検証
5. `lightdash stop-preview` でプレビュー環境を削除

ポイントは以下の3つです。

### `--skip-copy-content` で空のプレビューを作る

`start-preview` はデフォルトで本番プロジェクトのチャート・ダッシュボードをプレビューにコピーします。しかしこれだと、PR で削除したダッシュボードがプレビューに残ってしまい、正確な検証ができません。

`--skip-copy-content` を付けることで、空のプレビューに PR の YAML だけを upload し、PR の変更内容のみを正確に検証できます。

### `start-preview` の output から preview UUID を取得する

`lightdash upload` には `--preview` オプションがないため、プレビュー環境にアップロードするにはプロジェクト UUID を明示的に指定する必要があります。

`lightdash start-preview` は CI 環境（`CI=true`）で実行すると、GitHub Actions の step output として `project_uuid` を出力します。これを `steps.<id>.outputs.project_uuid` で取得し、`upload --project` に渡します。


## ワークフロー

```yaml
name: Lightdash Validate

on:
  pull_request:
    paths:
      - "path/to/dbt/project/**"

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  validate:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
      id-token: write
    env:
      LIGHTDASH_URL: https://your-lightdash-instance.example.com
      PREVIEW_NAME: pr-${{ github.event.pull_request.number }}

    defaults:
      run:
        working-directory: path/to/dbt/project

    steps:
      - uses: actions/checkout@v4

      # dbt compile（認証・セットアップは環境に応じて設定）
      - name: Compile dbt project
        run: dbt compile

      # Lightdash CLI のセットアップ
      - name: Install Lightdash CLI
        run: npm install -g @lightdash/cli@0.2520.2

      - name: Lightdash login
        run: lightdash login --token "${{ secrets.LIGHTDASH_API_TOKEN }}" "${{ env.LIGHTDASH_URL }}"
        env:
          CI: true

      # プレビュー環境を作成（本番コンテンツはコピーしない）
      - id: preview
        name: Create Lightdash preview
        run: lightdash start-preview --name "${{ env.PREVIEW_NAME }}" --skip-dbt-compile --skip-copy-content
        env:
          CI: true

      # PR のダッシュボード YAML をプレビュー環境に反映
      - name: Upload dashboards to preview
        run: lightdash upload --force --path lightdash --project "${{ steps.preview.outputs.project_uuid }}"
        env:
          CI: true

      # プレビュー環境に対して検証
      - name: Validate Lightdash preview
        run: lightdash validate --preview --skip-dbt-compile        env:
          CI: true

      # プレビュー環境を削除（成功・失敗問わず）
      - name: Stop Lightdash preview
        if: always()
        run: lightdash stop-preview --name "${{ env.PREVIEW_NAME }}"
        env:
          CI: true
```

## 検知できるもの

| 変更内容 | 検知 |
| --- | --- |
| dbt モデルのカラム削除・リネーム → チャートが壊れる | ✅ |
| ダッシュボード YAML の fieldId 間違い | ✅ |
| 新規ダッシュボードの定義エラー | ✅ |
| フィルター・ソートの参照先が存在しない | ✅ |

## 実行時間

実際の運用では以下の実行時間でした。

- `dbt compile`: 約2分（プロジェクト規模による）
- `start-preview`: 約30秒
- `upload`: 約10秒
- `validate`: 約20秒
- `stop-preview`: 約3秒

validate 自体は20秒程度なので、dbt の変更がある PR すべてで実行してもコスト的に問題ありません。

## まとめ

`lightdash start-preview` + `lightdash upload` + `lightdash validate --preview` の組み合わせにより、PR 単位でダッシュボードの完全な検証ができるようになりました。

BI as Code で重要なのは、ダッシュボードの定義をコード管理するだけでなく、**ソフトウェアエンジニアリングと同じように CI で検証する**ことだと思います。
