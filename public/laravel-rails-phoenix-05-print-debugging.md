---
title: Laravel・Rails・Phoenix 対応表（5/8）プリントデバッグ — dd() / pp / IO.inspect
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - debug
private: true
updated_at: '2026-08-24T18:27:37+09:00'
id: ab069ea3cc797b16e20d
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

「とりあえず変数の中身を見たい」ときに手が伸びる関数——Laravel なら `dd()`、Ruby なら `pp`、Elixir なら `IO.inspect`。本記事では、この3エコシステムのプリントデバッグ手段を対応表にまとめたうえで、`IO.inspect` のオプション、`dbg()` マクロの仕組み、機密情報を出力から隠す方法、ログ出力の対比まで深掘りします。

本記事は Laravel・Rails・Phoenix 対応表シリーズ（全8回）の第5回です。

対象バージョン（2026年8月執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.5 | Ruby 4.0（Rails 8.1 の必須要件は 3.2+） | Elixir 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

---

## 基本の対応表

| | PHP / Laravel | Ruby / Rails | Elixir |
|---|---|---|---|
| 雑に出力 | `var_dump($x)` / `print_r($x)` | `p x` / `pp x` | `IO.inspect(x)` |
| 出力して**実行継続**（値を返す） | `dump($x)` | `p x` / `pp x`（引数を返す） | `IO.inspect(x)` / `dbg(x)`（引数を返す） |
| 出力して**実行停止** | **`dd($x)`** | なし ※1 | なし ※2 |
| ラベル付き | `dump('users:', $users)` | `pp users:` … | `IO.inspect(x, label: "users")` |
| 文字列表現を得る | `var_export($x, true)` | `x.inspect` | `inspect(x)`（`Kernel.inspect/2`） |
| ログに出す | `Log::debug($x)` | `Rails.logger.debug` | `Logger.debug(inspect(x))` |

※1: Ruby には `dd()` 直接対応はなく、`pp x; raise "debug"` などで代用します。
※2: Elixir も同様。「そこで止めたい」は raise ではなく `IEx.pry()` や `break!` で「対話モードに入る」のが流儀です（インラインデバッグの記事で詳述します）。

**「dump して die する」関数を標準で持つのは Laravel だけ**です。Ruby / Elixir では「止めたい」と「見たい」が別の道具（REPL デバッガとプリント関数）に分かれており、Laravel の `dd()` は両者を一撃で兼ねる文化圏の関数だと言えます。逆に言うと、Laravel から来た人が Ruby / Elixir で `dd` を探して見つからないのは仕様です。

素の出力関数の使い分けも軽く整理しておきます。

- PHP: `var_dump` は型情報付きで冗長、`print_r` は人間向けに簡潔、`var_export` は「PHP コードとして再利用できる」表現を返します。Laravel（Symfony VarDumper）の `dump()` はブラウザでは折りたたみ可能な HTML、CLI では色付きテキストで出力する上位互換です
- Ruby: `p` は `inspect` 結果を1行で、`pp` はネストの深い構造をインデント付きで整形表示します。どちらも**引数をそのまま返す**のがポイントです
- Elixir: `IO.inspect/2` が唯一にして万能の入口です。次節で詳しく見ます

---

## IO.inspect のオプション詳解

`IO.inspect/2` は第2引数にキーワードリストを取り、挙動を細かく制御できます。よく使うものから。

```elixir
users
|> IO.inspect(label: "after filter")   # 行頭に "after filter: " を付ける
|> Enum.map(& &1.name)
|> IO.inspect(label: "names", limit: :infinity)
```

| オプション | 既定値 | 意味 |
|---|---|---|
| `label:` | なし | 出力の先頭にラベルを付ける。パイプラインに複数挟むとき必須級 |
| `limit:` | `200` | コレクションの表示要素数上限。超過分は `...` に省略。`:infinity` で全表示 |
| `printable_limit:` | `4096` | 文字列・charlist の表示文字数上限 |
| `pretty:` | `false` | `true` で改行・インデント付き整形 |
| `width:` | `80` | `pretty: true` 時の1行幅 |
| `structs:` | `true` | `false` にすると構造体を素の Map として表示 |
| `charlists:` | `:infer` | 整数リストを charlist と推論表示するか。`:as_lists` で常にリスト表示 |
| `binaries:` | `:infer` | `:as_binaries` でバイナリをバイト列 `<<...>>` として表示 |
| `syntax_colors:` | — | 型ごとの色付け（IEx では既定で有効） |

