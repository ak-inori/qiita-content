---
title: 'Laravel・Rails・Phoenix 対応表（6/8）テスト — PHPUnit・Pest / Minitest・RSpec / ExUnit'
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - テスト
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

Laravel / Rails 経験者が Phoenix に入門するとき（またはその逆）、テストまわりは「概念はほぼ同じなのに語彙が全部違う」領域です。本記事では PHPUnit・Pest / Minitest・RSpec / ExUnit の対応関係を、アサーション・setup・DB分離戦略・タグ実行・flaky対策・CI高速化まで掘り下げて整理します。

本記事は Laravel・Rails・Phoenix 対応表シリーズ（全8回）の第6回です。

対象バージョン（執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.5 | Ruby 4.0（Rails 8.1 の必須要件は 3.2+） | Elixir 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

---

## 全体対応表

| | PHP / Laravel | Ruby / Rails | Elixir |
|---|---|---|---|
| 標準/デファクト | PHPUnit（**Pest** も公式級） | Minitest（Rails 標準）/ **RSpec**（デファクト） | **ExUnit**（言語標準・ほぼ一択） |
| 実行 | `php artisan test` | `bin/rails test` / `bundle exec rspec` | `mix test` |
| ファイル指定 | `php artisan test tests/Feature/FooTest.php` | `rails test test/models/foo_test.rb` | `mix test test/blog/foo_test.exs` |
| 行指定 | `--filter`（メソッド名） | `rails test test/foo_test.rb:42` / `rspec spec/foo_spec.rb:42` | `mix test test/foo_test.exs:42` |
| 前回失敗したテスト | `--order-by=defects`（失敗分を先に実行） | `rspec --only-failures`（要設定） | **`mix test --failed`（標準装備）** |
| 並列実行 | `php artisan test --parallel`（paratest） | `parallelize`（Rails 標準） | `async: true` の test case を並列実行 |
| カバレッジ | `--coverage`（Xdebug/PCOV） | simplecov | `mix test --cover` / excoveralls |
| watch モード | phpunit-watcher | guard | mix_test_watch |
| 遅いテストの特定 | `php artisan test --profile` | `rspec --profile` | `--slowest N`（ExUnit） |

ExUnit は「言語に最初から入っているテストフレームワークが十分強いので、RSpec / Pest のような対抗馬が育たなかった」タイプです。DSL は Minitest 寄り（`assert` ベース）ですが、`describe` によるグルーピングや setup コールバックなど RSpec 的な構造化もできます。

---

## 三面図: 同じテストを3つの書き方で

### モデル / コンテキストのテスト（ユーザー登録）

**Laravel（Pest）** — `tests/Feature/RegisterUserTest.php`:

```php
<?php

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

describe('register user', function () {
    it('creates a user with valid attrs', function () {
        $user = User::create(['email' => 'a@example.com']);

        expect($user->email)->toBe('a@example.com')
            ->and($user->exists)->toBeTrue();
    });
});
```

**Rails（RSpec）** — `spec/models/user_spec.rb`:

```ruby
RSpec.describe User, type: :model do
  describe "registration" do
    it "creates a user with valid attrs" do
      user = User.new(email: "a@example.com")

      expect(user.save).to be true
      expect(user.email).to eq "a@example.com"
    end
  end
end
```

**Phoenix（ExUnit）** — `test/blog/accounts_test.exs`:

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

Phoenix だけ「モデルのテスト」ではなく「コンテキスト（`Accounts`）という公開APIのテスト」になっている点が思想の違いです。Ecto のスキーマ自体はただのデータ構造なので、テストの主役は関数になります。

### HTTP リクエストのテスト

**Laravel（Pest / Feature テスト）**:

```php
it('creates a user via API', function () {
    $response = $this->postJson('/api/users', ['email' => 'a@example.com']);

    $response->assertCreated()
             ->assertJsonPath('data.email', 'a@example.com');

    $this->assertDatabaseHas('users', ['email' => 'a@example.com']);
});
```

**Rails（RSpec / request spec）**:

