---
title: 'Laravel・Rails・Phoenix 対応表（4/8）インラインデバッグ — binding.pry と IEx.pry の世界'
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - デバッグ
private: true
updated_at: '2026-08-13T13:12:21+09:00'
id: 0f999ab54f620d829366
organization_url_name: null
slide: false
ignorePublish: false
---

「コードの途中で実行を止めて、その場の変数を触りながら原因を探る」——Rails の `binding.pry` に代表されるインラインデバッグは、3つのエコシステムでそれぞれ流儀が違います。Ruby は同種のツールが3つ並存して使い分けが必要、PHP は REPL 系（PsySH）と IDE 系（Xdebug）の二本立て、Elixir は **「iex 配下で動かしていること」が大前提**という独自の制約があります。

本記事では PHP/Laravel・Ruby/Rails・Elixir/Phoenix のインラインデバッグ手段を対応表で整理し、それぞれの使い分け・落とし穴・テスト実行中に止める方法・エディタ統合までまとめます。本記事は Laravel・Rails・Phoenix 対応表シリーズ（全8回）の第4回です。

対象バージョン（執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.5 | Ruby 4.0（Rails 8.1 の必須要件は 3.2+） | Elixir 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

---

## 全体対応表

| | PHP | Ruby | Elixir |
|---|---|---|---|
| ブレークポイントを書く | `eval(\Psy\sh());`（PsySH） | `binding.irb`（標準）<br>`binding.break`（debug gem）<br>`binding.pry`（pry） | `require IEx; IEx.pry()`<br>`dbg()`（`--dbg pry` 時） |
| 前提条件 | psysh が入っていること | それぞれの gem | **`iex` 配下で起動していること** |
| 再開 | `exit` / Ctrl+D | `continue`（debug gem）/ `exit` | `continue` / `respawn` |
| コードを書き換えずに止める | Xdebug（IDE 連携） | `rdbg` から `break` / IDE 連携 | `break! Mod.fun/arity` |
| ステップ実行 | Xdebug | debug gem（`step` / `next`） | `next`（行単位）※ |
| テスト中に止める | そのまま動く | そのまま動く | **`iex -S mix test --trace` が必要** |

※ Elixir の `next` は pry/breakpoint セッション内で「次の行」に進むヘルパーです。IDE 的なフル機能のステップ実行（ステップイン/アウト、コールスタック表示）が必要な場合は後述の ElixirLS デバッグアダプタを使います。

以降、エコシステムごとに深掘りします。

---

## Ruby: binding.irb / binding.break / binding.pry の使い分け

Ruby は「実行を止めて対話する」手段が3つ並存しているのが特徴です。

| | `binding.irb` | `binding.break` | `binding.pry` |
|---|---|---|---|
| 提供元 | IRB（言語標準） | **debug gem**（Ruby 3.1+ に同梱） | pry gem（サードパーティ） |
| 追加インストール | 不要 | Rails なら Gemfile に標準で入っている | `gem "pry"` が必要 |
| ステップ実行 | `next` 等を打つと debug gem に委譲 ※ | ◎（本業） | pry-byebug の併用が必要 |
| リモート接続 / IDE | — | ◎（`rdbg` / DAP / Chrome DevTools） | — |
| 得意分野 | 「その場で変数を見るだけ」 | 本格的なデバッグ全般 | 歴史的定番。`ls` / `show-source` 等の探索系 |

※ debug gem が読み込める環境の場合。

**現在の推奨は標準添付の debug gem（`binding.break`）です。** かつては `binding.pry` がデファクトでしたが、Ruby 3.1 以降 debug gem が言語に同梱され、Rails も新規アプリの Gemfile に debug を含めるため、追加セットアップなしで使えます。`binding.break` には `binding.b`、`debugger` というエイリアスもあります。

### debug gem の主要コマンド

```ruby
def create
  @post = Post.new(post_params)
  binding.break        # ここで止まる（require "debug" 済みの前提）
  @post.save!
end
```

止まった後のプロンプトで使う代表的なコマンド:

| コマンド | 動作 |
|---|---|
| `s[tep]` | ステップイン（次の停止可能ポイントへ。メソッドの中に入る） |
| `n[ext]` | ステップオーバー（次の行へ） |
| `fin[ish]` | 現在のフレームを抜けるまで実行 |
| `c[ontinue]` | 次のブレークポイントまで再開 |
| `i[nfo]` | ローカル変数・インスタンス変数・定数の一覧 |
| `b[reak] Foo#bar` / `b file:42` | 追加のブレークポイントを対話的に設定 |
| `catch SomeError` | 例外の発生地点で停止 |
| `watch @ivar` | 値の変化を監視して停止（公式ドキュメント曰く「super slow」） |

