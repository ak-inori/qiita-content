---
title: Laravel・Rails・Phoenix 対応表（1/8）パッケージ管理 — Composer / Bundler / Mix
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - Composer
private: true
updated_at: '2026-08-13T13:42:25+09:00'
id: 59ef9ff5937475c59d1b
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

PHP/Laravel、Ruby/Rails の経験者が Elixir/Phoenix に入門するとき（またはその逆）、最初に触るのがパッケージ管理です。本記事では Composer / Bundler / Mix の対応関係を、日常コマンドからバージョン制約の記法、ロックファイルの運用、プライベートパッケージ、モノレポ構成、依存解決アルゴリズムの違いまで掘り下げてまとめます。

本記事は Laravel・Rails・Phoenix 対応表シリーズ（全8回）の第1回です。

対象バージョン（執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.5 | Ruby 4.0（Rails 8.1 の必須要件は 3.2+） | Elixir 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

## 全体の対応関係

まず全体像から。**Elixir では Bundler に相当する別ツールは存在せず、言語標準のビルドツール Mix に依存管理が統合されている**のが最大の違いです。Hex（レジストリのクライアント）は初回に `mix local.hex` で入れますが、日常操作はすべて `mix` サブコマンドとして提供されます。

| | PHP | Ruby | Elixir |
|---|---|---|---|
| パッケージレジストリ | [Packagist](https://packagist.org) | [RubyGems](https://rubygems.org) | [Hex](https://hex.pm) |
| 管理ツール | Composer | Bundler（+ RubyGems） | **Mix**（言語標準に統合） |
| 依存の宣言 | `composer.json` | `Gemfile` | `mix.exs` の `deps/0` |
| ロックファイル | `composer.lock` | `Gemfile.lock` | `mix.lock` |
| dev専用依存 | `require-dev` | `group :development, :test` | `only: [:dev, :test]` オプション |
| オートロード/読込 | `vendor/autoload.php`（PSR-4） | `require` / Zeitwerk（Rails） | コンパイル時に解決 |
| ドキュメントホスティング | 各自（GitHub Pages 等） | rubydoc.info | **[HexDocs](https://hexdocs.pm)（公式・公開時に自動生成）** |

Elixir の依存宣言はこういう形です。設定ファイルではなく Elixir のコードなので、条件分岐やコメントも普通に書けます。

```elixir
# mix.exs
defp deps do
  [
    {:phoenix, "~> 1.8.0"},
    {:req, "~> 0.5"},
    {:credo, "~> 1.7", only: [:dev, :test], runtime: false}
  ]
end
```

## コマンド対応表

| やりたいこと | Composer | Bundler | Mix |
|---|---|---|---|
| 依存をインストール | `composer install` | `bundle install` | `mix deps.get` |
| 依存を追加 | `composer require foo/bar` | `bundle add foo` | `mix.exs` に手書き → `mix deps.get` ※1 |
| 依存を更新 | `composer update foo/bar` | `bundle update foo` | `mix deps.update foo` |
| 全依存を更新 | `composer update` | `bundle update` | `mix deps.update --all` |
| 古い依存の確認 | `composer outdated` | `bundle outdated` | `mix hex.outdated` |
| 脆弱性監査 | `composer audit` | `bundle exec bundler-audit`（gem） | `mix deps.audit`（mix_audit） |
| パッケージ情報 | `composer show foo/bar` | `gem info foo` | `mix hex.info foo` |
| 依存ツリー表示 | `composer show --tree` | `bundle graph`（plugin）※2 | `mix deps.tree` |
| 逆依存の調査 | `composer why foo/bar` | `gem dependency foo --reverse-dependencies` | `mix deps.tree` の目視 |
| グローバルツール導入 | `composer global require` | `gem install` | `mix archive.install hex foo` ※3 |
| ロック準拠で実行 | `vendor/bin/xxx` | `bundle exec xxx` | **不要**（mix は常に lock 準拠） |

※1: Mix には `bundle add` 相当のコアコマンドがありません。`mix hex.info foo` で最新バージョンを確認して `{:foo, "~> 1.0"}` を手書きするのが基本です（後述の Igniter を導入すると `mix igniter.install foo` で追加＋自動設定まで可能）。

※2: かつての `bundle viz` は Bundler 2.2 で本体から切り出され、公式プラグイン [bundler-graph](https://github.com/rubygems/bundler-graph) の `bundle graph` になりました（Graphviz が必要）。

※3: `mix archive.install` はジェネレータ系（`phx_new` など）、`mix escript.install` は CLI ツール系に使います。

```bash
# 例: Phoenix のプロジェクトジェネレータを入れる
mix local.hex --force                       # Hex クライアント（初回のみ）
mix archive.install hex phx_new --force     # gem install rails 相当
```

## バージョン制約の記法比較

3エコシステムとも「セマンティックバージョニング前提で、破壊的変更が入らない範囲を許可する」演算子を持ちますが、**記号と意味の対応が微妙にズレている**のが罠です。

| 意味 | Composer | Bundler / RubyGems | Mix / Hex |
|---|---|---|---|
| `>= 1.2.3` かつ `< 2.0.0` | `^1.2.3` | `~> 1.2`（+ `>= 1.2.3`） | `~> 1.2`（+ `>= 1.2.3`） |
| `>= 1.2.3` かつ `< 1.3.0` | `~1.2.3` | `~> 1.2.3` | `~> 1.2.3` |
| 完全固定 | `1.2.3` | `= 1.2.3` | `== 1.2.3` |
| 範囲指定 | `>=1.2 <2.0`（スペース = AND） | `">= 1.2", "< 2.0"`（複数引数） | `">= 1.2.0 and < 2.0.0"` |
| OR 条件 | `^1.0 \|\| ^2.0` | — | `"~> 1.0 or ~> 2.0"` |

覚え方はこうです。

- **Ruby と Elixir の `~>`（悲観的演算子）は同じ意味**: 「最後に書いた桁だけ動いてよい」。`~> 1.2` はマイナー更新まで許可（`< 2.0`）、`~> 1.2.3` はパッチ更新のみ許可（`< 1.3.0`）。桁数で意味が変わります
- **Composer の `^` は Ruby/Elixir の `~> 1.2` 相当**（メジャー未満を許可）。ただし `^0.3` のような 0.x 系は「0.x は何が起きても仕方ない」という semver の慣習に従い `>= 0.3.0 < 0.4.0` に狭まります
- **Composer の `~` は桁数依存**な点が要注意です。`~1.2.3` は `< 1.3.0` ですが、`~1.2` は `< 2.0.0` まで広がります。「チルダだからパッチのみ」と思い込んでいると、2桁指定でマイナーが上がって驚くことになります

実務では「Composer は `^`、Ruby / Elixir は `~>` + 3桁 or 2桁」に統一しておくのが混乱しない近道です。Hex は `"~> 1.0"` のようにライブラリ側の依存も緩めに書く文化で、`mix hex.info foo` が表示する `{:foo, "~> x.y"}` をそのまま貼るのが定番です。

## ロックファイルと CI での再現性

「宣言ファイルは *許容範囲*、ロックファイルは *実際に使う確定バージョン*」という2層構造は3つとも共通です。事故が起きるのは決まって「install のつもりが update だった」パターンなので、対応を整理します。

| やりたいこと | Composer | Bundler | Mix |
|---|---|---|---|
| lock どおりに入れる | `composer install` | `bundle install` | `mix deps.get` |
| lock を作り直す | `composer update` | `bundle update` | `mix deps.update --all` |
| CI で lock 逸脱を検出 | `composer validate --strict` | `bundle config set frozen true` | `mix deps.get --check-locked` |

- **Composer**: `composer install` は lock があればそれに従いますが、`composer.json` と lock が食い違っていると警告付きで lock を優先します。CI では `composer validate --strict` を先に走らせて「lock が composer.json と同期しているか」を落とすのが定石です。「1個だけ追加したいのに `composer update` を打って全部上がった」事故は、**追加は常に `composer require foo/bar`**（そのパッケージと依存だけを解決）に統一すれば防げます
- **Bundler**: `frozen`（旧 deployment mode）を有効にすると、`Gemfile` と `Gemfile.lock` が一致しない場合に `bundle install` が失敗します。CI では環境変数 `BUNDLE_FROZEN=true` を立てておくのが簡単です
- **Mix**: `mix deps.get` は lock を書き換えることがあります（新しい依存を追加した直後など）。CI では `mix deps.get --check-locked` を使うと、**lock に変更が必要な状態ならエラーで落ちる**ため、「lock のコミット忘れ」を検出できます

もうひとつ Mix 固有の注意点として、**依存パッケージは自分のアプリの `MIX_ENV` に関係なく常に `:prod` 相当でコンパイルされます**。「dev では動くのに依存の debug 用コードが本番で消えている」ような Rails 的な感覚のズレはなく、環境で変えたいのは `only:` オプション（インストールの有無）だけです。

## 依存解決アルゴリズムの違い

普段は意識しませんが、「解決が終わらない」「エラーメッセージが謎」というときに効いてくる部分です。

| | Composer 2 | Bundler | Mix / Hex |
|---|---|---|---|
| 解決アルゴリズム | SAT ソルバ | **PubGrub**（2.4 で Molinillo から移行） | **PubGrub**（hex_solver、Hex 2.0 で移行） |
| 特徴 | 高速・省メモリ（2.0 で大幅改善） | 失敗時の説明が読みやすい | 失敗時の説明が読みやすい |

面白いのは、**Bundler と Hex がまったく同じ結論（PubGrub）に別々にたどり着いている**ことです。PubGrub は Dart の pub 由来のアルゴリズムで、「なぜ解決に失敗したか」を人間が読める形で導出できるのが売りです。Bundler は 2.4 で長年使った Molinillo から移行し、Hex も 2.0 で PubGrub ベースの hex_solver に置き換えました（依存が複雑なプロジェクトで解決が数分〜無限に見えるレベルで固まる問題への対策）。バージョンコンフリクトのエラーが以前より格段に読みやすくなっているのはこのためです。

コンフリクト調査の実践コマンドはこのあたりです。

```bash
# Composer: なぜこのパッケージが入っているのか / なぜこのバージョンに上げられないのか
composer why foo/bar
composer why-not foo/bar 2.0

# Bundler / RubyGems
gem dependency rails --reverse-dependencies
bundle graph   # bundler-graph プラグイン（要 Graphviz）

# Mix
mix deps.tree
mix deps.unlock --check-unused   # lock に残った未使用エントリの検出（CI 向き）
```

## path 依存とモノレポ

社内ライブラリを切り出して同一リポジトリで開発するパターンの対応です。

| | Composer | Bundler | Mix |
|---|---|---|---|
| ローカルパス依存 | `path` リポジトリ | `gem "foo", path: "../foo"` | `{:foo, path: "../foo"}` |
| git 依存 | `vcs` リポジトリ | `gem "foo", github: "org/foo"` | `{:foo, github: "org/foo"}` |
| モノレポ標準機能 | — | — | **umbrella プロジェクト** |

Composer の path リポジトリは `composer.json` に書きます（デフォルトで symlink されるため、編集が即反映されます）。

```json
{
  "repositories": [
    { "type": "path", "url": "../packages/*" }
  ],
  "require": { "acme/billing": "*" }
}
```

Ruby / Elixir はもっと単純で、宣言に `path:` を足すだけです。

```ruby
# Gemfile
gem "billing", path: "engines/billing"
```

```elixir
# mix.exs
{:billing, path: "../billing"}
```

Elixir にはさらに一段上の仕組みとして **umbrella プロジェクト**（`mix new my_app --umbrella`）があります。`apps/` 配下に複数の Mix プロジェクトを並べ、相互依存を `{:billing, in_umbrella: true}` と書くと、ビルド成果物と依存（`deps/`、`_build/`）をルートで共有しつつ、アプリごとに独立してテスト・リリースできます。Rails Engine のように「フレームワークの機能」ではなく、ビルドツールのレイヤーでモノレポがサポートされているのが特徴です。ただし Phoenix コミュニティでは近年「まずは単一アプリ + コンテキスト分割で十分」という揺り戻しもあり、umbrella は「複数の独立したデプロイ単位が本当にあるとき」の選択肢と考えるのがよいです。

## プライベートパッケージ / 社内レジストリ

| 方式 | PHP | Ruby | Elixir |
|---|---|---|---|
| 公式の有償サービス | [Private Packagist](https://packagist.com) | — | **Hex organizations**（hex.pm 内蔵） |
| セルフホスト | Satis（静的リポジトリ生成） | geminabox / gemstash | `mix hex.registry build`（静的レジストリ生成） |
| git リポジトリ直接参照 | `vcs` リポジトリ | `gem "foo", github: "org/foo"` | `{:foo, github: "org/foo"}` |

**Hex は公式レジストリ自体にプライベートパッケージ機能（organizations）が組み込まれている**のが便利なところです。使う側は依存に `organization:` を足すだけです。

```elixir
{:secret, "~> 1.0", organization: "acme"}
```

認証は開発機なら `mix hex.user auth`（所属 organization すべてに自動アクセス）、CI では専用キーを発行して `mix hex.organization auth acme --key <hash>` を使います。公開側は `mix hex.publish --organization acme` です。

git 依存は3つとも書けますが、性質が異なります。

- **Ruby**: `gem install` は git を扱えないため、git 依存は Bundler 経由でのみ使えます
- **Elixir**: git / path 依存のパッケージは **Hex に publish できません**（Hex パッケージの依存は Hex パッケージのみ）。アプリでは自由に使えますが、ライブラリを書くときは制約になります
- **PHP**: `repositories` に `vcs` を並べる方式はリポジトリ数が増えると解決が遅くなるため、規模が出たら Satis / Private Packagist に寄せるのが定番です

## パッケージを公開する側の対応

| | Packagist | RubyGems | Hex |
|---|---|---|---|
| 公開コマンド | GitHub 連携（push で自動更新） | `gem build` → `gem push` | `mix hex.publish` |
| メタデータ | `composer.json` | `foo.gemspec` | `mix.exs` の `package/0` |
| ドキュメント | 各自 | rubydoc.info が自動生成 | **HexDocs へ同時公開** |
| ヤンク/取り下げ | 論理削除 | `gem yank` | `mix hex.retire foo 1.0.0` |

Hex の公開フローはほぼ Mix に統合されています。

```elixir
# mix.exs
def project do
  [
    app: :my_lib,
    version: "0.1.0",
    description: "何をするライブラリか",
    package: [
      licenses: ["MIT"],
      links: %{"GitHub" => "https://github.com/acme/my_lib"}
    ],
    deps: deps()
  ]
end
```

```bash
mix hex.user register   # 初回のみ
mix hex.publish         # パッケージ + ドキュメントを公開
```

`mix hex.publish` は ex_doc で生成した HTML ドキュメントを **hexdocs.pm に同時公開**します。Elixir ライブラリのドキュメントがどれも同じ見た目で hexdocs.pm に揃っているのはこの仕組みのおかげで、「README しかない gem」「docs サイトが野良ホスティングの package」が起きにくいエコシステムになっています。

## Igniter — 「composer require 相当 + 自動設定」の新定番

「依存を足したら config も書き換えてくれる」方向の進化として、[Igniter](https://hexdocs.pm/igniter/) というコード生成・プロジェクトパッチのフレームワークが近年広まっています（Ash Framework 発）。

```bash
# 既存プロジェクトに導入
# mix.exs に {:igniter, "~> 0.6", only: [:dev, :test]} を追加して deps.get 後
mix igniter.install oban        # 依存追加 + そのライブラリのインストーラを自動実行

# 新規プロジェクトを最初から Igniter で
mix archive.install hex igniter_new
mix igniter.new my_app --install ash,oban --with phx.new
```

単に `mix.exs` へ1行足すだけでなく、**ライブラリ側が用意したインストーラ（config 追記、supervision tree への追加など）まで実行してくれる**のがポイントで、Laravel のパッケージが service provider の auto-discovery で「入れたら動く」のに近い体験を後付けしています。対応しているのは Igniter 対応パッケージ（Ash、Oban など増加中）ですが、未対応パッケージでも依存追加だけは代行してくれます。

## まとめ

- **Mix は Composer + Bundler の仕事を言語標準ツールが兼ねる**。`bundle exec` 相当は不要、ドキュメント公開まで統合
- バージョン制約は「Composer は `^`、Ruby/Elixir は `~>`」。**Composer の `~1.2`（2桁）はマイナーまで動く**点だけ要注意
- CI の lock 検証は `composer validate --strict` / `BUNDLE_FROZEN=true` / `mix deps.get --check-locked`
- 依存解決は Bundler と Hex がともに **PubGrub** へ移行済み。コンフリクト時のエラーが読みやすい
- プライベートパッケージは Hex organizations が最も手数が少ない。git/path 依存のライブラリは Hex に publish できない点に注意
- `bundle add` 相当がない Mix の弱点は、**Igniter**（`mix igniter.install`）がむしろ「自動設定込み」で上回りつつある

## 参考リンク

- [Composer — Versions and constraints](https://getcomposer.org/doc/articles/versions.md) / [Repositories](https://getcomposer.org/doc/05-repositories.md)
- [Bundler](https://bundler.io/) / [RubyGems Guides — Patterns（悲観的演算子）](https://guides.rubygems.org/patterns/) / [bundler-graph](https://github.com/rubygems/bundler-graph)
- [Mix.Tasks.Deps.Get](https://hexdocs.pm/mix/Mix.Tasks.Deps.Get.html) / [Hex — Usage](https://hex.pm/docs/usage) / [Hex — Private packages](https://hex.pm/docs/private)
- [Hex v2.0 released with new version solver](https://hex.pm/blog/hex-v20-released-with-new-version-solver) / [hexpm/hex_solver](https://github.com/hexpm/hex_solver)
- [Migrate our resolver engine to PubGrub（rubygems#5960）](https://github.com/ruby/rubygems/pull/5960)
- [Igniter](https://hexdocs.pm/igniter/) / [mix hex.publish](https://hexdocs.pm/hex/Mix.Tasks.Hex.Publish.html)