いくつか落とし穴込みで補足します。

**`limit` の既定値はバージョンで違います。** 長らく 50 でしたが、Elixir 1.19 で 100、1.20 で 200 に引き上げられました。「リストの末尾が `...` になっていて肝心の要素が見えない」ときは `limit: :infinity` です。Rails 経験者なら「コンソールで `ActiveRecord::Relation` が省略表示される」感覚に近いです。

**「文字列に見えるけど文字列じゃない」問題。** Elixir で整数のリストがすべて印字可能な文字コードだと、charlist として推論表示されます。

```elixir
iex> IO.inspect([104, 101, 108, 108, 111])
~c"hello"
iex> IO.inspect([104, 101, 108, 108, 111], charlists: :as_lists)
[104, 101, 108, 108, 111]
```

「数値のリストを返したはずなのに `~c"..."` が出てくる」のは Elixir 入門者が最初に踏む罠の筆頭です。表示が紛らわしいだけで値はリストのままなので、`charlists: :as_lists` で正体を確認できます。逆に「文字列のはずなのに `<<239, 191, ...>>` と出る」ときは印字不能バイトが混ざったバイナリで、`binaries: :as_binaries` で全バイトを確認できます。

**構造体の中身を暴く `structs: false`。** `DateTime` や `Decimal` のように `Inspect` 実装が整形表示してしまう構造体も、`structs: false` なら内部フィールドまで素の Map として見えます。

**出力先も変えられます。** `IO.inspect/3` の第1引数にデバイスを渡せるので、標準出力がテスト等で汚したくない場面では `IO.inspect(:stderr, x, label: "debug")` と stderr に逃がせます。

ログメッセージに埋め込みたいときは、文字列を返す `Kernel.inspect/2` を使います（`IO.inspect` と同じオプションが効きます）。

```elixir
Logger.debug("params: #{inspect(params, pretty: true)}")
```

---

## 「値を返す」性質とチェーン挟み込みパターン

3エコシステムとも、デバッグ出力関数が**引数をそのまま返す**ことを活かして、既存のチェーンに「挟むだけ」で中間値を観察できます。書き換え量が少ないほど消し忘れも減るので、これが実戦での基本形です。

### Elixir: パイプラインに `IO.inspect` / `dbg`

```elixir
[1, 2, 3]
|> IO.inspect(label: "input")
|> Enum.map(&(&1 * 2))
|> IO.inspect(label: "doubled")
|> Enum.sum()
```

さらに `dbg()` をパイプライン末尾に置くと、**各ステップの中間値を一括表示**してくれます（次節で詳述）。

### Laravel: クエリビルダとコレクションの `->dump()` / `->dd()`

Laravel はヘルパ関数だけでなく、クエリビルダ・Eloquent ビルダ・コレクションに `dump()` / `dd()` メソッドを生やしています（Laravel 11 からは `Dumpable` トレイトとして整理され、`Stringable` などにも入っています）。

```php
User::where('active', true)
    ->dump()          // この時点の SQL とバインディングを出力して継続
    ->where('age', '>', 20)
    ->dd();           // ここで SQL を出力して停止

collect([1, 2, 3])
    ->map(fn ($n) => $n * 2)
    ->dump()          // [2, 4, 6] を出力して継続
    ->sum();
```

クエリビルダには Laravel 10.15 以降、**バインディングを埋め込んだ生 SQL** を出す `dumpRawSql()` / `ddRawSql()`（+ 文字列で得る `toRawSql()`）もあります。「`?` だらけの SQL をコピペして手で埋める」作業から解放されるので、Rails の `to_sql` 感覚で使えます。