```ruby
RSpec.describe "Users API", type: :request do
  it "creates a user" do
    expect {
      post "/api/users", params: { user: { email: "a@example.com" } }
    }.to change(User, :count).by(1)

    expect(response).to have_http_status(:created)
    expect(response.parsed_body.dig("data", "email")).to eq "a@example.com"
  end
end
```

**Phoenix（ExUnit / ConnCase）**:

```elixir
defmodule BlogWeb.UserControllerTest do
  use BlogWeb.ConnCase, async: true

  test "POST /api/users creates a user", %{conn: conn} do
    conn = post(conn, ~p"/api/users", user: %{email: "a@example.com"})

    assert %{"data" => %{"email" => "a@example.com"}} = json_response(conn, 201)
    assert Blog.Repo.get_by(Blog.Accounts.User, email: "a@example.com")
  end
end
```

3つとも「実際のHTTPスタックを通してレスポンスとDBを検証する」点は同じです。Phoenix の `~p` は検証付きルートで、存在しないパスを書くとコンパイル時に警告が出ます。`%{conn: conn}` は後述する setup の context 渡しです。

---

## アサーション対応表

| 意味 | PHPUnit | RSpec | ExUnit |
|---|---|---|---|
| 等しい | `assertSame($e, $a)` / `assertEquals` | `expect(a).to eq(e)` | `assert a == e` |
| 真である | `assertTrue($x)` | `expect(x).to be_truthy` | `assert x` |
| 偽/否定 | `assertFalse` / `assertNot*` | `expect(x).not_to ...` | `refute x` |
| nil / null | `assertNull($x)` | `expect(x).to be_nil` | `assert is_nil(x)` |
| 含む | `assertContains` | `expect(list).to include(x)` | `assert x in list` |
| 例外が起きる | `$this->expectException(Foo::class)` | `expect { ... }.to raise_error(Foo)` | `assert_raise Foo, fn -> ... end` |
| 浮動小数の近似 | `assertEqualsWithDelta` | `be_within(0.01).of(x)` | `assert_in_delta a, e, 0.01` |
| メッセージ受信 | — | — | **`assert_receive {:done, _}`** |

ExUnit のアサーションは実質 `assert`（と `refute`）の2つだけです。「マッチャーの語彙を覚える」のではなく、素の比較式を書けば `assert` マクロが式を分解して詳細な diff を表示してくれます。Minitest の `assert_equal` 的な世界観をマクロで洗練させたもの、と捉えると近いです。

**最強のアサーションはパターンマッチ**です。

```elixir
assert {:ok, %User{email: "a@example.com", confirmed_at: nil}} =
         Accounts.register_user(%{email: "a@example.com"})
```

この1行で「成功していること」「返り値が `User` 構造体であること」「email の値」「未確認状態であること」を同時に検証しています。Laravel の `assertJsonPath` や RSpec の `have_attributes` で個別に検証する内容が、言語機能そのもので書けるのが ExUnit の書き味です。マッチに失敗すると、左辺のパターンと右辺の実際の値の diff が色付きで表示されます。

もうひとつ Elixir 固有なのが `assert_receive` / `assert_received`。プロセス間のメッセージパッシングを検証するためのもので、非同期処理やPubSubのテストで多用します。

---

## setup の対応

| | PHPUnit | RSpec | ExUnit |
|---|---|---|---|
| 各テストの前 | `setUp()` | `before(:each)` | `setup` |
| クラス/モジュールで1回 | `setUpBeforeClass()` | `before(:all)` | `setup_all` |
| 各テストの後 | `tearDown()` | `after(:each)` | `on_exit`（setup 内で登録） |
| 遅延評価フィクスチャ | — | **`let` / `let!`** | —（context で明示的に渡す） |
| テストへのデータ受け渡し | インスタンスプロパティ | `let` / インスタンス変数 | **context（マップ）** |

RSpec の `let` は「参照されるまで評価されない・同一 example 内ではメモ化される」のが特徴で、これに直接対応するものは PHPUnit / ExUnit にはありません。ExUnit は逆に**暗黙の状態を持たない**方向に振っていて、setup が返したマップ（context）をテスト側がパターンマッチで明示的に受け取ります。

