---
title: 'Laravel・Rails・Phoenix 対応表: プロジェクト作成と日常のコマンド — artisan / rails / mix'
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - artisan
private: false
updated_at: '2026-08-27T13:17:45+09:00'
id: bc7fb3d522c04dd45205
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

PHP/Laravel、Ruby/Rails の経験者が Elixir/Phoenix に入門するとき（またはその逆）、最初に手が止まるのは「`php artisan make:model` って Phoenix だと何？」「`rails db:reset` 相当は？」といったコマンドの対応関係です。本記事では、プロジェクト作成から日常の開発コマンド（サーバー起動・ジェネレータ・マイグレーション・独自コマンドの作り方）までを3エコシステム並べて整理します。

本記事は Laravel・Rails・Phoenix 対応表シリーズの2本目です。

シリーズ一覧:

1. [パッケージ管理 — Composer / Bundler / Mix](https://qiita.com/ak-inori/items/59ef9ff5937475c59d1b)
2. プロジェクト作成と日常のコマンド — artisan / rails / mix **（本記事）**
3. [REPL — tinker / rails console / IEx](https://qiita.com/ak-inori/items/73d50b297a770bf22ec7)
4. [インラインデバッグ — binding.pry と IEx.pry の世界](https://qiita.com/ak-inori/items/0f999ab54f620d829366)
5. [プリントデバッグ — dd() / pp / IO.inspect](https://qiita.com/ak-inori/items/ab069ea3cc797b16e20d)
6. [テスト — PHPUnit・Pest / Minitest・RSpec / ExUnit](https://qiita.com/ak-inori/items/927201acc9e87d164501)
7. [定番ライブラリ — ORM・認証・ジョブ・リアルタイムまで](https://qiita.com/ak-inori/items/b8e0dbfeafc0ead6f4b0)
8. [3つのエコシステムの思想の違い](https://qiita.com/ak-inori/items/e716638c92c3fba4e10e)

対象バージョン（2026年8月執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.5 | Ruby 4.0（Rails 8.1 の必須要件は 3.2+） | Elixir 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

---

## 1. コマンド体系の全体像

個別のコマンドに入る前に、CLI の「入り口」の違いを押さえておきます。

| | Laravel | Rails | Phoenix |
|---|---|---|---|
| 入り口 | `php artisan`（フレームワーク同梱） | `bin/rails`（binstub 経由） | `mix`（**言語標準ツール**） |
| タスク一覧 | `php artisan list` | `bin/rails -T` | `mix help` |
| 個別のヘルプ | `php artisan help migrate` | `bin/rails db:migrate --help` | `mix help ecto.migrate` |
| タスクの実体 | Command クラス | Rake タスク + Rails コマンド | `Mix.Task` モジュール |

Laravel の artisan、Rails の rails/rake はどちらも「フレームワークが提供する CLI」ですが、**Elixir の mix はフレームワーク以前に言語標準のビルドツール**です。Phoenix は `phx.*`、Ecto は `ecto.*` という名前空間のタスクを mix に「追加」しているだけで、依存管理もテストもすべて同じ `mix` から実行します。プロジェクトに依存を足すと、その依存が提供する mix タスクも `mix help` に現れる、という拡張モデルです。

Rails の `bin/rails` はプロジェクト直下の binstub で、Bundler 経由で正しいバージョンの Rails を起動します（`bundle exec rails` 相当）。mix には binstub が不要です。常に `mix.lock` 準拠で動くためです。

## 2. 日常コマンド対応表

まず日常的に叩くコマンドの全体対応です。

| やりたいこと | Laravel | Rails | Phoenix |
|---|---|---|---|
| プロジェクト作成 | `laravel new blog` | `rails new blog` | `mix phx.new blog` |
| 開発サーバー起動 | `php artisan serve`（:8000） | `bin/rails server`（:3000） | `mix phx.server`（:4000） |
| サーバー + REPL | — | — | `iex -S mix phx.server` |
| ルーティング一覧 | `php artisan route:list` | `bin/rails routes` | `mix phx.routes` |
| ルーティング絞り込み | `route:list --path=posts` | `routes -g posts` | `mix phx.routes \| grep posts` |
| DB作成 | 標準コマンドなし ※1 | `bin/rails db:create` | `mix ecto.create` |
| マイグレーション | `php artisan migrate` | `bin/rails db:migrate` | `mix ecto.migrate` |
| ロールバック | `php artisan migrate:rollback` | `bin/rails db:rollback` | `mix ecto.rollback` |
| 適用状況の確認 | `php artisan migrate:status` | `bin/rails db:migrate:status` | `mix ecto.migrations` |
| マイグレーション生成 | `php artisan make:migration` | `bin/rails g migration` | `mix ecto.gen.migration` |
| シード投入 | `php artisan db:seed` | `bin/rails db:seed` | `mix run priv/repo/seeds.exs` |
| ワンオフ実行 | `php artisan tinker --execute="…"` | `bin/rails runner "…"` | `mix run -e "…"` |

※1: Laravel はデフォルトが SQLite で、`laravel new` の時点で `database/database.sqlite` の作成とマイグレーションまで済ませてくれます。MySQL / PostgreSQL を使う場合は `.env` の `DB_*` を書き換えてから `php artisan migrate` を実行します。`rails db:create` / `mix ecto.create` のような DB 作成専用コマンドはありませんが、近年の Laravel は `migrate` 実行時にデータベースが存在しなければ作成を試みます（接続ユーザーに CREATE 権限がない環境では事前作成が必要です）。

ロールバックの「何個戻すか」の指定は三者三様です。

```bash
php artisan migrate:rollback --step=3   # Laravel: 直近3バッチ
bin/rails db:rollback STEP=3            # Rails: 環境変数風の引数
mix ecto.rollback --step 3              # Phoenix: --step / --to <version> / --all
```

シードだけ Phoenix が異質で、**専用タスクではなく「ただのスクリプト実行」**です。`priv/repo/seeds.exs` は普通の Elixir コードで、`mix run` で実行しているだけです。「シードの仕組み」を覚える必要がなく、複数のシードファイルを置いて使い分けるのも自由です。

また Laravel 13 では、`php artisan serve` 単体よりも `composer run dev` が推奨の開発起動になっています。開発サーバー・キューワーカー・Vite を1コマンドでまとめて起動するスクリプトが `composer.json` に生成されるためです。Rails 7 以降の `bin/dev`（Procfile.dev で server + CSS/JS ウォッチャーを起動）と同じ発想です。Phoenix は esbuild / Tailwind のウォッチャーが Endpoint 設定に組み込まれており、`mix phx.server` 一発でアセットビルドまで面倒を見ます。

## 3. プロジェクト作成オプションの比較

「とりあえず全部入り」を生成する点は共通ですが、何を選べるか・どこで選ぶかが違います。

### Laravel: 対話式プロンプトでスターターキットを選ぶ

`laravel new` は基本的にフラグではなく**対話式プロンプト**で構成を決めます。

```bash
laravel new blog
# → スターターキットの選択を促される:
#    None / React / Vue / Svelte / Livewire
#    （React/Vue/Svelte は Inertia ベース、認証は Fortify）
```

- スターターキットには WorkOS AuthKit（ソーシャルログイン / パスキー / SSO）版のバリアントもあり、`laravel new` 中に選択できます
- Packagist 上のコミュニティ製スターターキットは `laravel new blog --using=example/starter-kit` で指定できます
- DB はデフォルト SQLite。`rails new -d postgresql` のような DB 選択フラグで切り替えるのではなく、生成後に `.env` を書き換える方式です

### Rails: フラグの豊富さで構成を削る

`rails new` は逆に**フラグ中心**です。デフォルトが「フルスタック全部入り」で、そこから削っていきます。

```bash
rails new blog -d postgresql          # DB を指定（sqlite3 / mysql / trilogy / postgresql など。デフォルトは sqlite3）
rails new blog --api                  # API モード: ビュー・アセット関連を持たない構成
rails new blog -j esbuild -c tailwind # JS バンドラと CSS の選択（デフォルトは importmap）
rails new blog --minimal              # ほぼ全部 skip した最小構成
rails new blog -m template.rb         # アプリケーションテンプレートで生成をカスタマイズ
```

`--skip-*` 系フラグが非常に多いのが特徴で、Rails 8 では Solid Queue / Solid Cache / Solid Cable や Kamal（デプロイツール）もデフォルト同梱のため、`--skip-solid` `--skip-kamal` のように外せます。`rails new --help` で全量を確認できます。

### Phoenix: `--no-*` フラグで機能単位に外す

`mix phx.new` は Rails に近い「全部入りから削る」方式ですが、フラグが機能単位で素直です。

```bash
mix phx.new blog --database mysql   # postgres（デフォルト）/ mysql / mssql / sqlite3
mix phx.new blog --no-ecto          # DB なし構成（Ecto 関連を一切生成しない）
mix phx.new blog --no-html --no-assets  # API 専用（rails new --api 相当）
mix phx.new blog --no-live          # LiveView なし
mix phx.new blog --binary-id        # 主キーを binary_id（UUID）に
mix phx.new blog --umbrella         # アンブレラ構成（複数 OTP アプリのモノレポ）
mix phx.new blog --adapter cowboy   # HTTP サーバー（デフォルトは bandit）
```

他にも `--no-mailer`（Swoosh を外す）、`--no-gettext`（i18n を外す）、`--no-dashboard`（LiveDashboard を外す）などがあります。`--umbrella` は Rails / Laravel に直接の対応物がない Phoenix 固有の選択肢で、ビジネスロジックと Web 層を別 OTP アプリケーションに分けたモノレポを最初から生成します。

**「API モード」の作り方が思想の違いを表しています**。Rails は `--api` という「モード」を用意し、Laravel はフルスタックのまま API ルートを足す運用が主流（必要なら Sanctum を追加）、Phoenix は `--no-html --no-assets` と「不要な機能を個別に外す」だけでモードという概念がありません。

## 4. ジェネレータの対応

ここが一番「直訳」できないポイントです。まず対応表から。

| 生成したいもの | Laravel | Rails | Phoenix |
|---|---|---|---|
| モデル/スキーマ単体 | `make:model Post` | `g model post title:string` | `phx.gen.schema Blog.Post posts title:string` |
| コントローラ単体 | `make:controller PostController` | `g controller posts index show` | `phx.gen.html` の一部として ※2 |
| バリデーション入力 | `make:request StorePostRequest` | —（モデルに書く） | —（changeset に書く） |
| CRUD 一式（HTML） | `make:model Post --all` | `g scaffold post title:string` | `phx.gen.html Blog Post posts title:string` |
| CRUD 一式（JSON API） | `make:model` + `make:controller --api` | `g scaffold post --api` | `phx.gen.json Blog Post posts title:string` |
| CRUD 一式（リアクティブUI） | —（Livewire 側で生成） | — | `phx.gen.live Blog Post posts title:string` |
| ビジネスロジック層 | —（任意で Service 等） | — | `phx.gen.context Blog Post posts title:string` |
| 認証一式 | スターターキット（`laravel new` 時） | `g authentication`（Rails 8 標準） | `mix phx.gen.auth Accounts User users` |

※2: Phoenix にはコントローラ単体のジェネレータがなく、`phx.gen.html` / `phx.gen.json` がコントローラを含む一式を生成します（手書きするのも数行です）。

### Phoenix のジェネレータは「context」を要求する

`phx.gen.html Blog Post posts` という引数の並びに注目してください。`Blog` が **context**、`Post` がスキーマ、`posts` がテーブル名です。

context は「関連するスキーマ群への公開 API となるモジュール」で、Laravel でいう Service クラス、DDD でいう境界づけられたコンテキストに近い概念です。Phoenix のジェネレータは**コントローラがスキーマ（モデル）を直接触るコードを生成しません**。必ず context を経由します。

```elixir
# phx.gen.html が生成するコントローラ（抜粋）。Repo を直接呼ばない
def index(conn, _params) do
  posts = Blog.list_posts()          # ← context 関数を呼ぶ
  render(conn, :index, posts: posts)
end
```

Rails の scaffold が `Post.all` とモデル直呼びのコントローラを生成するのと対照的で、**「Web 層とドメイン層の分離」をジェネレータのレベルで強制している**のが Phoenix の設計です。ジェネレータの粒度も context 概念に沿って段階的になっています。

- `phx.gen.schema` — スキーマ + マイグレーションだけ（`rails g model` 相当）
- `phx.gen.context` — 上に加えて context モジュール + テスト
- `phx.gen.html` / `phx.gen.json` / `phx.gen.live` — さらにコントローラ（or LiveView）+ テンプレート + テスト

`phx.gen.live` は LiveView（サーバー駆動のリアクティブ UI）で CRUD 一式を生成するもので、「Livewire / Hotwire 込みの scaffold」に相当するものが標準ジェネレータにある、と考えると Laravel / Rails 経験者にはわかりやすいはずです。

### Laravel の `make:model` はフラグで「盛る」

Laravel は逆に、モデルを起点にフラグで生成物を足していきます。

```bash
php artisan make:model Post -m        # + マイグレーション
php artisan make:model Post -mcr      # + マイグレーション + リソースコントローラ
php artisan make:model Post --all     # + factory / seeder / policy / form requests まで全部
```

Rails の `g scaffold` が「全部入りか否か」の二択なのに対し、必要な部品だけ選べるのが artisan 流です。なお `make:request` が生成する FormRequest（リクエスト単位のバリデーションクラス）は Laravel 固有の部品で、Rails ではモデルのバリデーション、Phoenix では context 内の changeset が同じ役割を担うため、対応するジェネレータ自体が存在しません。

## 5. DB をまっさらに戻す（リセット系コマンド）

「開発中に DB を作り直したい」は毎日やる操作なのに、各フレームワークで意味が微妙に違う要注意ポイントです。

| やりたいこと | Laravel | Rails | Phoenix |
|---|---|---|---|
| 全部消して再構築 | `migrate:fresh` | `db:reset` | `mix ecto.reset`（`phx.new` 生成エイリアス）※3 |
| 再構築 + シード | `migrate:fresh --seed` | `db:reset`（seed 込み） | `ecto.reset`（seeds.exs 込み） |
| ロールバックで巻き戻して再適用 | `migrate:refresh` | `db:migrate:reset` ※4 | —（drop して作り直す） |
| 「よしなに」最新化 | — | `db:prepare` | — |

※3: `ecto.reset` は Ecto 本体のタスクではなく、**`phx.new` が `mix.exs` に生成するエイリアス**です。

```elixir
# mix.exs（phx.new が生成）
defp aliases do
  [
    setup: ["deps.get", "ecto.setup", ...],
    "ecto.setup": ["ecto.create", "ecto.migrate", "run priv/repo/seeds.exs"],
    "ecto.reset": ["ecto.drop", "ecto.setup"]
  ]
end
```

エイリアスの中身が見える分、挙動が把握しやすく、プロジェクト固有の手順（例: 検索インデックスの再構築）を足すのも1行です。

※4: `db:reset` と `db:migrate:reset` の違いは Rails 特有です。`db:reset` は **`schema.rb`（スキーマダンプ）からロード**するのに対し、`db:migrate:reset` は**マイグレーションを最初から全部実行**します。Laravel の `migrate:fresh`（全テーブル drop → マイグレーション再実行）と `migrate:refresh`（rollback で巻き戻してから再実行）は、どちらもマイグレーション実行型で、スキーマダンプからのロードは行いません。Ecto も同様にマイグレーション実行型が基本です（`mix ecto.dump` / `ecto.load` で structure.sql 方式も選べます）。

`db:prepare` は Rails 独自の便利タスクで、「DB がなければ create + schema load + seed、あれば migrate だけ」を状況判断してくれます。CI やコンテナ起動時に「とにかくこれを叩けばいい」コマンドとして重宝します。

## 6. 環境の切り替え

| | Laravel | Rails | Phoenix |
|---|---|---|---|
| 環境変数 | `APP_ENV` | `RAILS_ENV` | `MIX_ENV` |
| デフォルト環境 | `local` | `development` | `dev` |
| テスト時 | `testing`（phpunit.xml が設定） | `test`（自動） | `test`（`mix test` が自動設定） |
| 環境ごとの設定 | `.env` + `config/*.php` | `config/environments/*.rb` | `config/{dev,test,prod}.exs` + `runtime.exs` |
| 実行例 | `.env` を切り替え | `RAILS_ENV=production bin/rails c` | `MIX_ENV=prod mix compile` |

**Elixir の `MIX_ENV` だけは「実行時の設定」ではなく「コンパイル時の区別」**である点が最大の違いです。ビルド成果物自体が `_build/dev` / `_build/test` / `_build/prod` と環境ごとに分かれ、`config/config.exs` や `config/prod.exs` の値はコンパイル時に焼き込まれます。「環境変数を読んで実行時に振る舞いを変える」用途には `config/runtime.exs`（起動時に評価される設定ファイル）を使う、という2段構えです。

Laravel の `.env`（実行時に読む）に慣れていると「`config/prod.exs` に書いた値がリリースビルド後に環境変数で変えられない」ことに面食らうので、**動的な値（DB URL、APIキー等）は `runtime.exs`、静的な値はコンパイル時 config** と覚えておくと安全です。Rails の credentials / ENV 併用に近い運用感覚になります。

## 7. ワンオフ実行（REPL を開かずに1行だけ）

cron やデバッグで「アプリのコンテキストでコードを1行だけ実行したい」ときの対応です。

```bash
# Laravel: tinker の --execute オプション
php artisan tinker --execute="echo App\Models\User::count();"

# Rails: runner（ファイルも渡せる。cron タスクの定番）
bin/rails runner 'puts User.count'
bin/rails runner lib/scripts/cleanup.rb
bin/rails runner -e production 'puts User.count'

# Phoenix: mix run（アプリを起動してから式を評価する）
mix run -e 'IO.puts(Blog.Repo.aggregate(Blog.Accounts.User, :count))'
mix run priv/repo/seeds.exs        # ファイル実行（シードはこれの応用）
```

`mix run -e` はデフォルトでアプリケーション（Repo 含む監視ツリー）を起動してから式を評価するので、`rails runner` とほぼ同じ感覚で使えます。アプリ起動が不要なら `--no-start` を付けられます。

なお本番リリース（`mix release` でビルドしたバイナリ）には mix が含まれないため、本番でのワンオフは `bin/blog eval "Blog.Release.migrate()"` のように **`eval` サブコマンド**を使います。「開発は mix、本番は release スクリプト」という使い分けは Phoenix 運用の頻出パターンです。

## 8. 独自コマンドの作り方

定期バッチや運用タスクを CLI コマンドにする方法の対比です。「古いセッションを消す」タスクを3通りで書いてみます。

### Laravel: artisan コマンド（クラス）

```bash
php artisan make:command PruneSessions
```

```php
// app/Console/Commands/PruneSessions.php
class PruneSessions extends Command
{
    protected $signature = 'sessions:prune {--days=30}';
    protected $description = '古いセッションを削除する';

    public function handle(): int
    {
        $cutoff = now()->subDays((int) $this->option('days'));
        $count = DB::table('sessions')
            ->where('last_activity', '<', $cutoff->getTimestamp())
            ->delete();

        $this->info("{$count} 件削除しました");

        return self::SUCCESS;
    }
}
```

`php artisan sessions:prune --days=7` で実行できます。`$signature` に引数・オプション・デフォルト値まで宣言的に書ける DSL が特徴で、`app/Console/Commands` に置くだけで自動登録されます。

### Rails: Rake タスク

```bash
bin/rails g task sessions prune   # lib/tasks/sessions.rake の雛形を生成
```

```ruby
# lib/tasks/sessions.rake
namespace :sessions do
  desc "古いセッションを削除する"
  task :prune, [:days] => :environment do |_t, args|
    days = (args[:days] || 30).to_i
    count = Session.where(updated_at: ...days.days.ago).delete_all
    puts "#{count} 件削除しました"
  end
end
```

`bin/rails sessions:prune[7]` で実行します。ポイントは `=> :environment` で、**これを忘れると Rails アプリ（モデル等）がロードされないまま実行される**という定番の罠があります。引数の `[7]` 記法（zsh では要エスケープ）が不評なこともあり、複雑な引数を取るタスクは `rails runner` + スクリプトで書く流儀もあります。

### Phoenix: Mix タスク

ジェネレータはなく、`lib/mix/tasks/` にモジュールを置きます。

```elixir
# lib/mix/tasks/sessions.prune.ex
defmodule Mix.Tasks.Sessions.Prune do
  use Mix.Task
  import Ecto.Query

  @shortdoc "古いセッションを削除する"
  @requirements ["app.start"]

  @impl Mix.Task
  def run(args) do
    days = args |> List.first("30") |> String.to_integer()
    cutoff = DateTime.add(DateTime.utc_now(), -days, :day)

    {count, _} = Blog.Repo.delete_all(from s in "sessions", where: s.inserted_at < ^cutoff)
    Mix.shell().info("#{count} 件削除しました")
  end
end
```

`mix sessions.prune 7` で実行します。モジュール名 `Mix.Tasks.Sessions.Prune` がそのままタスク名 `sessions.prune` になる規約です。Rake の `=> :environment` に対応するのが **`@requirements ["app.start"]`** で、これを書くとタスク実行前にアプリケーション（Repo 含む）が起動します。書き忘れると Repo が起動しておらずクエリ実行時に落ちる、という罠の構造まで Rails とそっくりです。

前述の通り、Mix タスクは本番リリースには同梱されない点にも注意してください。本番でも実行したいタスクは `lib/blog/release.ex` のような通常モジュールに本体を書き、Mix タスクと `bin/blog eval` の両方から呼ぶ形にしておくのが定石です。

## まとめ

- **入り口の思想**: artisan / rails はフレームワークの CLI、mix は言語標準ツールにフレームワークがタスクを追加する拡張モデル。`bundle exec` や binstub に相当する儀式が mix には不要
- **プロジェクト作成**: Laravel は対話式プロンプト、Rails は豊富なフラグ、Phoenix は `--no-*` で機能単位に削る方式。API 専用構成の作り方（`--api` という「モード」か、機能を外すだけか）に思想が出る
- **ジェネレータ**: Phoenix は context（ドメイン層モジュール）を必ず経由する構成を生成する。`rails g scaffold` の感覚で `phx.gen.html` を叩くと、1層多いコードが出てくるのは仕様であり設計思想
- **リセット系**: `ecto.reset` はただの mix エイリアスで中身が `mix.exs` に見えている。Rails の `db:reset`（スキーマロード）と `db:migrate:reset`（マイグレーション再実行）の違いには注意
- **`MIX_ENV` はコンパイル時の区別**。実行時に変えたい設定は `runtime.exs` へ。ここだけは Laravel / Rails の感覚を持ち込むとハマる

## 参考リンク

- [Laravel 13.x Installation](https://laravel.com/docs/13.x/installation) / [Starter Kits](https://laravel.com/docs/13.x/starter-kits) / [Artisan Console](https://laravel.com/docs/13.x/artisan)
- [Rails ガイド: コマンドラインツール](https://guides.rubyonrails.org/command_line.html) / [Active Record マイグレーション](https://guides.rubyonrails.org/active_record_migrations.html)
- [mix phx.new](https://hexdocs.pm/phoenix/Mix.Tasks.Phx.New.html) / [Contexts](https://hexdocs.pm/phoenix/contexts.html) / [Mix.Task](https://hexdocs.pm/mix/Mix.Task.html)
- [mix ecto.migrate / rollback ほか Ecto SQL タスク](https://hexdocs.pm/ecto_sql/Mix.Tasks.Ecto.Migrate.html)
- [Mix releases（本番でのワンオフ実行）](https://hexdocs.pm/mix/Mix.Tasks.Release.html)

---

次の記事: **[Laravel・Rails・Phoenix 対応表: REPL — tinker / rails console / IEx](https://qiita.com/ak-inori/items/73d50b297a770bf22ec7)**