### Ruby: `tap` で挟む

Ruby は `p` / `pp` 自体が引数を返すので式の先頭に置けますが、メソッドチェーンの途中に挟むなら `tap` です。

```ruby
users
  .select(&:active?)
  .tap { pp it }        # Ruby 3.4+ の it パラメータ。挟んでも値はそのまま流れる
  .map(&:name)
```

`tap` はブロックの戻り値を捨ててレシーバを返すので、「観察するだけで流れを変えない」ことが型として保証されます。

### `tap` 系の対応表

| | PHP / Laravel | Ruby | Elixir |
|---|---|---|---|
| 値を観察して素通し | `tap($x, fn ($v) => dump($v))` | `x.tap { pp it }` | `x \|> tap(&IO.inspect/1)` |
| 値を変換して次へ | `Str::of($x)->pipe(...)` など | `x.then { ... }` | `x \|> then(&...)` |

Elixir の `Kernel.tap/2` / `then/2`（1.12+）は Ruby の `Object#tap` / `then` の直訳で、Laravel の `tap()` ヘルパも同じ発想です。**「tap = 観察（戻り値を捨てる）、then = 変換（戻り値を使う）」**という使い分けは3者共通で通用します。

---

## dbg() の深掘り — 関数ではなくマクロ

Laravel の `dd()` に一番近そうに見える Elixir の `dbg()` は、実際には **dump して die しない**（実行継続・値を返す）ので `dump()` 側の対応です。そのうえで `dump()` より賢い。理由は `dbg` が**関数ではなくマクロ**だからです。

関数の `IO.inspect` は「評価済みの値」しか受け取れませんが、マクロの `dbg` はコンパイル時に**式の AST（構文木）ごと**受け取ります。だから「どのファイルの何行目で」「どんなコードが」「何に評価されたか」まで表示でき、パイプラインを渡せば分解して各ステップを個別に評価・表示できます。

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

### パイプライン以外も特別扱いされる

Elixir 1.18 から、`dbg` は `case` / `cond` / `if` 式にも対応しました。式全体を包むと、**条件の評価値と「どの節にマッチしたか」**（`clause #n matched` の形式）まで表示されます。「この `case`、どの節に落ちてるんだ？」を `IO.inspect` を各節にばら撒かずに確認できるのは、マクロならではの芸当です。

```elixir
dbg(
  case status do
    :active -> "有効"
    :suspended -> "停止中"
    _ -> "不明"
  end
)
```

### `--dbg pry` でブレークポイントに化ける

`iex --dbg pry -S mix phx.server` で起動すると、`dbg()` 到達時にその場で pry（対話モード）に入るか聞かれます。つまり **`dbg` は「プリントデバッグ」と「ブレークポイント」を起動フラグで切り替えられる両用マーカー**です。カスタマイズの余地も公式に用意されており、`:elixir` アプリケーション設定の `:dbg_callback` で `dbg` の展開処理そのものを差し替えられます。

### 本番に `dbg` を残さないために

「値を返すから挟んだまま動いてしまう」のはチェーン挟み込み系デバッグ共通の弱点です。消し忘れ対策も3エコシステムで揃っています。

| | PHP / Laravel | Ruby / Rails | Elixir |
|---|---|---|---|
| 静的検出 | PHPStan 拡張（banned-code 系）等で `dd` / `dump` を禁止 | RuboCop `Lint/Debugger`（`binding.irb` 等）/ rubocop-rails `Rails/Output`（`p` / `pp` / `puts`） | Credo `Credo.Check.Warning.Dbg` / `Credo.Check.Warning.IoInspect` |

Credo の `Dbg` / `IoInspect` チェックはデフォルト構成に含まれているので、CI で `mix credo` を回していれば消し忘れはまず通りません。Phoenix 1.8 がプロジェクト生成時に用意する `mix precommit` エイリアスに Credo を足しておくのが手軽です。

---

## 機密情報を inspect から隠す