```elixir
defmodule Blog.PostsTest do
  use Blog.DataCase, async: true

  setup do
    user = insert(:user)            # ExMachina のファクトリ
    on_exit(fn -> IO.puts("後片付けはここ") end)
    %{user: user}                   # ← context に積む
  end

  setup :create_post                # 名前付き setup（関数を合成できる）

  test "lists posts for the user", %{user: user, post: post} do
    assert Posts.list_posts(user) == [post]
  end

  defp create_post(%{user: user}) do
    %{post: insert(:post, author: user)}
  end
end
```

「テスト関数のシグネチャを見れば、そのテストが何に依存しているか全部わかる」のが利点です。RSpec で `let` が何段も継承されて出所を探し回る、あの体験の対極にあります。なお `setup_all` はモジュールごとに別プロセスで1回だけ実行されるため、DBサンドボックス（後述）のコネクションは使えない点に注意してください。

---

## DB分離戦略 — ここが一番思想が違う

「テストごとにDBをきれいな状態に戻す」問題への答えが三者三様です。

### Laravel: トレイトで戦略を選ぶ

| トレイト | 動き |
|---|---|
| `RefreshDatabase` | 最初に1回マイグレーション → 各テストをトランザクションで包んでロールバック |
| `LazilyRefreshDatabase` | 同上だが、実際にDBへ触れるテストまで初期化を遅延 |
| `DatabaseMigrations` | 毎テストでマイグレーションをやり直す（遅い） |
| `DatabaseTransactions` | トランザクションのみ（DBは事前に自分で用意） |

並列実行（`php artisan test --parallel`、要 paratest）では、**プロセスごとに `your_db_test_1`, `your_db_test_2` … とDBを複製**して分離します。作ったDBは次回のために残り、スキーマを変えたら `--recreate-databases` で作り直します。プロセスごとのシード投入などは `ParallelTesting::setUpTestDatabase()` フックで行います。

### Rails: トランザクション + プロセスごとのDB複製

Rails は標準で各テストをトランザクションで包み、終了時にロールバックします（`use_transactional_tests`、デフォルト有効）。並列化は `test_helper.rb` に生成される次の1行で、**プロセスを fork してワーカーごとにDBを複製**します（DB名に番号サフィックス）。

```ruby
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
end
```

`parallelize(workers: 4, with: :threads)` とするとスレッド並列（DBは1つを共有）にもできます。ワーカー数は `PARALLEL_WORKERS` 環境変数でも上書き可能です。RSpec には標準の並列機構がなく、parallel_tests gem などを併用します。

### Ecto: SQL Sandbox — DBを複製せずに並列化する

Elixir の答えは **`Ecto.Adapters.SQL.Sandbox`** です。発想が根本から違います。

- コネクションプールの各コネクションを**トランザクションで包んだ状態で貸し出す**
- 各テストプロセスがコネクションを1本 checkout し、**所有（ownership）**する
- テスト終了時にロールバック。**DBは1つのまま、テストはコネクション単位で分離**される

Laravel / Rails が「プロセス並列 × DB複製」で分離するのに対し、Ecto は「BEAM の軽量プロセス並列 × コネクション所有権」で分離します。DBの複製もプロセスの fork も不要で、`use Blog.DataCase, async: true` と書くだけで**DBを触るテストがモジュール単位で並列実行**されます。

Phoenix が生成する `DataCase` / `ConnCase` の中身はこうなっています。

```elixir
setup tags do
  pid = Ecto.Adapters.SQL.Sandbox.start_owner!(Blog.Repo, shared: not tags[:async])
  on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
  :ok
end
```

- `async: true` → テストごとに専用コネクション。完全並列
- `async: false` → **shared モード**。全プロセスが同じコネクションを共有する代わりに直列実行

テスト内で `Task.async` や LiveView のように**別プロセス**がDBに触る場合、そのプロセスにはコネクションの所有権がありません。shared モードにするか、`Ecto.Adapters.SQL.Sandbox.allow/3` で明示的に許可を渡します。Phoenix 入門者が最初に踏む `DBConnection.OwnershipError` はほぼこれが原因です。

