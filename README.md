# Zenn Contents

[Zenn](https://zenn.dev/ota2000) の記事を管理するリポジトリです。

## セットアップ

```bash
pnpm install
```

## コマンド

| コマンド | 説明 |
| --- | --- |
| `pnpm preview` | ローカルプレビューを起動 |
| `pnpm new:article` | 新しい記事を作成 |
| `pnpm new:book` | 新しい本を作成 |
| `pnpm lint` | textlint で記事を検査 |
| `pnpm lint:fix` | textlint で自動修正 |
| `pnpm lint:md` | markdownlint で記事を検査 |
| `pnpm lint:md:fix` | markdownlint で自動修正 |

## Lint

文章品質と Markdown フォーマットを 2 つのツールでチェックしています。

### textlint

日本語テクニカルライティング向けの文章校正ツールです。

| ルール | チェック内容 |
| --- | --- |
| preset-ja-technical-writing | 一文の長さ、句読点、冗長表現など基本ルール集 |
| preset-jtf-style | JTF 日本語標準スタイルガイド準拠 |
| @textlint-ja/preset-ai-writing | AI 生成文章の特徴的な表現を検出 |
| @textlint-ja/no-synonyms | 同義語の表記揺れを検出 |
| ja-hiragana-keishikimeishi | 形式名詞のひらがな表記（こと、もの、ところ等） |
| ja-hiragana-fukushi | 副詞のひらがな表記（すべて、ほとんど等） |
| ja-hiragana-hojodoushi | 補助動詞のひらがな表記（ください、いただく等） |
| no-hoso-kinshi-yogo | 放送禁止用語の検出 |
| no-dead-link | リンク切れの検出 |
| ja-unnatural-alphabet | 不自然な半角アルファベットを検出 |
| period-in-list-item | リスト項目末尾の句点を統一 |
| @textlint-rule/no-unmatched-pair | 括弧の閉じ忘れを検出 |

### markdownlint

Markdown の構文・フォーマットをチェックするツールです。Zenn 固有の記法（bare URL のカード埋め込み、`:::` コンテナ記法等）に合わせてルールを調整しています。

設定は [.markdownlint-cli2.jsonc](.markdownlint-cli2.jsonc) を参照してください。

## 参考

- [Zenn CLI Guide](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Markdown Guide](https://zenn.dev/zenn/articles/markdown-guide)