プリントデバッグとログの実運用で必ず問題になるのが「パスワードやトークンが出力に混ざる」ことです。3エコシステムとも「inspect / ログ時にマスクする」仕組みを持っていますが、**効く場所が違う**ので対比しておきます。

| | Laravel | Rails | Elixir / Ecto |
|---|---|---|---|
| リクエストパラメータのログ | —（そもそもパラメータを自動ログしない） | `config.filter_parameters`（`[FILTERED]` 表示） | Phoenix `filter_parameters`（既定で `"password"` をフィルタ） |
| モデルの inspect | — | `filter_attributes`（Rails 6+、`filter_parameters` を継承） | `@derive {Inspect, only: [...]}` / Ecto `field ..., redact: true` |
| シリアライズ（JSON化） | `$hidden` / `$visible` | `serializable_hash` のオプション等 | JSON ビューで出すフィールドを明示 |

### Elixir: Inspect プロトコルと `redact: true`

Elixir では構造体の表示自体が `Inspect` プロトコルで定義されるので、derive で表示フィールドを絞れます。

```elixir
defmodule MyApp.Accounts.User do
  @derive {Inspect, only: [:id, :email]}
  defstruct [:id, :email, :password_hash]
end

iex> IO.inspect(%MyApp.Accounts.User{id: 1, email: "a@example.com", password_hash: "..."})
#MyApp.Accounts.User<id: 1, email: "a@example.com", ...>
```

`only:` の代わりに `except:` も使えます。Ecto スキーマならフィールド定義に `redact: true` を付けるだけで、**changeset 内の変更値は `**redacted**` と表示され、スキーマの inspect からも除外**されます（Ecto が `Inspect` を自動 derive します）。`mix phx.gen.auth` が生成する User スキーマの `password` フィールドにも最初から付いています。

注意点として、`IO.inspect(x, structs: false)` は `Inspect` 実装を素通しして Map として表示するので、**redact は絶対的な防壁ではありません**。「事故で画面やログに出るのを防ぐ」ための仕組みと捉えてください。

### Rails: filter_parameters と filter_attributes

Rails は伝統的に `config/initializers/filter_parameter_logging.rb` の `config.filter_parameters` でリクエストログをマスクしてきましたが、Rails 6 からは同じリストが **ActiveRecord の `inspect` にも効きます**（`filter_attributes` として継承。モデル単位の上書きも可能）。

```ruby
class User < ApplicationRecord
  self.filter_attributes += [:legacy_token]
end

irb> user
#<User id: 1, email: "a@example.com", password_digest: [FILTERED], ...>
```

さらに Rails 7.2 からは `attributes_for_inspect` が導入され、**本番環境では `inspect` が既定で id しか表示しません**（development / test は `:all`）。「本番コンソールで巨大レコードをうっかり inspect して固まる・機密が出る」の両方への対策です。

### Laravel: `$hidden` は serialization 専用

Laravel の `$hidden` は `toArray()` / `toJson()` からの除外であって、**`dd($user)` や `var_dump` には効きません**。`dd()` はモデルの内部プロパティを生で表示するので、attributes 配列の中に password ハッシュもそのまま見えます。プリントデバッグの文脈では「Laravel にはモデル inspect のマスク機構がない」と覚えておくのが安全で、必要なら `dd($user->toArray())` と serialization 経由で見る（= `$hidden` を効かせる）のが簡単な回避策です。

---

## 出力はどこに出るのか — ブラウザ・ターミナル・専用ツール

「出力して、どこで見るか」も三者三様です。

| | Laravel | Rails | Phoenix |
|---|---|---|---|
| 既定の出力先 | ブラウザ（HTML）/ CLI | サーバーのターミナル・ログ | **サーバーのターミナル** |
| ビューに埋めて見る | `{{ dump($post) }}` | `<%= debug @post %>`（YAML表示ヘルパ） | `<pre><%= inspect(@post, pretty: true) %></pre>` |
| レスポンスを汚さず横取り | **Telescope の Dump ウォッチャー**（`php artisan dump-server` は後述の注意あり） | — | —（もともと端末に出る） |
| ブラウザ内コンソール | — | web-console gem（開発時デフォルト、`console` ヘルパ） | — |
| 専用GUIツール | Telescope の Dump ウォッチャー / Ray（Spatie製・有償） | — | LiveDashboard（メトリクス寄り） |