`break` / `catch` には `if:`（条件）、`do:`（実行して自動継続）などの修飾子も付けられます。「ループの1000回目だけ止めたい」が1行で書けるのは pry-byebug 時代からの大きな進歩です。

### binding.irb は debug gem への入り口になった

「とりあえず止める」だけなら `binding.irb` が最軽量ですが、現在の IRB は debug gem と統合されており、**`binding.irb` のセッション内で `debug`、`break`、`catch`、`next`、`step`、`continue`、`finish`、`backtrace`、`info` を打つと、そのまま `irb:rdbg` セッション（debug gem）に昇格します**。「軽く見るつもりで `binding.irb` したが、ステップ実行したくなった」ときに仕込み直しが不要です。逆方向の統合として、環境変数 `RUBY_DEBUG_IRB_CONSOLE=1` を設定すると debug gem のコンソールが IRB になります。

### コードを書き換えずに止める

debug gem は「後から外部から繋ぐ」使い方に強く、これは pry にはない能力です。

```bash
rdbg -c -- bin/rails server   # デバッガ配下で起動
rdbg --open -- ruby app.rb    # ソケットを開いて起動、別ターミナルから rdbg -A で接続
```

`rdbg --open=vscode` なら VS Code（拡張 vscode-rdbg）に、`rdbg --open=chrome` なら Chrome DevTools に直接接続できます。

---

## PHP: PsySH（REPL 系）と Xdebug（IDE 系）の二本立て

PHP のインラインデバッグは性格の違う2系統があります。

| | PsySH | Xdebug（step debug） |
|---|---|---|
| 方式 | コードに1行書いてシェルに入る | ブレークポイントは IDE 側で設定 |
| コード変更 | 必要 | **不要** |
| ステップ実行 | ✕（対話のみ） | ◎ |
| セットアップ | composer で入れるだけ | PHP 拡張 + php.ini + IDE 設定 |
| Rails で例えると | `binding.irb` | IDE 連携の debug gem |

### PsySH: `eval(\Psy\sh())`

Laravel の `php artisan tinker` の中身が PsySH なので、Laravel プロジェクトなら追加インストールなしで使えます。止めたい場所に次の1行を書きます。

```php
public function store(Request $request)
{
    $post = Post::create($request->validated());
    eval(\Psy\sh());   // ここで対話シェルに入る。$post や $request が見える
    return redirect()->route('posts.show', $post);
}
```

シェル内ではその地点のスコープの変数にアクセスでき、`ls` で定義済み変数やクラスの中身を探索、`doc` でドキュメント参照、`show` でソース表示ができます。`exit`（または Ctrl+D）で抜けると実行が再開されます。Ruby の `binding.irb` にかなり近い体験ですが、**ステップ実行はできません**。「1行ずつ追いたい」となったら Xdebug の出番です。

### Xdebug 3: 最小構成

php.ini（またはconf.d のini ファイル）に書くのは実質3行です。

```ini
zend_extension=xdebug
xdebug.mode=debug
xdebug.start_with_request=trigger
```

- `xdebug.mode=debug` でステップデバッグ機能が有効になります
- 接続ポートは **9003 がデフォルト**（`xdebug.client_port`）。Xdebug 2 時代の 9000 から変わっているので古い記事に注意
- `start_with_request=trigger` にすると常時接続ではなく、リクエストに `XDEBUG_TRIGGER`（または `XDEBUG_SESSION` クッキー。ブラウザ拡張が面倒を見てくれます）が付いたときだけデバッガに接続します。CLI では `export XDEBUG_SESSION=1` です

VS Code 側は拡張「PHP Debug」を入れて、`.vscode/launch.json` に最小構成を置くだけです。

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Listen for Xdebug",
      "type": "php",
      "request": "launch",
      "port": 9003
    }
  ]
}
```

「Listen for Xdebug」を開始 → エディタ上で行の左をクリックしてブレークポイント → ブラウザからトリガー付きでアクセス、で止まります。Docker の場合は `xdebug.client_host`（コンテナから見た IDE のホスト。Docker Desktop なら `host.docker.internal`）の指定が追加で必要です。Laravel Sail には `SAIL_XDEBUG_MODE` 環境変数が用意されており、`.env` に `SAIL_XDEBUG_MODE=develop,debug` と書くだけで有効化できます。

---

## Elixir: IEx.pry / break! / dbg の3枚看板

Elixir のインラインデバッグ最大の特徴は、**「iex 配下で動いているプロセス」でないと止まれない**ことです。Phoenix 開発中は `mix phx.server` ではなく `iex -S mix phx.server` で起動しておくのが、pry を使うための前提になります。

### パターン1: IEx.pry() — binding.pry 相当

```elixir
def create(conn, params) do
  require IEx; IEx.pry()    # iex -S mix phx.server で起動していないと素通り
  ...
