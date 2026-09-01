---
title: 'Laravel・Rails・Phoenix 対応表: REPL — tinker / rails console / IEx'
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - REPL
private: false
updated_at: '2026-08-27T02:20:47+09:00'
id: 73d50b297a770bf22ec7
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

フレームワークを行き来するとき、真っ先に手に馴染ませたいのが対話モード（REPL）です。「Laravel の `tinker` に相当するものは Phoenix だと何？」「`reload!` は？」「サンドボックスモードは？」——本記事では PHP/Laravel の tinker（PsySH）、Ruby/Rails の rails console（IRB）、Elixir の IEx を、基本操作から設定ファイル、リモート接続、シェル固有の操作体系まで対応付けて解説します。

本記事は Laravel・Rails・Phoenix 対応表シリーズの3本目です。

シリーズ一覧:

1. [パッケージ管理 — Composer / Bundler / Mix](https://qiita.com/ak-inori/items/59ef9ff5937475c59d1b)
2. [プロジェクト作成と日常のコマンド — artisan / rails / mix](https://qiita.com/ak-inori/items/bc7fb3d522c04dd45205)
3. REPL — tinker / rails console / IEx**（本記事）**
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

## 基本対応表

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 素の REPL | `php -a` | `irb` | `iex` |
| アプリを読み込んだ REPL | `php artisan tinker`（PsySH） | `bin/rails console`（IRB） | `iex -S mix` |
| REPL の実体 | [PsySH](https://psysh.org/) | [IRB](https://github.com/ruby/irb) | [IEx](https://hexdocs.pm/iex/IEx.html)（言語標準） |
| サンドボックス | — | `rails console --sandbox` | —（トランザクションで代用） |
| コード再読込 | tinker 再起動 | `reload!` | `recompile()` |
| **サーバーと REPL の同居** | — | — | **`iex -S mix phx.server`** |

3者の立ち位置には微妙な違いがあります。**IEx は言語に組み込まれた標準 REPL で、`iex -S mix` は「Mix プロジェクトを読み込んで IEx を起動する」だけの汎用コマンド**です。Phoenix 専用の console コマンドは存在せず、必要ありません。一方 tinker は「PsySH というサードパーティ REPL を Laravel が同梱している」構図、rails console は「Ruby 標準の IRB に Rails がアプリ読み込みとヘルパーを足している」構図です。

## それぞれの REPL でできること

### tinker（PsySH）

Eloquent モデル、ジョブ、イベントなどアプリ全体を対話的に触れます。1点だけ有名な落とし穴があるので押さえておきましょう。

```php
> App\Models\User::count()
= 42

> Bus::dispatch(new App\Jobs\SendReport)   // ジョブ投入は dispatch() ではなくこちら
```

**tinker 内では `dispatch()` ヘルパーが正しく動きません。** `dispatch()` はガベージコレクションのタイミングでジョブをキューに積む仕組みのため、REPL では `Bus::dispatch` か `Queue::push` を使うのが公式の案内です。また、tinker 内で実行できる Artisan コマンドは許可リスト方式で、デフォルトでは `migrate` や `up` / `down` など8個だけです（後述の `tinker.php` で追加可能）。

### rails console（IRB）

Rails 側の武器は `app` と `helper` という2つのオブジェクトです。

```ruby
irb(main)> app.root_path                 # ルーティングヘルパー
=> "/"
irb(main)> app.get "/posts"              # サーバーを立てずにリクエストを発行
irb(main)> app.response.status
=> 200
irb(main)> helper.time_ago_in_words(3.days.ago)
=> "3 days"
```

`app` は結合テストで使われるものと同じセッションオブジェクトなので、**console の中だけで HTTP リクエスト〜レスポンス検証まで完結**できます。環境の切り替えは `bin/rails console -e production` のように `-e` で行います。

### iex -S mix

Elixir では `iex -S mix` がアプリを読み込んだ REPL です。素の `iex` との違いは「Mix プロジェクトをコンパイル・起動してからシェルに入る」ことだけで、追加の魔法はありません。

```elixir
iex(1)> Blog.Repo.aggregate(Blog.Accounts.User, :count)
5
iex(2)> Blog.Accounts.get_user!(1) |> Map.get(:email)
"a@example.com"
```

注意点として、Rails/Laravel のようなグローバルなヘルパー関数はなく、`Blog.Accounts.list_users()` のように**常にモジュール名から書きます**。毎回フルパスを打つのは面倒なので、次節の `.iex.exs` でエイリアスを仕込むのが定番です。

## 設定ファイル: .irbrc / .iex.exs / config.php

各 REPL は起動時に設定ファイルを読み込みます。探索場所の対応は次のとおりです。

| | PsySH | IRB | IEx |
|---|---|---|---|
| ユーザー設定 | `~/.config/psysh/config.php` | `$XDG_CONFIG_HOME/irb/irbrc` または `~/.irbrc` | `~/.iex.exs`（`IEX_HOME` 配下） |
| プロジェクト設定 | `.psysh.php`（プロジェクト直下） | カレントの `.irbrc` | カレントの `.iex.exs` |
| 環境変数で指定 | — | `$IRBRC` | `IEX_HOME` |
| tinker/Rails 固有 | `config/tinker.php` | — | — |

読み込みの挙動に重要な違いがあります。**IEx は「カレントの `.iex.exs` → `IEX_HOME` 配下（既定は `~`）の `.iex.exs`」の順に探し、最初に見つかった1つだけを評価します**（両方は読まれません。`IEX_HOME` はファイルではなく、グローバルの `.iex.exs` を探すディレクトリの指定です）。IRB は探索順（`$IRBRC` → XDG → `~/.irbrc` → カレント）で最初の1つを読み、PsySH はユーザー設定に加えてプロジェクトの `.psysh.php` も読みます（信頼していないプロジェクトでは制限モードになりローカル設定をスキップ）。

### プロジェクト用 .iex.exs の定番パターン

Phoenix 開発では、プロジェクト直下に `.iex.exs` を置いてエイリアスと import を仕込むのがほぼ必須のイディオムです。

```elixir
# .iex.exs（プロジェクト直下・チームで共有するならコミットする）
alias Blog.{Repo, Accounts, Posts}
alias Blog.Accounts.User
alias Blog.Posts.Post

import Ecto.Query

# よく使うワンライナーを関数として置いておくのも定番
defmodule H do
  def last(schema), do: Blog.Repo.one(from x in schema, order_by: [desc: x.id], limit: 1)
end
```

```elixir
$ iex -S mix
iex(1)> Repo.all(from u in User, select: u.email)   # フルパス不要になる
iex(2)> H.last(Post)
```

ファイルの中身は「シェルに1行ずつ打ち込んだのと同じ」扱いで評価されるので、シェルで使える書き方はすべて使えます。

### IRB / PsySH 側の対応物

IRB では `.irbrc` に `IRB.conf` を書きます。

```ruby
# ~/.irbrc
IRB.conf[:SAVE_HISTORY] = 10_000
IRB.conf[:COMMAND_ALIASES][:sl] = :show_source   # コマンドに独自エイリアス
```

PsySH では `defaultIncludes` オプションで「セッション開始時に必ず読み込むファイル」を指定でき、オートローダーや自作ヘルパーの読み込みに使えます。tinker 固有の設定（実行を許可する Artisan コマンドの `commands` 配列、クラスの自動エイリアスを抑止する `dont_alias` 配列）は `php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"` で生成される `config/tinker.php` に書きます。

## 便利ヘルパー対応表

3つとも「ドキュメント参照」「ソース表示」「オブジェクト調査」が REPL 内で完結します。

| やりたいこと | PsySH (tinker) | IRB (rails console) | IEx |
|---|---|---|---|
| ドキュメントを読む | `doc strlen` | `show_doc Array#sum` | `h Enum.map/2` |
| ソースコードを見る | `show User::create` | `show_source User#save`（別名 `$`） | `open Enum.map/2`（エディタで開く） |
| オブジェクト/モジュールの調査 | `ls $user` | `ls user` | `i user` / `exports(Enum)` |
| 現在地の確認 | `whereami` | `whereami`（別名 `@`） | `whereami`（pry 中のみ） |
| 直近の例外を見る | `wtf` | —（`$!` で参照） | —（そもそも例外でシェルは死なない） |
| 過去の評価値を再利用 | `$_`（直前のみ） | `_` | `v(3)`（履歴 n 番目の値） |
| 履歴 | `history`（検索・再実行可） | `history` | 上矢印（後述の設定が必要） |
| 再コンパイル/再読込 | — | `reload!` | `r Blog.Accounts` / `recompile()` |

IEx のヘルパーでとくに便利なのが次の4つです。

- **`h/1`** — hexdocs を引かずにドキュメントを表示。`h Enum.reduce/3` のように arity まで指定可能
- **`i/1`** — 任意の値の型・実装プロトコル・参照モジュールを表示。「これ何？」に一発で答える
- **`v/1`** — `v(3)` で3番目の評価結果を取得。`v()` は直前の値。「さっきの結果を変数に入れ忘れた」を救済
- **`r/1` / `recompile/0`** — `r Blog.Accounts` は指定モジュールだけ再コンパイル、`recompile()` はプロジェクト全体の変更を反映

IRB 側も近年のバージョンで `show_source`（`$`）、`ls`、`whereami`（`@`）が揃い、PsySH の使い勝手にかなり近づいています。`ls` はどちらも「そのオブジェクトに生えているメソッド・変数の一覧」を出すもので、探索的にコードを触るときの主力です。

## コード再読込の思想差

| | tinker | rails console | IEx |
|---|---|---|---|
| 再読込コマンド | なし（再起動） | `reload!` | `recompile()` / `r Mod` |
| 理由 | PHP はクラスの再宣言不可 | Zeitwerk が再ロード | BEAM のホットコードローディング |

**tinker にはコード再読込コマンドがありません。** PHP は同名クラスの再定義ができないため、コードを変更したら tinker を再起動するのが基本です（この不便を解消する商用ツールとして Tinkerwell が公式ドキュメントでも紹介されています）。Rails の `reload!` は Zeitwerk のオートロード機構による再読み込み、Elixir の `recompile()` は BEAM が元々持つホットコードローディング機能の上に乗っています。

## iex -S mix phx.server — サーバーと REPL の同居

Elixir 固有にして最大の売りがこれです。

```elixir
$ iex -S mix phx.server
[info] Running BlogWeb.Endpoint with Bandit 1.8.0 at 0.0.0.0:4000 (http)
iex(1)> Blog.Repo.aggregate(Blog.Accounts.User, :count)   # サーバー稼働中にクエリ
5
iex(2)> recompile()                                        # コード変更を即反映
:ok
```

なぜこれが可能かというと、BEAM（Erlang VM）では **HTTP サーバーも IEx シェルも、同じ VM 内の「軽量プロセス」のひとつにすぎない**からです。Phoenix のリクエスト処理は1リクエスト = 1プロセスで並行実行されており、そこにシェル用のプロセスが1つ増えても互いに干渉しません。プロセス間はメモリを共有せず、メッセージパッシングだけでやり取りするため、「シェルからサーバーの状態を覗く・変更する」ことが安全にできます。

Rails で例えるなら「`rails server` しているまさにそのプロセスの中で `rails console` が開いている」状態です。これができると:

- ブラウザでリクエストを送った直後に、シェルからその結果（DB の状態やプロセスの状態）を確認できる
- `IEx.pry()` を仕込んだコードにリクエストが到達すると、**このシェルがそのままデバッガになる**（インラインデバッグ編で詳述）
- `recompile()` で再起動なしにコード変更を反映できる

開発中は `mix phx.server` 単体ではなく、常に `iex -S mix phx.server` で起動しておくのがおすすめです。

## サンドボックスモード

**3者の中でサンドボックスを標準装備しているのは Rails だけ**です。

```bash
$ bin/rails console --sandbox
Loading development environment in sandbox (Rails 8.1.3)
Any modifications you make will be rolled back on exit
```

すべての DB 操作がトランザクションに包まれ、終了時にロールバックされます。「本番データで破壊的な操作を試したい」ときの安全網です。さらに Rails 7.1 以降は `config.sandbox_by_default = true` を設定すると **console がデフォルトでサンドボックス起動**になります（development / test 環境では無視され、通常モードには `--no-sandbox` が必要）。本番 console の事故防止に有効です。

Elixir に `--sandbox` 相当のフラグはありませんが、Ecto のトランザクションで同じことができます。

```elixir
iex(1)> Blog.Repo.transaction(fn ->
...(1)>   Blog.Repo.delete_all(Blog.Posts.Post)   # 試したい破壊的操作
...(1)>   Blog.Repo.aggregate(Blog.Posts.Post, :count) |> IO.inspect()
...(1)>   Blog.Repo.rollback(:sandbox)             # 明示的にロールバック
...(1)> end)
0
{:error, :sandbox}
```

なお、テストで使われる `Ecto.Adapters.SQL.Sandbox` はこの発想をコネクションプールのレベルで仕組み化したもので、「テストごとにトランザクションを張り、終了時にロールバックする」ことで `async: true` の並列テストを可能にしています。名前は同じ「サンドボックス」ですが、console 用ではなくテスト基盤である点に注意してください。tinker には相当機能がないため、`DB::beginTransaction()` を手で張るのが代用になります。

## リモート接続 — 本番の REPL に入る

| | Laravel | Rails | Elixir |
|---|---|---|---|
| 定番の方法 | サーバーに SSH して `php artisan tinker` | サーバーに SSH して `bin/rails console -e production` | **稼働中の VM に後から接続**（`bin/blog remote`） |
| プロセスの関係 | 新規プロセスを起動 | 新規プロセスを起動 | **既存プロセスに相乗り** |

ここにも BEAM の設計思想が現れます。Rails / Laravel の「本番 console」は、Web サーバーとは**別の新しいプロセス**を起動してアプリを読み込み直すものです。一方 Elixir では、`mix release` で作ったリリースに `remote` コマンドが同梱されており、**すでに動いている本番ノードの VM 内部にシェルで入ります**。

```bash
$ bin/blog remote        # 稼働中の blog ノードに接続
iex(blog@prod-1)1> :sys.get_state(Blog.SomeWorker)   # 生きているプロセスの状態を直接見る
```

内部的には Erlang の分散機能（`--remsh`）で、`iex --sname console --cookie <cookie> --remsh blog@prod-1` を手で打つのと同等です。接続先のシェルは Ctrl+C 2回で終了でき、落ちるのは手元の接続用ノードだけでリモート本体には影響しません。本当の罠は別にあります: リモートシェルに打ち込んだコードは**リモートノード上で実行される**ため、`System.stop()` や `:init.stop()` を実行すると**本番ノード本体が停止**します。新規プロセス起動型と違い、メモリ上の状態（ETS テーブル、GenServer の内部状態、稼働中のプロセス一覧）をそのまま観察できるのが決定的な違いで、本番障害の調査能力が一段変わります。

## 履歴・補完・マルチライン編集

| | PsySH (tinker) | IRB (rails console) | IEx |
|---|---|---|---|
| 履歴の永続化 | 標準で有効 | 標準で有効（`SAVE_HISTORY` 件数指定） | **要設定**（下記） |
| タブ補完 | クラス・関数・変数 | 補完ダイアログ（候補をポップアップ表示） | モジュール・関数名 |
| マルチライン編集 | 対応 | 対応（Reline による複数行編集） | 継続行方式 ※ |
| 入力の途中破棄 | Ctrl+C | Ctrl+C | `#iex:break` と入力 |

IEx の履歴はデフォルトで**セッションを跨いで保存されません**。実務では環境変数で有効化しておくのがほぼ必須です。

```bash
# ~/.bashrc など
export ERL_AFLAGS="-kernel shell_history enabled"
```

※ マルチラインについて: IEx は「式が閉じていなければ `...(1)>` プロンプトで継続入力」という方式で、IRB のように複数行をまとめて上下移動しながら編集するスタイルとは異なります（Erlang/OTP 26 以降はシェル自体の複数行編集・補完が大きく改善されています）。継続入力の途中で「この式を捨てたい」ときは、Ctrl+C ではなく **`#iex:break` と打って Enter** すると入力を破棄してプロンプトに戻れます。

## IEx 特有の操作: BREAK メニューと Ctrl+G

Laravel / Rails 経験者が IEx で最初に戸惑うのが「終了の仕方」です。`exit` と打っても関数がないと言われます。対応は次のとおりです。

| やりたいこと | tinker | rails console | IEx |
|---|---|---|---|
| 終了 | `exit` / Ctrl+D | `exit` / Ctrl+D | **Ctrl+C を2回**（または Ctrl+\ で即終了） |
| 割り込みメニュー | — | — | Ctrl+C（BREAK メニュー） |
| シェルの切替・多重化 | — | — | Ctrl+G（User switch command） |

Ctrl+C を1回押すと BREAK メニューが開きます。

```text
BREAK: (a)bort (A)bort with dump (c)ontinue (p)roc info (i)nfo
       (l)oaded (v)ersion (k)ill (D)b-tables (d)istribution
```

`a` で VM ごと終了、`c` で何事もなかったように復帰、`p` でプロセス情報の表示などができます。「Ctrl+C 2回で終了」は実際には「BREAK メニューを開いてもう一度 Ctrl+C で abort」という操作です。これは IEx ではなく **Erlang VM 自体の機能**なので、abort で落ちるのは「いま端末が繋がっているVM」です。`iex -S mix phx.server` を直接動かしている端末や、コンテナに `docker attach` している場合は**サーバーごと止まります**。一方 `remote`（remsh）接続中なら落ちるのは手元の接続用ノードだけで、リモート本体は無事です（リモート側を止めてしまうのは前述の `System.stop()` 系の方です）。

Ctrl+G の「User switch command」はさらに独特で、1つの端末の中で複数のシェルを起動して切り替えたり（`s` で新シェル、`c 2` で2番へ切替）、`r` コマンドで他ノードのリモートシェルに接続したりできます。tmux のウィンドウ機能が VM に内蔵されているようなイメージです。

## まとめ

- 3者とも「アプリを読み込んだ REPL + ドキュメント/ソース閲覧 + オブジェクト調査」という基本装備は同等。IRB の近年の進化（`show_source` / `ls` / 補完ダイアログ）で日常の使い勝手の差はかなり縮まっています
- 設定ファイルは `~/.config/psysh/config.php` / `.irbrc` / `.iex.exs`。**Phoenix ではプロジェクト直下の `.iex.exs` にエイリアスを仕込むのが定番**
- サンドボックスは Rails だけの標準装備。Elixir は `Repo.transaction` + `rollback` で代用
- Elixir 固有の強みは `iex -S mix phx.server`（サーバーと REPL の同居）と `bin/app remote`（稼働中の本番 VM への接続）。どちらも BEAM が軽量プロセスの集合体であることの直接の恩恵です
- IEx の履歴保存は `ERL_AFLAGS="-kernel shell_history enabled"` を忘れずに。終了は Ctrl+C 2回、入力破棄は `#iex:break`

## 参考リンク

- [IEx — hexdocs](https://hexdocs.pm/iex/IEx.html) / [IEx.Helpers](https://hexdocs.pm/iex/IEx.Helpers.html)
- [PsySH](https://psysh.org/) / [PsySH Config options](https://github.com/bobthecow/psysh/wiki/Config-options)
- [Laravel 13.x: Artisan Console — Tinker (REPL)](https://laravel.com/docs/13.x/artisan#tinker)
- [IRB — GitHub](https://github.com/ruby/irb) / [IRB Configurations](https://ruby.github.io/irb/Configurations_md.html)
- [Rails ガイド: コマンドライン — bin/rails console](https://guides.rubyonrails.org/command_line.html)
- [rails/rails#48984 — sandbox_by_default](https://github.com/rails/rails/pull/48984)
- [Erlang/OTP 26 Highlights（シェル改善）](https://www.erlang.org/blog/otp-26-highlights/)
- [Ecto.Adapters.SQL.Sandbox](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.Sandbox.html)

---

次の記事: **[Laravel・Rails・Phoenix 対応表: インラインデバッグ — binding.pry と IEx.pry の世界](https://qiita.com/ak-inori/items/0f999ab54f620d829366)**