Laravel の `dump()` / `dd()` は Web リクエスト中ならレスポンス（ブラウザ）に出るため、**API のレスポンス JSON を壊す**のが難点です。Telescope を使っているなら Dump ウォッチャー（Telescope の Dumps 画面を開いている間だけ記録）でレスポンスを汚さずに確認できます。開発環境に Laravel Herd を使っている場合は Herd の Dumps 画面も同じ用途に使えます。この分野の古参である beyondcode/laravel-dump-server（`php artisan dump-server`）は、執筆時点の最新版（2.1.0）が **Laravel 12 までの対応**なので、Laravel 13 では対応状況を確認してから導入してください。

Rails / Phoenix は「開発サーバーを起動したターミナルに出る」が基本なので、この問題自体が起きにくい構図です。Rails のビューには古参の `debug` ヘルパ（オブジェクトを YAML で `<pre>` 表示）があり、Phoenix / LiveView では同じことを `inspect(..., pretty: true)` を HEEx に埋めて行います。

**LiveView での注意点**: LiveView のコード内の `IO.inspect` は**ブラウザではなくサーバーのターミナル**に出ます。JavaScript の `console.log` の感覚で「ブラウザの開発者ツール」を眺めていても何も出ないので、見る場所は `mix phx.server` を起動した端末です。マウント時とイベント時で2回出る（HTTP レンダリングと WebSocket 接続後）のも LiveView 特有の「あるある」です。

---

## ログに出す — 構造化コンテキストの対比

プリントデバッグの「恒久版」がログ出力です。3者ともメッセージに**構造化されたコンテキスト（メタデータ）**を付けられます。

| | Laravel | Rails | Elixir |
|---|---|---|---|
| デバッグログ | `Log::debug('msg', ['user_id' => 1])` | `Rails.logger.debug("msg")` | `Logger.debug("msg", user_id: 1)` |
| コンテキストの付け方 | 第2引数の連想配列 / `Log::withContext()` | タグ: `Rails.logger.tagged("PAY") { ... }` | メタデータ（キーワードリスト）/ `Logger.metadata/1` |
| 遅延評価 | — | `logger.debug { heavy_dump }` | `Logger.debug(fn -> inspect(heavy) end)` |
| ログレベル設定 | `config/logging.php`（`LOG_LEVEL` 環境変数） | `config/environments/*.rb` の `config.log_level` | `config/dev.exs` 等の `config :logger, level: :debug` |

それぞれの流儀の違いが出るポイントを3つ。

**1. コンテキストの表現。** Laravel は PSR-3 流の「メッセージ + コンテキスト配列」、Rails は `ActiveSupport::TaggedLogging` の「行頭タグ」（`[PAY] msg` 形式）、Elixir は Logger の**メタデータ**です。Elixir のメタデータはログ行に必ず印字されるわけではなく、フォーマッタ側で表示対象を指定します。

```elixir
# config/dev.exs — 表示したいメタデータを指定（:all も可）
config :logger, :default_formatter, metadata: [:request_id, :user_id]
```

Phoenix は最初から `Plug.RequestId` で `request_id` をメタデータに積んでいるので、1リクエストのログを grep で追跡できます。Rails の `config.log_tags = [:request_id]`、Laravel の `Log::withContext(['request-id' => ...])`（ミドルウェアで仕込む定番パターン）がそれぞれの同型です。

**2. 遅延評価。** 「debug レベルは本番で出ないのに、`inspect(巨大な構造体)` の計算コストだけ払う」のを避けるため、Rails はブロック、Elixir は無名関数を渡せます。Elixir はさらに一歩進んでいて、コンパイル時にログ呼び出しごと消す設定があります。

```elixir
# config/prod.exs — :info 未満の Logger 呼び出しをコンパイル時に除去
config :logger, compile_time_purge_matching: [[level_lower_than: :info]]
```

