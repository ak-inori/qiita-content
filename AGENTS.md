# AGENTS.md

Qiita記事を [qiita-cli](https://github.com/increments/qiita-cli) で管理するリポジトリです。アプリケーションコードはなく、記事のMarkdownファイルとCI設定のみで構成されています。

## リポジトリ構成

```
public/                     # 記事本体（1記事 = 1つの .md ファイル）
qiita.config.json           # qiita-cli の設定（preview 用ホスト/ポートなど）
.github/workflows/publish.yml  # main/master への push で Qiita へ自動公開
```

## 公開フロー

- `main` または `master` に push すると、GitHub Actions（`increments/qiita-cli/actions/publish@v1`）が `public/` 配下の記事を Qiita に公開・更新する
- リポジトリシークレット `QIITA_TOKEN` が必要
- 公開後、CLI が `id` や `updated_at` を書き戻すコミットを自動生成することがある

## 記事ファイルの規約

### フロントマター（必須）

すべての記事ファイルは先頭にYAMLフロントマターが必要。欠けているとCIの検証で失敗する。新規記事の雛形:

```yaml
---
title: 記事タイトル
tags:
  - タグ1
  - タグ2
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
```

- `id` / `updated_at`: 新規記事では `null` / `''` のままにする。公開時にCLIが自動で埋めるため、手動で書き換えない
- `tags`: 1〜5個の配列。Qiita上での検索性を考えて選ぶ
- `private`: `true` = 限定共有記事。**一度 `false`（全体公開）で公開すると `true` には戻せない**（Qiitaの仕様）。逆は可能
- `ignorePublish`: `true` にするとCIが公開対象から除外する。「まだ公開したくない記事」はこれを使う
- `slide`: スライドモード。通常は `false`

### 本文

- 記事タイトルはフロントマターの `title` に書く。**本文先頭にH1（`# タイトル`）を重複して書かない**（Qiita上で二重表示になる）
- 本文の見出しは `##` から始める
- 記事は日本語で書く

## よく使うコマンド

```bash
npx qiita new 記事のファイル名   # 雛形付きで新規記事を作成
npx qiita preview               # http://localhost:8888 でプレビュー
npx qiita publish 記事のファイル名  # 手動公開（通常はCI任せでよい）
npx qiita pull                  # Qiita側の変更をローカルに取り込む
```

## Git運用

- コミットメッセージは日本語で簡潔に書く
- 記事の追加・修正は意味のある単位でコミットする
- push すると即公開される点に注意。公開前の記事は `ignorePublish: true` か `private: true` で制御する
