---
title: Laravel・Rails・Phoenix 対応表 — パッケージ管理からデバッグ、テストライブラリまで
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
private: false
updated_at: '2026-08-09T16:38:49+09:00'
id: 939e1a32483812035057
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

PHP/Laravel、Ruby/Rails の経験者が Elixir/Phoenix に入門するとき（またはその逆）、「あれって Phoenix だと何なんだっけ？」と都度調べることになりがちです。本記事では3つのエコシステムの対応関係を、パッケージ管理・REPL・デバッグ・テスト・定番ライブラリまで網羅的にまとめます。

対象バージョン（執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.3+ | Ruby 3.2+（Rails 8.1 の必須要件、推奨 3.4+） | Elixir 1.18+ / 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

---

## 1. パッケージ管理

まず全体の対応関係から。**Elixir では Bundler に相当する別ツールは存在せず、言語標準のビルドツール Mix に依存管理が統合されている**のが最大の違いです。

| | PHP | Ruby | Elixir |
|---|---|---|---|
| パッケージレジストリ | [Packagist](https://packagist.org) | [RubyGems](https://rubygems.org) | [Hex](https://hex.pm) |
| 管理ツール | Composer | Bundler（+ RubyGems） | **Mix**（言語標準に統合） |
| 依存の宣言 | `composer.json` | `Gemfile` | `mix.exs` の `deps/0` |
| ロックファイル | `composer.lock` | `Gemfile.lock` | `mix.lock` |
| オートロード/読込 | `vendor/autoload.php`（PSR-4） | `require` / Zeitwerk（Rails） | コンパイル時に解決 |

### コマンド対応表

| やりたいこと | Composer | Bundler | Mix |
|---|---|---|---|
| 依存をインストール | `composer install` | `bundle install` | `mix deps.get` |
| 依存を追加 | `composer require foo/bar` | `bundle add foo` | `mix.exs` に手書き → `mix deps.get` ※1 |
| 依存を更新 | `composer update foo/bar` | `bundle update foo` | `mix deps.update foo` |
| 全依存を更新 | `composer update` | `bundle update` | `mix deps.update --all` |
| 古い依存の確認 | `composer outdated` | `bundle outdated` | `mix hex.outdated` |
| 脆弱性監査 | `composer audit` | `bundle exec bundler-audit`（gem） | `mix deps.audit`（mix_audit） |
| パッケージ情報 | `composer show foo/bar` | `gem info foo` | `mix hex.info foo` |
| グローバルツール導入 | `composer global require` | `gem install` | `mix archive.install hex foo` ※2 |
| ロック準拠で実行 | `vendor/bin/xxx` | `bundle exec xxx` | **不要**（mix は常に lock 準拠） |

※1: Mix には `bundle add` 相当のコアコマンドがありません。`mix hex.info foo` で最新バージョンを確認して `{:foo, "~> 1.0"}` を手書きするのが基本です（[Igniter](https://hexdocs.pm/igniter/) を導入すると `mix igniter.install foo` で追加＋自動設定まで可能）。

※2: `mix archive.install` はジェネレータ系（`phx_new` など）、`mix escript.install` は CLI ツール系に使います。

```bash
# 例: Phoenix のプロジェクトジェネレータを入れる
mix local.hex --force                       # Hex クライアント（初回のみ）
mix archive.install hex phx_new --force     # gem install rails 相当
```

---

## 2. プロジェクト作成と日常のコマンド

| やりたいこと | Laravel | Rails | Phoenix |
|---|---|---|---|
| プロジェクト作成 | `laravel new blog` | `rails new blog` | `mix phx.new blog` |
| 開発サーバー起動 | `php artisan serve`（:8000） | `bin/rails server`（:3000） | `mix phx.server`（:4000） |
| ルーティング一覧 | `php artisan route:list` | `bin/rails routes` | `mix phx.routes` |
| DB作成 | 標準コマンドなし（DB は事前作成） | `bin/rails db:create` | `mix ecto.create` |
| マイグレーション | `php artisan migrate` | `bin/rails db:migrate` | `mix ecto.migrate` |
| ロールバック | `php artisan migrate:rollback` | `bin/rails db:rollback` | `mix ecto.rollback` |
| マイグレーション生成 | `php artisan make:migration` | `bin/rails g migration` | `mix ecto.gen.migration` |
| ジェネレータ | `php artisan make:model` | `bin/rails g model` | `mix phx.gen.schema` / `phx.gen.context` / `phx.gen.html` |
| シード投入 | `php artisan db:seed` | `bin/rails db:seed` | `mix run priv/repo/seeds.exs` |
| タスク一覧 | `php artisan list` | `bin/rails -T` | `mix help` |
| タスクのヘルプ | `php artisan help migrate` | — | `mix help ecto.migrate` |

---

## 3. 対話モード（REPL）

| | PHP / Laravel | Ruby / Rails | Elixir |
|---|---|---|---|
| 素の REPL | `php -a` | `irb` | `iex` |
| アプリを読み込んだ REPL | `php artisan tinker`（PsySH） | `bin/rails console`（IRB） | `iex -S mix` |
| サンドボックス | — | `rails console --sandbox`（終了時ロールバック） | — |
| コード再読込 | tinker 再起動 | `reload!` | `recompile()` |
| **サーバーと REPL の同居** | — | — | **`iex -S mix phx.server`** |

Elixir 固有の強みが最後の行です。BEAM（Erlang VM）は軽量プロセスの並行実行が前提なので、**HTTP サーバーを動かしたまま、同じ VM に対話シェルで入る**ことができます。

```elixir
$ iex -S mix phx.server
[info] Running BlogWeb.Endpoint with Bandit 1.8.0 at 0.0.0.0:4000 (http)
iex(1)> Blog.Repo.aggregate(Blog.Accounts.User, :count)  # サーバー稼働中にクエリ
iex(2)> recompile()                                       # コード変更を反映
```

Rails で例えるなら「`rails server` しているプロセスの中で同時に `rails console` が開いている」状態です。後述する pry 系デバッグの前提にもなるので、開発中は基本これで起動しておくのがおすすめです。

---

## 4. インラインデバッグ（実行を止めて対話するやつ）

| | PHP | Ruby | Elixir |
|---|---|---|---|
| ブレークポイントを書く | `eval(\Psy\sh());`（PsySH） | `binding.irb`（標準）<br>`binding.break`（debug gem）<br>`binding.pry`（pry） | `require IEx; IEx.pry()`<br>`dbg()`（`--dbg pry` 時） |
| 前提条件 | psysh が入っていること | それぞれの gem | **`iex` 配下で起動していること** |
| 再開 | `exit` / Ctrl+D | `continue`（debug gem）/ `exit` | `continue` / `respawn` |
| コードを書き換えずに止める | Xdebug（IDE 連携） | `debugger` CLI から `break` | `break! Mod.fun/arity` |
| ステップ実行 | Xdebug | debug gem（`step` / `next`） | なし（pry は対話のみ）※ |

※ Elixir でステップ実行が必要な場合は ElixirLS（VS Code 拡張）のデバッガや `:debugger`（Erlang の GUI）を使います。

### Elixir の使い分け

```elixir
# パターン1: 必ず止める（binding.pry 相当）
def create(conn, params) do
  require IEx; IEx.pry()    # iex -S mix phx.server で起動していないと素通り
  ...
end

# パターン2: dbg() + フラグで止める
# iex --dbg pry -S mix phx.server で起動すると dbg() 到達時に pry に入る
def create(conn, params) do
  params |> dbg()
  ...
end
```

```elixir
# パターン3: コード無変更でブレークポイント（IEx セッションから）
iex> break! BlogWeb.PostController.create/2
# 次にこのアクションが呼ばれた瞬間に停止する
```

---

## 5. プリントデバッグ関数

| | PHP / Laravel | Ruby / Rails | Elixir |
|---|---|---|---|
| 雑に出力 | `var_dump($x)` / `print_r($x)` | `p x` / `pp x` | `IO.inspect(x)` |
| 出力して**実行継続**（値を返す） | `dump($x)` | `p x`（引数を返す）/ `x.tap { pp _1 }` | `IO.inspect(x)` / `dbg(x)`（引数を返す） |
| 出力して**実行停止** | **`dd($x)`** | なし ※1 | なし ※2 |
| ラベル付き | `dump('users:', $users)` | `pp users:` … | `IO.inspect(x, label: "users")` |
| ログに出す | `Log::debug($x)` | `Rails.logger.debug` | `Logger.debug(inspect(x))` |

※1: Ruby には `dd()` 直接対応はなく、`pp x; raise "debug"` などで代用します。
※2: Elixir も同様。「そこで止めたい」は raise ではなく pry 系（前節）で「対話モードに入る」のが流儀です。

### `dbg()` は「Laravel の `dump()` の強化版」

Laravel の `dd()` に一番近そうに見える Elixir の `dbg()` は、実際には **dump して die しない**（実行継続・値を返す）ので `dump()` 側の対応です。そのうえで、パイプラインに挟むと**各ステップの中間値を全部表示**してくれます。

```elixir
[1, 2, 3]
|> Enum.map(&(&1 * 2))
|> Enum.sum()
|> dbg()
```

```text
[lib/blog/demo.ex:5: Blog.Demo.run/0]
[1, 2, 3] #=> [1, 2, 3]
|> Enum.map(&(&1 * 2)) #=> [2, 4, 6]
|> Enum.sum() #=> 12
```

`dump()` を返り値そのままにメソッドチェーンへ挟める、と考えると Laravel 経験者にはしっくりくるはずです。

---

## 6. テスト

| | PHP / Laravel | Ruby / Rails | Elixir |
|---|---|---|---|
| 標準/デファクト | PHPUnit（**Pest** も公式級） | Minitest（Rails 標準）/ **RSpec**（デファクト） | **ExUnit**（言語標準・ほぼ一択） |
| 実行 | `php artisan test` | `bin/rails test` / `bundle exec rspec` | `mix test` |
| ファイル指定 | `php artisan test tests/Feature/FooTest.php` | `rails test test/models/foo_test.rb` | `mix test test/blog/foo_test.exs` |
| 行指定 | `--filter`（メソッド名） | `rspec spec/foo_spec.rb:42` | `mix test test/foo_test.exs:42` |
| 前回失敗したテストの扱い | `--order-by=defects`（失敗分を先に実行） | `rspec --only-failures`（要設定） | **`mix test --failed`（標準装備）** |
| 並列実行 | `php artisan test --parallel` | `parallelize`（Rails標準） | `async: true` を付けた test case を並列実行 |
| カバレッジ | `--coverage`（Xdebug/PCOV） | simplecov | `mix test --cover` / excoveralls |
| watch モード | phpunit-watcher | guard | mix_test_watch |

ExUnit は「言語に最初から入っているテストフレームワークが十分強いので、RSpec/Pest のような対抗馬が育たなかった」タイプです。DSL は Minitest 寄り（`assert` ベース）ですが、`describe` によるグルーピングや setup コールバックなど RSpec 的な構造化もできます。

```elixir
defmodule Blog.AccountsTest do
  use Blog.DataCase, async: true   # ← DB を使う test case も並列実行（SQL Sandbox）

  describe "register_user/1" do
    test "creates a user with valid attrs" do
      assert {:ok, user} = Accounts.register_user(%{email: "a@example.com"})
      assert user.email == "a@example.com"
    end
  end
end
```

`async: true` は Ecto の SQL Sandbox（テストごとにトランザクション分離）と組み合わせることで、**DB を触る test case も安全に並列化**できます。ただし並列になるのは `async: true` を付けた test case 同士で、同じ case 内の test は直列に実行されます。

---

## 7. 定番ライブラリ対応表

### ORM・データベース

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| ORM / データマッパー | Eloquent | ActiveRecord | **Ecto** ※ |
| マイグレーション | 標準 | 標準 | Ecto（標準同梱） |
| ページネーション | 標準（`paginate`） | kaminari / pagy | scrivener_ecto / Flop |
| シード | 標準 | 標準 | `priv/repo/seeds.exs` |

※ Ecto は Active Record パターンではなくデータマッパー + チェンジセット方式です。「モデルに `save` メソッドが生えている」世界観ではなく、`Repo.insert(changeset)` のように操作を明示します。バリデーションはモデルではなく **changeset** に書きます。

### テスト補助

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| ファクトリ | Eloquent Factories（標準） | FactoryBot | ExMachina |
| ダミーデータ | FakerPHP（標準同梱） | faker | faker（elixir 版） |
| モック/スタブ | Mockery / PHPUnit標準 | rspec-mocks / mocha | **Mox** ※ |
| HTTP モック | `Http::fake()` | WebMock / VCR | Bypass / Req.Test |
| E2E / ブラウザ | Laravel Dusk | Capybara + Selenium | Wallaby / PhoenixTest |

※ Mox は「ビヘイビア（インターフェース）を定義してモックする」方針で、モンキーパッチによる任意メソッドの差し替えはできません。設計段階でビヘイビアを切る文化とセットです。

### HTTP クライアント・API

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| HTTP クライアント | Http ファサード（Guzzle） | Faraday / HTTParty | **Req**（推奨）/ Finch / HTTPoison（旧世代） |
| JSON シリアライズ | API Resources | jbuilder / Alba / AMS | JSON ビューモジュール + Jason |
| GraphQL | Lighthouse | graphql-ruby | Absinthe |

### 認証・認可

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| 認証（フルスタック） | Breeze / Fortify | Devise / Rails 8 標準認証ジェネレータ | **`mix phx.gen.auth`（標準）** |
| API トークン認証 | Sanctum | devise-jwt など | Guardian（JWT） |
| 認可 | Gates / Policies（標準） | Pundit / CanCanCan | Bodyguard / LetMe |

### 非同期処理・リアルタイム

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| バックグラウンドジョブ | Queue（Horizon / Redis） | ActiveJob + Sidekiq / Solid Queue | **Oban**（DBベース） |
| 定期実行 | Scheduler（標準） | whenever / sidekiq-cron | Oban.Plugins.Cron / Quantum |
| WebSocket | Reverb + Echo | Action Cable / AnyCable | **Phoenix Channels（標準組み込み）** |
| リアルタイムUI | Livewire | Hotwire（Turbo/Stimulus） | **LiveView（標準）** |
| プレゼンス管理 | — | — | Phoenix.Presence（標準） |

Elixir はここが本領で、Sidekiq が Redis を要求するのに対し **Oban はアプリのデータベースだけで動きます**（PostgreSQL が第一級サポートですが、v2.18 以降は MySQL 8.0+ や SQLite3 にも対応。BEAM の並行性でポーリングが安く済むため）。WebSocket・リアルタイム系も外部ミドルウェアなしでフレームワーク標準です。

### メール・その他

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| メール送信 | Mailable + Mail | Action Mailer | **Swoosh**（標準同梱） |
| 開発時のメール確認 | Mailpit など | letter_opener | Swoosh Local Adapter（`/dev/mailbox` 標準） |
| 画像処理 | Intervention Image | image_processing（libvips） | image（Vix / libvips） |
| i18n | 標準（lang/） | rails-i18n（標準） | Gettext（標準同梱） |
| 環境変数 | .env（標準） | dotenv-rails / credentials | `config/runtime.exs` + dotenvy |
| 監視ダッシュボード | Telescope | rack-mini-profiler 等 | **Phoenix LiveDashboard（標準）** |
| エラートラッキング | Sentry / Bugsnag | Sentry / Bugsnag | Sentry（sentry-elixir） |

### コード品質

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| フォーマッタ | Pint（公式） | RuboCop（-a）/ standard | **`mix format`（言語標準）** |
| リンタ / スタイル | PHP_CodeSniffer | RuboCop | **Credo** |
| 静的解析 / 型 | PHPStan（Larastan）/ Psalm | Sorbet / Steep + RBS | Dialyzer（Dialyxir） |
| pre-commit 一括 | 各自構成 | 各自構成 | `mix precommit` エイリアス（Phoenix 1.8 生成） |

RuboCop 的な立ち位置に一番近いのは **Credo**（コードスメル・スタイル検出）ですが、整形は言語公式の `mix format` が担うため「フォーマット論争がそもそも存在しない」のが Elixir の特徴です。

---

## 8. まとめ: 3つのエコシステムの思想の違い

最後に、対応表からこぼれる「感触」の違いを3点だけ。

**1. Elixir はツールチェーンが一枚岩**

Composer / Bundler + Rake + artisan / rails コマンドという分業が、Elixir では **Mix ひとつ**に集約されています。`bundle exec` 相当も不要（常に lock 準拠）、フォーマッタも言語標準。ツール選定で迷う余地が意図的に消されています。

**2. BEAM の並行性が開発体験にも効いてくる**

`iex -S mix phx.server`（サーバーと REPL の同居）、`async: true` のデフォルト並列テスト、Redis 不要の Oban、外部ミドルウェアなしの Channels / LiveView。ランタイムの性質がそのまま「インフラの部品数が減る」方向に働きます。

**3. Rails / Laravel の「モデル中心」から Ecto の「データ変換中心」へ**

一番書き味が変わるのは ORM です。`$user->save()` / `user.save` の世界から、`changeset` を組み立てて `Repo` に渡す世界へ。最初は冗長に感じますが、「バリデーションがコンテキストごとに違う」問題（登録時とプロフィール更新時で必須項目が違う、など）が自然に解けるようになっています。

3つとも成熟したフルスタックフレームワークなので、対応表を手元に置けば相互の行き来はかなりスムーズです。この記事が「あれって何だっけ」の検索時間を減らせれば幸いです。

---

## 参考リンク

- [Composer](https://getcomposer.org/) / [Bundler](https://bundler.io/) / [Mix (Hexdocs)](https://hexdocs.pm/mix/Mix.html)
- [PsySH](https://psysh.org/) / [IRB](https://github.com/ruby/irb) / [IEx](https://hexdocs.pm/iex/IEx.html)
- [Laravel 13.x ドキュメント](https://laravel.com/docs/13.x) / [Rails ガイド](https://guides.rubyonrails.org/) / [Phoenix ドキュメント](https://hexdocs.pm/phoenix/overview.html)
- [`Kernel.dbg/2`](https://hexdocs.pm/elixir/Kernel.html#dbg/2) / [`IEx.pry/0`](https://hexdocs.pm/iex/IEx.html#pry/0)
- [Ecto](https://hexdocs.pm/ecto/Ecto.html) / [Oban](https://hexdocs.pm/oban/Oban.html) / [Swoosh](https://hexdocs.pm/swoosh/Swoosh.html)