実行時に「レベルを見て捨てる」のではなく、**本番ビルドのバイトコードから debug ログが消える**ので、引数の評価コストもゼロになります。

**3. レベル設定の場所。** Laravel は `.env` の `LOG_LEVEL`（チャンネル単位は `config/logging.php`）、Rails は環境別設定ファイル、Elixir は `config/{dev,prod}.exs` と、いずれも「環境ごとの設定」に置く点は共通です。

---

## まとめ

- **`dd()`（dump して die）を標準で持つのは Laravel だけ**。Ruby / Elixir は「見る」はプリント関数、「止まる」は REPL デバッガと役割分担する
- `IO.inspect` はオプションが本体。`label:` / `limit: :infinity` / `charlists: :as_lists` / `structs: false` の4つを覚えておくと「表示が省略される」「文字列に見える」系の混乱をその場で解ける
- 3エコシステムとも**デバッグ出力が値を返す**ので、チェーン・パイプラインに挟むのが基本形。Laravel はビルダの `->dump()` / `->dumpRawSql()`、Ruby は `tap`、Elixir は `IO.inspect` / `dbg` を挟む
- `dbg()` はマクロなのでコードの構造ごと表示できる（パイプライン分解、Elixir 1.18+ では `case` / `cond` / `if` のマッチ節表示）。`--dbg pry` でブレークポイントにも化ける
- 機密のマスクは Elixir が `@derive Inspect` / Ecto `redact: true`、Rails が `filter_parameters` / `filter_attributes` / `attributes_for_inspect`。Laravel の `$hidden` は **serialization 専用で `dd()` には効かない**点に注意
- 消し忘れ対策（Credo / RuboCop / PHPStan 拡張）を CI に入れておくと、チェーン挟み込みデバッグを安心して多用できる

---

## 参考リンク

- [`Kernel.dbg/2`（Hexdocs）](https://hexdocs.pm/elixir/Kernel.html#dbg/2) / [`IO.inspect/2`](https://hexdocs.pm/elixir/IO.html#inspect/2) / [`Inspect.Opts`](https://hexdocs.pm/elixir/Inspect.Opts.html)
- [Elixir 公式ガイド: Debugging](https://hexdocs.pm/elixir/debugging.html)
- [Ecto.Schema — `redact: true`](https://hexdocs.pm/ecto/Ecto.Schema.html)
- [Credo.Check.Warning.Dbg](https://hexdocs.pm/credo/Credo.Check.Warning.Dbg.html) / [Credo.Check.Warning.IoInspect](https://hexdocs.pm/credo/Credo.Check.Warning.IoInspect.html)
- [Laravel: Helpers（`dd` / `dump`）](https://laravel.com/docs/13.x/helpers) / [Laravel: Logging](https://laravel.com/docs/13.x/logging) / [Laravel: Eloquent Serialization（`$hidden`）](https://laravel.com/docs/13.x/eloquent-serialization)
- [laravel/framework #47507 — `toRawSql` / `dumpRawSql` / `ddRawSql`](https://github.com/laravel/framework/pull/47507)
- [beyondcode/laravel-dump-server](https://github.com/beyondcode/laravel-dump-server) / [Laravel Telescope](https://laravel.com/docs/13.x/telescope) / [Laravel Herd: Dumps](https://herd.laravel.com/docs/macos/debugging/dumps)
- [Rails at Scale: Fixing a footgun in ActiveRecord::Core#inspect（`attributes_for_inspect`）](https://railsatscale.com/2024-04-30-fixing-a-footgun-in-activerecord-core-inspect/)
- [Rails API: `filter_attributes`](https://api.rubyonrails.org/classes/ActiveRecord/Core/ClassMethods.html) / [ActiveSupport::TaggedLogging](https://api.rubyonrails.org/classes/ActiveSupport/TaggedLogging.html)
- [Logger（Hexdocs） — メタデータと `compile_time_purge_matching`](https://hexdocs.pm/logger/Logger.html)
