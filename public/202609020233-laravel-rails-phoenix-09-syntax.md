---
title: 'Laravel・Rails・Phoenix 対応表: 言語シンタックス — リテラル・パターンマッチ・コレクション操作'
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - パターンマッチ
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

ここまでのシリーズはパッケージ管理やデバッグといったツールチェーンを扱ってきましたが、本記事は言語そのものに降りて、PHP・Ruby・Elixir の**構文の対応関係**を整理します。フレームワークのコードを読み書きするときに地味に手が止まる「Ruby の `%w` って Elixir だと何？」「Elixir の `{:ok, user} = ...` に相当する書き方は？」「`reduce` の引数の順番どっちだっけ？」を一気に片付けるのが目的です。

本記事は Laravel・Rails・Phoenix 対応表シリーズの9本目です。

シリーズ一覧:

1. [パッケージ管理 — Composer / Bundler / Mix](https://qiita.com/ak-inori/items/59ef9ff5937475c59d1b)
2. [プロジェクト作成と日常のコマンド — artisan / rails / mix](https://qiita.com/ak-inori/items/bc7fb3d522c04dd45205)
3. [REPL — tinker / rails console / IEx](https://qiita.com/ak-inori/items/73d50b297a770bf22ec7)
4. [インラインデバッグ — binding.pry と IEx.pry の世界](https://qiita.com/ak-inori/items/0f999ab54f620d829366)
5. [プリントデバッグ — dd() / pp / IO.inspect](https://qiita.com/ak-inori/items/ab069ea3cc797b16e20d)
6. [テスト — PHPUnit・Pest / Minitest・RSpec / ExUnit](https://qiita.com/ak-inori/items/927201acc9e87d164501)
7. [定番ライブラリ — ORM・認証・ジョブ・リアルタイムまで](https://qiita.com/ak-inori/items/b8e0dbfeafc0ead6f4b0)
8. [3つのエコシステムの思想の違い](https://qiita.com/ak-inori/items/e716638c92c3fba4e10e)
9. 言語シンタックス — リテラル・パターンマッチ・コレクション操作 **（本記事）**

対象バージョン（2026年9月執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.5 | Ruby 4.0（Rails 8.1 の必須要件は 3.2+） | Elixir 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

## 基本リテラルの対応表

| | PHP | Ruby | Elixir |
|---|---|---|---|
| 文字列 | `'foo'` / `"foo"` | `'foo'` / `"foo"` | `"foo"`（シングルクォートは別物 ※1） |
| 式展開 | `"Hello {$name}"` | `"Hello #{name}"` | `"Hello #{name}"` |
| シンボル / アトム | —（定数や enum で代用） | `:foo` | `:foo` |
| 整数の桁区切り | `1_000_000` | `1_000_000` | `1_000_000` |
| 配列 / リスト | `[1, 2, 3]` | `[1, 2, 3]` | `[1, 2, 3]`（連結リスト ※2） |
| タプル | — | —（配列で代用） | `{1, 2}` |
| 連想配列 / ハッシュ / マップ | `['a' => 1]` | `{ a: 1 }` / `{ "a" => 1 }` | `%{a: 1}` / `%{"a" => 1}` |
| キーワードリスト | — | キーワード引数 | `[timeout: 5000]`（`[{:timeout, 5000}]` の糖衣） |
| 範囲 | `range(1, 5)`（関数） | `1..5` / `1...5`（終端排他） | `1..5` / `1..10//2`（ステップ付き） |
| 正規表現 | `'/foo/'`（文字列で表現） | `/foo/` | `~r/foo/` |
| null / nil | `null` | `nil` | `nil`（実体は `:nil` アトム） |

※1: Elixir のシングルクォート（現在は `~c"foo"` 表記を推奨）は charlist（文字コードの整数リスト）で、Erlang ライブラリとの受け渡し以外ではほぼ使いません。`'abc'` と `"abc"` は別の型なので、Ruby の「どちらでも文字列」の感覚で書くと型エラーになります。

※2: Elixir のリストは配列ではなく連結リストです。先頭への追加 `[head | tail]` は O(1) ですが、`list[500]` のような添字アクセスは O(n) になるため、そもそも `Enum` 系の走査で書くのが前提です。

Ruby のシンボルと Elixir のアトムはほぼ同じ概念ですが、運用上の注意が1つ違います。Ruby のシンボルは 2.2 以降 GC 対象ですが、**Elixir のアトムは GC されません**。ユーザー入力を `String.to_atom/1` に流すとアトムテーブルを食い潰せるため、外部入力には `String.to_existing_atom/1` を使うのが定石です。

## %記法とシジル

Ruby の %記法に相当するのが Elixir のシジル（`~` 記法）です。PHP には相当する構文がありません。

| 用途 | Ruby | Elixir |
|---|---|---|
| 文字列の配列 / リスト | `%w[foo bar]` | `~w(foo bar)` |
| シンボル / アトムのリスト | `%i[foo bar]` | `~w(foo bar)a`（`a` 修飾子） |
| 式展開ありの文字列 | `%Q(...)` / `%(...)` | `~s(...)` |
| 式展開なしの文字列 | `%q(...)` | `~S(...)` |
| 正規表現 | `%r{...}` | `~r/.../` |
| 外部コマンド | `%x(ls)` | —（`System.cmd/3` を使う） |
| 日付・時刻リテラル | — | `~D[2026-09-01]` / `~T[12:00:00]` / `~U[2026-09-01 12:00:00Z]` |

覚えるときの罠がひとつ: **大文字・小文字の意味が両者で逆**です。Ruby は大文字（`%W` / `%Q`）が式展開あり、Elixir は小文字（`~w` / `~s`）が式展開ありで、大文字が「エスケープも展開もしない raw 版」です。

Elixir のシジルには、もうひとつ Ruby にない性質があります。`~x` は `sigil_x` という名前の関数・マクロの呼び出しに脱糖されるため、**ユーザー定義できる**のです。Phoenix を使っていると毎日目にする、コンパイル時にルーティングの存在を検証する `~p"/posts"`、HEEx テンプレートの `~H"""..."""` は、どちらもこの仕組みで実装されたライブラリ定義のシジルです。「%記法の一覧は言語仕様で固定」という Ruby との違いは、マクロを持つ言語らしいところです。

```elixir
defmodule MySigils do
  # ~k(foo bar) でアトムキーのマップを作る自作シジル
  def sigil_k(string, _modifiers) do
    string |> String.split() |> Map.new(&{String.to_atom(&1), true})
  end
end

iex> import MySigils
iex> ~k(admin editor)
%{admin: true, editor: true}
```

## 変数: 再代入と再束縛、そしてイミュータブル

3言語とも変数は snake_case ですが、「変数に入っているものを変更できるか」が根本的に違います。

```php
// PHP: 配列は値渡しコピー、オブジェクトは参照。ミュータブル
$user['name'] = 'Alice';
```

```ruby
# Ruby: オブジェクトはミュータブル。破壊的メソッドには ! を付ける慣習
name.upcase!          # name 自体が変わる
name.upcase           # 新しい文字列を返す（元は不変）
```

```elixir
# Elixir: すべてのデータがイミュータブル。「変更」は常に新しい値を返す
user = %{user | name: "Alice"}   # 更新構文も新しいマップを作って再束縛している
```

Elixir では `x = x + 1` のような再束縛は自由にできるので、書き味は普通の言語と変わりません。ただし**データそのものは絶対に変わらない**ため、「関数にリストを渡したら中身を書き換えられて戻ってきた」という事故が構造的に起きません。`Enum.sort(list)` が破壊的かどうかを調べる必要がない（破壊的なわけがない）のは、Ruby の `sort` / `sort!` の使い分けに慣れた身には新鮮なはずです。

定数の扱いも三者三様です。PHP は `const` / `define()`、Ruby は大文字始まりの定数（ただし警告付きで再代入可能）、Elixir には定数がなく、モジュール属性 `@timeout 5000` をコンパイル時定数のように使います。

## パターンマッチ

ここが本記事の主役です。Elixir では `=` は代入演算子ではなく**マッチ演算子**で、パターンマッチは言語の全域に浸透しています。Ruby は 3.0 で `case/in` によるパターンマッチが正式化され、かなり近い表現力を手に入れました。PHP は分割代入と `match` 式はあるものの、構造のマッチはできません。

| できること | PHP | Ruby | Elixir |
|---|---|---|---|
| 配列の分割代入 | `[$a, $b] = $arr` | `a, b = arr` | `[a, b] = list` |
| 残り付き分割 | — | `first, *rest = arr` | `[first \| rest] = list` |
| ハッシュ / マップの分割 | `['a' => $a] = $arr` | `case h in {name:}` | `%{name: name} = map` |
| ネスト構造のマッチ | — | `in {user: {name: String => n}}` | `%{user: %{name: name}} = data` |
| 値の一致を要求 | — | `in [1, *]` | `{:ok, user} = fetch()`（`:ok` 以外なら例外） |
| 既存変数の値でマッチ | — | `in ^expected` | `^expected = value` |
| ガード条件 | — | `in Integer => n if n > 0` | `when is_integer(n) and n > 0` |
| 関数定義でのマッチ | — | — | `def handle({:ok, val})` / `def handle({:error, msg})` |

書き比べるとこうなります。「成功なら値を取り出し、失敗ならメッセージを出す」処理です。

```elixir
# Elixir: 関数ヘッダで分岐する。if がいらない
def render({:ok, user}), do: "こんにちは #{user.name} さん"
def render({:error, :not_found}), do: "ユーザーが見つかりません"
def render({:error, reason}), do: "エラー: #{inspect(reason)}"
```

```ruby
# Ruby: case/in で同じ構造が書ける（3.0+）
def render(result)
  case result
  in [:ok, user]              then "こんにちは #{user.name} さん"
  in [:error, :not_found]     then "ユーザーが見つかりません"
  in [:error, reason]         then "エラー: #{reason.inspect}"
  end
end
```

```php
// PHP: match は「値の一致」だけなので、構造の分解は自分でやる
function render(array $result): string
{
    [$status, $payload] = $result;
    return match ($status) {
        'ok'    => "こんにちは {$payload->name} さん",
        'error' => $payload === 'not_found' ? 'ユーザーが見つかりません' : "エラー: {$payload}",
    };
}
```

Ruby の `case/in` は Elixir にかなり忠実な移植で、ハッシュの部分マッチ（`in {name:}` は他のキーがあってもマッチ）や `deconstruct_keys` による自作クラス対応など、実用性は十分です。ワンライナー用に、真偽値を返す `in` と、マッチ失敗で例外を投げる `=>`（rightward assignment）も使い分けられます。

```ruby
response => {status: 200, body:}   # 分解して body を取り出す。合わなければ例外
user in {role: :admin}             # true / false を返すのでガード節に便利
```

決定的に違うのは**関数定義そのものでマッチできるか**です。Elixir の関数クローズ（同名関数の複数定義）は「引数の構造で分岐する if 文の連鎖」を関数の頭に押し出したもので、再帰と組み合わせたときの見通しの良さは Ruby の `case/in` では代替しきれません。逆に言えば、Elixir のコードを読んでいて「この関数、定義が3つある」と感じたら、それは Ruby の `case` 文1つ分だと読み替えると頭に入りやすいはずです。

## 無名関数とブロック

コレクション操作の前に、それぞれの「関数を渡す」構文を並べます。

| | PHP | Ruby | Elixir |
|---|---|---|---|
| 完全形 | `function ($x) use ($y) { ... }` | `->(x) { ... }`（lambda） | `fn x -> ... end` |
| 短縮形 | `fn ($x) => $x * 2` | `{ it * 2 }`（3.4+ / `_1` も可） | `&(&1 * 2)` |
| 既存関数を渡す | `strlen(...)`（8.1+） | `&:upcase`（シンボル→ブロック） | `&String.upcase/1` |
| 呼び出し | `$f($x)` | `f.call(x)` / `f.(x)` | `f.(x)` |

Ruby だけ毛色が違うのは、第一級の関数オブジェクトより**ブロック**（`do ... end` / `{ ... }`）が主役だからです。メソッドは暗黙にブロックを1つ受け取れて、`yield` で呼び出せます。PHP と Elixir は「無名関数を引数として明示的に渡す」スタイルで統一されています。

Elixir のキャプチャ構文 `&` は最初は読みにくいですが、`&(&1 * 2)` は `fn x -> x * 2 end` の短縮、`&String.upcase/1` は「既存関数への参照」と、Ruby の `{ it * 2 }` と `&:upcase` にきれいに対応しています。なお `f.(x)` と呼び出しにドットが要るのは、名前付き関数と変数に束縛された関数を構文で区別する Elixir の仕様です。

## コレクション操作

日常で最も頻繁に書く部分です。Laravel は生の `array_*` 関数より Collection（`collect()`）を使うことが多いので併記します。

| やりたいこと | PHP / Laravel Collection | Ruby | Elixir |
|---|---|---|---|
| 変換 | `array_map($f, $a)` / `->map($f)` | `map` | `Enum.map/2` |
| 絞り込み | `array_filter($a, $f)` / `->filter($f)` | `select` / `filter` | `Enum.filter/2` |
| 畳み込み | `array_reduce($a, $f, $init)` / `->reduce()` | `reduce` / `inject` | `Enum.reduce/3` |
| 1件探す | `->first($f)` | `find` / `detect` | `Enum.find/2` |
| 平坦化しつつ変換 | `->flatMap($f)` | `flat_map` | `Enum.flat_map/2` |
| グループ化 | `->groupBy()` | `group_by` | `Enum.group_by/2` |
| 出現数を数える | `->countBy()` | `tally` | `Enum.frequencies/1` |
| n個ずつ区切る | `->chunk(n)` | `each_slice(n)` | `Enum.chunk_every/2` |
| 添字付き | `->map` + キー / `foreach` | `each_with_index` | `Enum.with_index/1` |
| 合計 | `array_sum($a)` / `->sum()` | `sum` | `Enum.sum/1` |

見た目は素直に対応しますが、注意点が3つあります。

**1. `reduce` のアキュムレータの位置が逆**

```ruby
# Ruby: ブロック引数は (アキュムレータ, 要素) の順
[1, 2, 3].reduce(0) { |acc, x| acc + x }
```

```elixir
# Elixir: 無名関数の引数は (要素, アキュムレータ) の順
Enum.reduce([1, 2, 3], 0, fn x, acc -> acc + x end)
```

Ruby → Elixir の乗り換えで全員が一度は踏む罠です。逆順に書いてもエラーにならず「答えがなんか変」になるのが厄介なところです。

**2. メソッドチェーンの代わりがパイプ演算子**

Elixir の関数はオブジェクトに生えていないので、チェーンの代わりに `|>`（第1引数に前の結果を流し込む）を使います。

```ruby
users.select(&:active?).map(&:name).tally
```

```elixir
users
|> Enum.filter(& &1.active?)
|> Enum.map(& &1.name)
|> Enum.frequencies()
```

ちなみに PHP 8.5 でもパイプ演算子 `|>` が入りました。`$result = $arr |> array_filter(...) |> array_values(...);` のように書けるようになり、`array_filter` と `array_map` で引数の順番が逆（コールバックが先か配列が先か）という長年のストレスを緩和する構文として歓迎されています。

**3. 遅延評価の入り口**

「無限とみなせる列から少しだけ取る」「巨大ファイルを1行ずつ処理する」ときの構えも対応しています。

| | PHP | Ruby | Elixir |
|---|---|---|---|
| 遅延コレクション | ジェネレータ（`yield`） | `lazy` | `Stream` モジュール |

```ruby
(1..Float::INFINITY).lazy.map { it * 2 }.first(3)   # => [2, 4, 6]
```

```elixir
Stream.iterate(1, &(&1 + 1)) |> Stream.map(&(&1 * 2)) |> Enum.take(3)   # => [2, 4, 6]
```

Elixir は「`Stream.*` で組み立てて、最後に `Enum.*` で実体化する」という役割分担が明快で、ファイル処理の `File.stream!/1` や Ecto の `Repo.stream/1` も同じインターフェースに乗っています。

## nil / null との付き合い方

| | PHP | Ruby | Elixir |
|---|---|---|---|
| 安全な呼び出し | `$user?->profile?->name`（8.0+） | `user&.profile&.name` | —（下記） |
| デフォルト値 | `$name ?? 'guest'` | `name \|\| 'guest'` | `name \|\| "guest"` ※ |
| ネストから安全に取得 | `$arr['a']['b'] ?? null` | `hash.dig(:a, :b)` | `get_in(map, [:a, :b])` |
| 偽と扱われる値 | `null`, `false`, `0`, `"0"`, `""`, `[]` など多数 | `nil` と `false` だけ | `nil` と `false` だけ |

Elixir に safe navigation 演算子（`&.`）がないのは意図的で、「nil かもしれない値を黙って伝播させず、`{:ok, value} / {:error, reason}` のタプルを返してパターンマッチで受ける」のがイディオムだからです。連続する「失敗するかもしれない処理」は `with` 式でまとめます。

```elixir
with {:ok, user} <- fetch_user(id),
     {:ok, profile} <- fetch_profile(user) do
  {:ok, profile.name}
else
  {:error, reason} -> {:error, reason}
end
```

Ruby で `&.` を3連発したくなる場面は、Elixir では「そもそも nil を返さない設計にする」方向に矯正される、と考えると思想が掴みやすいはずです。なお PHP の truthiness は表のとおり落とし穴が多く（`"0"` が偽！）、Ruby / Elixir の「`nil` と `false` 以外はすべて真」というシンプルさは移住先として安心できる仕様です。

## 制御構造は「式」か「文」か

Ruby と Elixir では `if` も `case` も**値を返す式**なので、結果を直接束縛できます。

```ruby
label = if score >= 80 then "合格" else "不合格" end
```

```elixir
label = if score >= 80, do: "合格", else: "不合格"
```

PHP の `if` は文なので値を返せず、式として書きたい場合は三項演算子か `match` 式を使います。細かい対応は次のとおりです。

| | PHP | Ruby | Elixir |
|---|---|---|---|
| if が式 | ×（`match` / 三項演算子で代用） | ○ | ○ |
| 否定形の分岐 | — | `unless` | `unless` は**非推奨**（`if` + 否定で書く） |
| 後置修飾 | — | `puts x if debug?` | — |
| 多分岐 | `match` / `switch` | `case/when`（値）・`case/in`（構造） | `case`（構造）・`cond`（条件列） |
| ループ構文 | `for` / `foreach` / `while` | `while` / `each`（メソッド） | **なし**（再帰と `Enum` で表現） |

Elixir に `for` 文や `while` 文が（ほぼ）ないのは初見で最も戸惑う点ですが、実務のループの9割は `Enum` 系関数で書けます。残りの「終了条件まで回し続ける」類は再帰か `Enum.reduce_while/3` で表現します。なお `for` というキーワード自体はリスト内包表記（comprehension）として存在し、`for x <- 1..3, do: x * 2` のように使いますが、これも「ループ文」ではなく値を返す式です。

## まとめ

- リテラルはほぼ1対1で対応するが、Elixir の**リストは連結リスト**、**アトムは GC されない**、**シングルクォートは charlist** という3点だけ足元が違う
- %記法とシジルは鏡写し。ただし**大文字・小文字の意味が逆**で、Elixir のシジルは**ユーザー定義可能**（`~p` / `~H` はその産物）
- パターンマッチは Elixir の中心機能で、Ruby 3.0+ の `case/in` は良質な移植。**関数ヘッダでのマッチ**だけは Elixir 固有
- コレクション操作は語彙がよく対応する。罠は **`reduce` の引数順が Ruby と逆**なことと、チェーンの代わりに `|>` を使うこと
- nil の扱いは「safe navigation で耐える」（PHP / Ruby）から「タプル + パターンマッチで設計から潰す」（Elixir）へ発想を切り替える

構文の対応が頭に入ると、フレームワークのドキュメントを読む速度が体感で変わります。シリーズの他の記事（特に思想編）と合わせて、行き来の摩擦を減らす一助になれば幸いです。

## 参考リンク

- [Elixir: Basic types](https://hexdocs.pm/elixir/basic-types.html) / [Sigils](https://hexdocs.pm/elixir/sigils.html) / [Pattern matching](https://hexdocs.pm/elixir/pattern-matching.html) / [Enum](https://hexdocs.pm/elixir/Enum.html) / [Stream](https://hexdocs.pm/elixir/Stream.html)
- [Ruby: Pattern matching（リファレンスマニュアル）](https://docs.ruby-lang.org/ja/latest/doc/spec=2fpattern_matching.html) / [Enumerable](https://docs.ruby-lang.org/ja/latest/class/Enumerable.html)
- [PHP: match](https://www.php.net/manual/ja/control-structures.match.php) / [アロー関数](https://www.php.net/manual/ja/functions.arrow.php) / [PHP 8.5 リリース情報](https://www.php.net/releases/8.5/en.php)
- [Laravel: Collections](https://laravel.com/docs/13.x/collections)
- [Phoenix: 検証済みルート `~p`](https://hexdocs.pm/phoenix/routing.html) / [HEEx `~H`](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html)

---

最初の記事から読み返す: **[Laravel・Rails・Phoenix 対応表: パッケージ管理 — Composer / Bundler / Mix](https://qiita.com/ak-inori/items/59ef9ff5937475c59d1b)**