end
```

到達すると iex シェル側に「Request to pry #PID<0.xxx.0>...（Allow? [Yn]）」という**許可プロンプト**が出て、Y で pry セッションに入ります。その関数のローカル変数・import・alias にアクセスできますが、Ruby の pry と違って**そこから値を書き換えて実行を続けることはできません**（Elixir のデータは不変で、pry は「覗く」ためのものです）。抜けるには:

- `continue` — 実行を再開する（次のブレークポイントがあればそこまで）
- `respawn` — pry を破棄して新しい iex シェルを立ち上げ直す

### パターン2: dbg() + --dbg pry

Elixir 1.14 で入った `dbg/2` は通常「値とコード位置を表示して実行継続」ですが、**`iex --dbg pry -S mix phx.server` で起動しておくと、`dbg()` 到達時に pry に入るブレークポイントに化けます**。

```elixir
def create(conn, params) do
  params
  |> Map.get("post")
  |> normalize()
  |> dbg()
  ...
end
```

pry に入った後、パイプラインに対しては **`n`（`next`）で「次のパイプへ」1段ずつ進めます**。パイプの各段の中間値を見ながら歩けるので、「パイプラインのどこで値が壊れたか」を探すのに最適です。コードには `dbg()` と書くだけで、止めるかどうかは起動フラグ側で切り替えられる（フラグなしなら print デバッグとして動く）のが実用上の利点です。

### パターン3: break! — コード無変更のブレークポイント

```elixir
iex> break! BlogWeb.PostController.create/2
iex> breaks()          # 設定済みブレークポイントの一覧と ID
```

次にそのアクションが呼ばれた瞬間に停止します。関連ヘルパーが一通り揃っています。

| ヘルパー | 動作 |
|---|---|
| `break!(Mod.fun/arity, stops)` | ブレークポイント設定。**`stops` は「止まる回数」でデフォルト 1** |
| `breaks/0` | ブレークポイント一覧（ID 付き） |
| `whereami/1` | 現在の停止位置の前後コードを表示 |
| `n/0`（`next/0`） | 次の行へ進む |
| `continue/0` | 再開 |
| `reset_break/1` | 指定 ID の残り停止回数を 0 にする |
| `remove_breaks/0,1` | 全部（またはモジュール単位で）ブレークポイント削除 |

`stops` がデフォルト 1 なのは要注意ポイントで、**2回目のリクエストでは素通りします**。繰り返し止めたいときは `break! BlogWeb.PostController.create/2, 10` のように回数を渡します。また公式ドキュメントに明記されている制約として、コンパイル済みモジュールに貼った breakpoint では **alias や import にアクセスできません**（`IEx.pry` を直接書いた場合は可能）。

### プロセスの世界ならではの補助ツール

pry で止めている間、ブロックされるのは**そのプロセスだけ**です。BEAM は他のリクエストを平然と処理し続けます。止めずに外から観察する道具も揃っています。

```elixir
iex> Process.info(pid)                    # プロセスのメッセージキュー長・メモリ等
iex> :sys.get_state(pid)                  # GenServer の現在のステートを覗く
iex> :observer.start()                    # GUI でプロセスツリー・負荷を可視化
```

---

## テスト実行中に止める — ここが一番差が出る

| | Laravel | Rails | Phoenix |
|---|---|---|---|
| 止め方 | Xdebug（IDE から）/ `eval(\Psy\sh())` | テストコードに `binding.break` 等 | `require IEx; IEx.pry()` |
| 実行コマンド | `php artisan test` のまま | `bundle exec rspec` のまま | **`iex -S mix test --trace`** |
| タイムアウト | なし | なし | **あり（デフォルト60秒）→ `--trace` で無効化** |

Ruby と PHP は「テストファイルにブレークポイントを書いて普通に実行」でそのまま止まります。RSpec で `binding.break`（または `binding.pry`）してじっくり30分調べても誰にも怒られません。

Elixir は2つ罠があります。まず `mix test` は iex 配下ではないので、**そのまま実行すると `IEx.pry()` は素通り**します。iex 配下で mix task を動かす形にする必要があります。さらに ExUnit には**テストごとのタイムアウト（デフォルト60秒）**があり、pry で対話している間に殺されます。`--trace` オプションはテストを直列＋詳細表示にすると同時に**タイムアウトを無効化する**ので、公式ドキュメントもこの形を案内しています。

```bash
iex -S mix test --trace                       # pry したいときの決まり文句
iex -S mix test --trace test/blog/post_test.exs:42   # ファイル・行指定も併用可
```

「`mix test` で pry が効かない！」は Elixir 入門者が最初に踏む定番の罠なので、この1行はスニペット登録しておく価値があります。

---

## IDE / エディタ統合の対応表

「コードに1行書く」方式の対比が本記事の主題ですが、GUI でステップ実行したい場合の対応関係も載せておきます。

| | PHP | Ruby | Elixir |
|---|---|---|---|
| 仕組み | Xdebug（DBGp プロトコル） | debug gem の `rdbg`（DAP） | ElixirLS のデバッグアダプタ（DAP） |
| VS Code 拡張 | PHP Debug | VSCode rdbg Ruby Debugger | ElixirLS |
| ステップ実行 | ◎ | ◎ | ◎ |
| 備考 | ほぼ全 PHP IDE が DBGp 対応 | Chrome DevTools 接続も可 | プロジェクトのモジュールを事前にインタープリタ実行に切り替える方式。ブレークポイント上限100個、テスト等 `.exs` は launch 設定の `requireFiles` 指定が必要 |

Elixir の ElixirLS デバッガは Erlang のインタープリタ（`:int`）ベースで、**デバッグ対象モジュールを解釈実行に切り替える**ため通常実行よりかなり遅くなります。日常は `dbg` / `IEx.pry` / `break!`、複雑なコールスタックを追うときだけ ElixirLS、という使い分けが現実的です。なお Erlang 標準の GUI デバッガ `:debugger` も昔からあり、こちらもステップ実行できます。

---

## LiveView・非同期プロセスで pry するときの注意

BEAM 上ではすべてがプロセスなので、「どのプロセスで pry が発火するか」を意識する必要があります。

- **pry で止まるのはそのプロセスだけ**。Rails のように「デバッガで止めたらサーバー全体が固まる」ことはなく、他のリクエストは処理され続けます。便利な反面、「止めている間に同じコードパスを別リクエストが通って、pry の許可プロンプトが次々降ってくる」ことがあります
- **LiveView の場合**: `mount/3` や `handle_event/3` に仕込んだ pry で LiveView プロセスを止めている間、そのクライアントへの応答も止まります。クライアント側のタイムアウトで再接続（再 mount）が走ると、**同じブレークポイントに再突入して pry 要求が繰り返し届く**ことがあります。プロンプトが増殖して混乱したら、いったん `respawn` で仕切り直すか `remove_breaks()` してから整理するのが安全です
- **Task や GenServer など非同期プロセス内**: そのプロセスが iex 配下の VM で動いてさえいれば pry は発火します（許可プロンプトは iex シェルに届きます）。ただし呼び出し元が `Task.await/2`（デフォルト5秒）や `GenServer.call/3`（デフォルト5秒）で待っている場合、**pry で対話している間に呼び出し元がタイムアウトして先に死ぬ**点に注意してください。テストの60秒タイムアウトと同根の問題で、止めて調べたいときはタイムアウトを一時的に `:infinity` にするか、`:sys.get_state/1` など「止めない観察」に切り替えるのが実務的です

---

## まとめ

- **Ruby**: 現在の本命は標準添付の debug gem（`binding.break`）。`step` / `next` / `finish` / `catch` / `watch` とリモート接続まで揃い、`binding.irb` からもシームレスに昇格できる。`binding.pry` は歴史的定番
- **PHP**: 対話なら PsySH の `eval(\Psy\sh())`（tinker と同じエンジン）、ステップ実行なら Xdebug。Xdebug 3 はポート 9003 + `mode=debug` + `start_with_request=trigger` が基本形
- **Elixir**: **iex 配下が大前提**。書いて止める `IEx.pry`、フラグで切り替える `dbg`（パイプを `n` で歩ける）、書き換えず止める `break!`（デフォルト1回で解除）の3枚看板。テストは `iex -S mix test --trace` が決まり文句
- 「実行を止める」体験は Ruby が最も手厚く、Elixir は「止めるより観察する」（`Process.info` / `:sys.get_state` / `:observer`）文化が補完している、というのが3者を触った実感です

---

## 参考リンク

- [IEx — pry / break! / helpers（Hexdocs）](https://hexdocs.pm/iex/IEx.html)
- [Debugging — Elixir 公式ガイド](https://hexdocs.pm/elixir/debugging.html)
- [`Kernel.dbg/2`（Hexdocs）](https://hexdocs.pm/elixir/Kernel.html#dbg/2)
- [ruby/debug（debug gem）](https://github.com/ruby/debug)
- [IRB — Debugging with IRB](https://ruby.github.io/irb/)
- [PsySH](https://psysh.org/)
- [Xdebug 3 — Step Debugging](https://xdebug.org/docs/step_debug)
- [ElixirLS](https://github.com/elixir-lsp/elixir-ls)