注意点として、**Sandbox の並列実行を安全に使えるのは PostgreSQL のみ**です。MySQL ではトランザクションの実装上デッドロックが起きうるため、公式ドキュメントが並列テストを明確に非推奨としています。また並列になるのは `async: true` の test case（モジュール）同士で、同じモジュール内のテストは直列です。ユニーク制約のあるカラム（email など）は、並列時の衝突を避けるためファクトリで連番やランダム値を使うのが定石です。

---

## タグとフィルタ実行

「遅いテスト・外部APIを叩くテストを普段はスキップしたい」の対応です。

| | PHPUnit / Pest | RSpec | ExUnit |
|---|---|---|---|
| 付け方 | `#[Group('slow')]` 属性 / Pest は `->group('slow')` | メタデータ `it "...", :slow do` | `@tag :slow` |
| モジュール一括 | クラスに `#[Group]` | `describe "...", :slow do` | `@moduletag` / `@describetag` |
| 指定して実行 | `--group slow` | `--tag slow` | `mix test --only slow` |
| 除外して実行 | `--exclude-group slow` | `--tag ~slow` | `mix test --exclude slow` |
| デフォルト除外の設定 | `phpunit.xml` の `<groups>` | `config.filter_run_excluding :slow` | `test_helper.exs` で `ExUnit.start(exclude: [:external])` |

ExUnit のタグには2つ独自色があります。まず**タグは値を持てて、setup の context に流れ込みます**。

```elixir
@tag timeout: :timer.minutes(2)   # このテストだけタイムアウト延長
@tag :capture_log                 # ログ出力をキャプチャして失敗時のみ表示
test "slow external call" do ... end
```

前節の `DataCase` が `tags[:async]` を読んで sandbox のモードを切り替えていたように、「タグ → setup → テスト」とメタデータが一本のパイプラインでつながっているのが ExUnit の設計です。デフォルト除外したタグは `mix test --include external` で一時的に呼び戻せます。

なお Minitest には標準のタグ機構がなく、`-n /pattern/` の名前フィルタで代用するのが基本です（この点は RSpec が明確に優位です）。

---

## doctest — ドキュメントがそのままテストになる

Elixir 固有の武器が doctest です。関数の `@doc` に書いた IEx セッション例が、**そのままテストとして実行されます**。

```elixir
defmodule Blog.Slug do
  @doc """
  タイトルを URL スラッグに変換します。

      iex> Blog.Slug.slugify("Hello World")
      "hello-world"

      iex> Blog.Slug.slugify("")
      ""
  """
  def slugify(title) do
    title |> String.downcase() |> String.replace(" ", "-")
  end
end
```

```elixir
defmodule Blog.SlugTest do
  use ExUnit.Case, async: true
  doctest Blog.Slug        # ← この1行で @doc 内の iex> 例が全部テストになる
end
```

実装を変えてドキュメントの更新を忘れると**テストが落ちる**ので、「ドキュメントのコード例が嘘をつかない」ことが機械的に保証されます。hexdocs のライブラリドキュメントに正確な実行例が多いのは、この文化によるものです。PHP の phpDocumentor や YARD のコード例は実行されないので、直接の対応物はありません（Python の doctest が輸入元です）。純粋関数のエッジケース列挙は doctest に、副作用のあるロジックは通常のテストに、と使い分けるのが定石です。

---

## 実行順序ランダム化・seed・flaky 対策

テスト間の暗黙の依存（実行順序に依存するテスト）を洗い出すための機構です。

| | PHPUnit | Minitest / RSpec | ExUnit |
|---|---|---|---|
| ランダム実行 | `--order-by=random`（**opt-in**） | Minitest: デフォルト / RSpec: `--order random` | **デフォルト** |
| seed 指定して再現 | `--random-order-seed <N>` | `--seed <N>` | `mix test --seed <N>` |
| 定義順で実行 | デフォルトがそれ | — | `--seed 0` |
| flaky の炙り出し | — | `rspec --bisect`（最小再現の探索） | `--repeat-until-failure <N>` |
| 失敗で打ち切り | `--stop-on-failure` | `--fail-fast` | `--max-failures <N>` |

三者で立ち位置が逆なのが面白いところで、**PHPUnit だけがランダム実行 opt-in**（デフォルトは宣言順）、Minitest と ExUnit はデフォルトでランダムです。ExUnit は失敗時に必ず seed が表示されるので、`mix test --seed 12345` でその順序を完全再現できます。

flaky 対策の専用装備も個性が出ます。RSpec の `--bisect` は「どのテストの組み合わせで落ちるか」を二分探索してくれる名機能。ExUnit の `--repeat-until-failure 10000` は同一VM内でスイートを失敗するまで繰り返すもので、「たまにしか落ちない」テストの再現に使います。

```bash
mix test test/blog/flaky_test.exs --repeat-until-failure 1000
```

---

## CI 高速化の対比

| 目的 | Laravel | Rails | Phoenix |
|---|---|---|---|
| 並列実行 | `--parallel --processes=N` | `parallelize` / `PARALLEL_WORKERS=N` | `async: true`（デフォルトで並列） |
| マシンをまたぐ分割 | paratest の機能や CI 側で分割 | CI 側で分割 | **`--partitions N`** + `MIX_TEST_PARTITION` |
| 失敗したものだけ再実行 | `--order-by=defects`（要リザルトキャッシュ） | `rspec --only-failures` / `--next-failure` ※ | `mix test --failed` |
| 変更に関係する分だけ | — | — | **`mix test --stale`** |
| 依存キャッシュ | `~/.composer/cache` + `vendor` | Bundler キャッシュ + bootsnap | `deps` + `_build` をキャッシュ |

※ RSpec の `--only-failures` は `config.example_status_persistence_file_path` の設定が必要です。

Elixir 側で特筆すべきは2つ。`--partitions 4` はテストファイルをラウンドロビンで4分割し、CI の4ジョブに `MIX_TEST_PARTITION=1..4` を渡すだけでマシン並列が組める標準機能です。`--stale` は前回実行時からの**モジュール依存グラフの変化**を追跡し、影響のあるテストファイルだけを実行します（ローカルの反復開発向け。CI では全件実行が基本です）。

一方、コンパイル言語ゆえの注意点として、Elixir の CI では `_build` ディレクトリのキャッシュが効かないとコンパイル時間が支配的になります。`mix deps.get` と依存のコンパイル結果は必ずキャッシュしましょう。あわせて `mix test --warnings-as-errors` を CI に入れて、警告の混入をブロックするのが Phoenix コミュニティの定番構成です。

---

## まとめ

- **語彙の対応**: `setUp` / `before`・`let` / `setup`+context、`#[Group]` / `:tag` / `@tag`、`assertEquals` / `eq` / `assert ==` — 概念はほぼ1対1で対応する
- **DB分離だけは思想が違う**: Laravel / Rails は「プロセス並列 × DB複製」、Ecto は「軽量プロセス × SQL Sandbox のコネクション所有権」。`async: true` と書くだけでDBテストが並列になる代わりに、別プロセスの所有権（`allow/3` / shared モード）という新しい概念を1つ学ぶ必要がある
- **ExUnit 固有の武器**: パターンマッチによるアサーション、doctest、`--failed` / `--stale` / `--partitions` / `--repeat-until-failure` の標準装備。フレームワーク選定で迷う余地がない分、道具は言語側に厚く積まれている

「RSpec の表現力」「Pest の書き味」に相当する糖衣は ExUnit にはありませんが、パターンマッチと context の明示渡しがその穴を別の方向から埋めている、というのが行き来してみた実感です。

---

## 参考リンク

- [Laravel 13.x Testing: Getting Started](https://laravel.com/docs/13.x/testing) / [Database Testing](https://laravel.com/docs/13.x/database-testing)
- [PHPUnit CLI Options](https://docs.phpunit.de/en/12.5/cli-options.html) / [Pest](https://pestphp.com/)
- [Rails ガイド: Rails テスティングガイド](https://guides.rubyonrails.org/testing.html)
- [RSpec ドキュメント](https://rspec.info/documentation/)
- [ExUnit (Hexdocs)](https://hexdocs.pm/ex_unit/ExUnit.html) / [mix test](https://hexdocs.pm/mix/Mix.Tasks.Test.html)
- [Ecto.Adapters.SQL.Sandbox](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.Sandbox.html)
- [ExUnit.DocTest](https://hexdocs.pm/ex_unit/ExUnit.DocTest.html)
