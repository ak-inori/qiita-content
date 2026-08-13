---
title: Laravel・Rails・Phoenix 対応表（7/8）定番ライブラリ — ORM・認証・ジョブ・リアルタイムまで
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - ecto
private: true
updated_at: '2026-08-13T13:42:25+09:00'
id: b8e0dbfeafc0ead6f4b0
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

Laravel や Rails で「これに使うライブラリはあれ」と即答できる定番の選択肢が、Phoenix では何に対応するのか（またはその逆）を、用途別の対応表とコード例でまとめます。ORM・テスト補助・HTTPクライアント・認証認可・非同期/リアルタイム・メール・コード品質の7カテゴリを扱います。

本記事は Laravel・Rails・Phoenix 対応表シリーズ（全8回）の第7回です。

対象バージョン（執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.5 | Ruby 4.0（Rails 8.1 の必須要件は 3.2+） | Elixir 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

先に全体の傾向を一言でいうと、**Phoenix は「標準・準標準でカバーされる範囲が広く、サードパーティのデファクトは各用途に1つに収束している」**エコシステムです。Laravel の「公式パッケージ群が厚い」、Rails の「gem の選択肢が豊富で組み合わせ自由」との対比を意識しながら読むと、表が頭に入りやすいはずです。

---

## 1. ORM・データベース

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| ORM / データマッパー | Eloquent | ActiveRecord | **Ecto** |
| マイグレーション | 標準 | 標準 | Ecto（標準同梱） |
| ページネーション | 標準（`paginate`） | kaminari / pagy | **Flop** / scrivener_ecto ※1 |
| シード | 標準 | 標準 | `priv/repo/seeds.exs` |

※1: 長らく定番だった scrivener_ecto は現在メンテナンスが低調（プルリクと重大バグのみ対応）です。新規なら、フィルタ・ソート・カーソルページネーションまで揃った **Flop**（+ LiveView 用コンポーネントの flop_phoenix）が第一候補です。

このカテゴリが3フレームワーク間で**一番書き味が変わる**ところです。Eloquent と ActiveRecord はどちらも Active Record パターン（モデルのインスタンスが保存やバリデーションの責務を持つ）ですが、Ecto は**データマッパー + チェンジセット方式**です。「モデルに `save` が生えている」世界観ではなく、`Repo.insert(changeset)` のように永続化の操作を明示します。

とはいえ、クエリを書く分には見た目はかなり近いです。「メールアドレスで1件取る」「公開済み記事をコメント付きで eager load する」を並べてみます。

```php
// Laravel (Eloquent)
$user = User::where('email', $email)->first();

$posts = Post::with('comments')
    ->where('published', true)
    ->get();
```

```ruby
# Rails (ActiveRecord)
user = User.where(email: email).first  # 実務では User.find_by(email: email)

posts = Post.includes(:comments).where(published: true)
```

```elixir
# Phoenix (Ecto)
import Ecto.Query

user = User |> where(email: ^email) |> Repo.one()
# ショートカット: Repo.get_by(User, email: email)

posts =
  Post
  |> where(published: true)
  |> preload(:comments)
  |> Repo.all()
```

Ecto で最初に戸惑うポイントを2つ挙げておきます。

- **`^`（ピン演算子）**: クエリ内で外部の変数を使うときは `^email` のように明示します。値は常にプリペアドステートメントのパラメータとして渡されるため、SQL インジェクションが構文レベルで起きにくい設計です
- **eager load は自動でクエリを見て判断しない**: `preload` を書かない限りアソシエーションはロードされず、ロードせずに触ると `Ecto.Association.NotLoaded` が返ります。「うっかり N+1」ではなく「うっかり例外」に倒してあるので、遅延ロード起因の N+1 がそもそも発生しません

### changeset の最小例

バリデーションはモデル（スキーマ）そのものではなく **changeset 関数**に書きます。

```elixir
defmodule Blog.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    timestamps()
  end

  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name])              # 受け付けるキーの許可リスト
    |> validate_required([:email])
    |> validate_format(:email, ~r/@/)
    |> unique_constraint(:email)                 # DB のユニーク制約違反を検出
  end
end
```

```elixir
# 使う側
%Blog.Accounts.User{}
|> Blog.Accounts.User.registration_changeset(params)
|> Repo.insert()
# => {:ok, %User{...}} または {:error, %Ecto.Changeset{}}
```

`cast/3` の許可リストが Rails の Strong Parameters / Laravel の `$fillable` に相当します。changeset 関数は用途別に何個でも作れるため、「登録時とプロフィール更新時で必須項目が違う」ようなコンテキスト依存のバリデーションが、条件分岐なしで自然に書き分けられます。ここが冗長さと引き換えに得られる一番のメリットです。

---

## 2. テスト補助

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| ファクトリ | Eloquent Factories（標準） | FactoryBot | ExMachina |
| ダミーデータ | FakerPHP（標準同梱） | faker | faker（Elixir 版） |
| モック/スタブ | Mockery / PHPUnit標準 | rspec-mocks / mocha | **Mox** ※ |
| HTTP モック | `Http::fake()` | WebMock / VCR | Bypass / **Req.Test** |
| E2E / ブラウザ | Laravel Dusk | Capybara + Selenium | Wallaby / PhoenixTest |

※ Mox は「ビヘイビア（インターフェース）を定義してモックする」方針で、モンキーパッチによる任意メソッドの差し替えは**できません**。`allow any instance of User to receive(:save)` のような書き方に慣れていると窮屈に感じますが、これは意図的な制約です。差し替えたい依存は最初からビヘイビアとして切り出し、実装をアプリ設定で注入する——という設計をテストツール側が強制してくる、と理解してください。

もうひとつの理由がテストの並列実行です。ExUnit は `async: true` でテストケースを並列実行するのが標準なので、グローバルにメソッドを書き換えるモンキーパッチ方式とは相性が悪く、プロセスごとにモックの期待値を隔離できる Mox の方式が採られています。

HTTP モックは、後述する Req を使っているなら **Req.Test**（Req 本体に同梱のスタブ機構）が最短です。外部プロセスとして本物の HTTP サーバーを立てる WebMock 的な体験が欲しい場合は Bypass を使います。

---

## 3. HTTP クライアント・API

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| HTTP クライアント | Http ファサード（Guzzle） | Faraday / HTTParty | **Req**（推奨）/ Tesla / Finch / HTTPoison（旧世代） |
| JSON シリアライズ | API Resources | jbuilder / Alba / AMS | JSON ビューモジュール + Jason |
| GraphQL | Lighthouse | graphql-ruby | Absinthe |

Elixir の HTTP クライアントは世代交代が激しかった領域ですが、現在は **Req** が事実上の推奨です（Phoenix 1.8 の新規プロジェクトにもデフォルトで入ります）。売りは「batteries-included」で、次がすべて**デフォルトで有効**です。

- レスポンスの自動 JSON デコード・自動解凍
- リダイレクト追従
- 一時的なエラー（接続失敗・HTTP 408/429/500/502/503/504 など）の**自動リトライ**

```elixir
Req.get!("https://api.github.com/repos/elixir-lang/elixir").body["stargazers_count"]
# => 25xxx  （デコード済みのマップがそのまま返る）

# ベース URL やリトライ回数を持つクライアントを組み立てて使い回す
req = Req.new(base_url: "https://api.example.com", max_retries: 5)
Req.get!(req, url: "/users")
```

内部の機能はすべて「ステップ」と呼ばれる関数の組み合わせでできており、Faraday や Guzzle のミドルウェアに相当する拡張がプラグインとして書けます。Laravel の `Http::retry()` や Faraday + faraday-retry で自前設定していた内容が、素の状態で入っている、と考えると位置づけが掴みやすいはずです。

JSON API のレスポンス生成は、Phoenix ではシリアライザライブラリを使わず、**JSON 用のビューモジュールに変換関数を書く**のが標準スタイルです（`mix phx.gen.json` が雛形を生成します）。jbuilder のようなテンプレートというより、Alba や API Resources で `toArray()` を書く感覚に近いです。

---

## 4. 認証・認可

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| 認証（フルスタック） | 公式スターターキット（Fortify ベース）※1 | Devise / Rails 8 標準認証ジェネレータ | **`mix phx.gen.auth`（標準）** |
| API トークン認証 | Sanctum | devise-jwt など | Guardian（JWT） |
| 認可 | Gates / Policies（標準） | Pundit / CanCanCan | Bodyguard / LetMe |

※1: 長年の定番だった Breeze / Jetstream は Laravel 12 でインストーラから外れ、React / Vue / Livewire の3種の公式スターターキット（内部は Fortify）に世代交代しています。

3エコシステム共通の潮流として、**「認証ライブラリを入れる」から「認証コードを生成して自分のアプリに持つ」への移行**が進んでいます。

**Phoenix: `mix phx.gen.auth`** はその代表で、User スキーマ・登録/ログイン/ログアウト・メール確認・パスワードリセット・「重要操作前の再認証（sudo モード）」までを、ライブラリ依存ではなく**自分のアプリのコードとして生成**します。Phoenix 1.8 では2つ大きな変更が入りました。

- **デフォルトが magic link（メールリンク）ログイン**になりました。パスワードは「後から任意で設定するもの」という位置づけで、登録時はメールアドレスだけ。結果としてメールアドレス確認が必ず済んでいる状態になります（パスワード方式にも変更可能です）
- **Scope** という仕組みが導入されました。「現在のユーザー」を包む構造体（`current_scope`）が生成され、**以後の `phx.gen.html` などのジェネレータが作る context 関数はすべて scope を引数に取る**ようになります。`Blog.list_posts(scope)` のように「誰のデータか」をクエリレベルで強制する、IDOR（他人のリソースへの直接アクセス）対策をデフォルトにする設計です

**Rails: Rails 8 の標準認証ジェネレータ**（`bin/rails generate authentication`）も同じ思想で、User / Session モデル、ログイン、パスワードリセットを生成します。ただし公式の位置づけは **Devise の置き換えではなく「土台の提供」**です。ユーザー登録画面すら生成されない最小構成なので、OAuth・2FA・アカウントロックなどまで欲しければ引き続き Devise（または生成コードへの自前実装）が現実解です。

**Laravel** はもともと Fortify（ヘッドレスな認証バックエンド）+ スターターキットという「生成型」に近い構成でしたが、スターターキット自体がパッケージではなく**プロジェクトテンプレート**になり、認証まわりのコードが最初から手元にある形になりました。

認可は Laravel だけが標準（Gates / Policies）で、Rails は Pundit、Phoenix は Bodyguard がそれぞれ「ポリシーオブジェクト方式」の定番です。Phoenix の場合、前述の Scope が浸透してくると「認可ライブラリなしで context 関数のクエリ自体が認可を兼ねる」形も増えていきそうです。

---

## 5. 非同期処理・リアルタイム

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| バックグラウンドジョブ | Queue（Horizon / Redis） | ActiveJob + Sidekiq / Solid Queue | **Oban**（DBベース） |
| 定期実行 | Scheduler（標準） | whenever / sidekiq-cron / Solid Queue の recurring | Oban.Plugins.Cron / Quantum |
| WebSocket | Reverb + Echo | Action Cable / AnyCable | **Phoenix Channels（標準組み込み）** |
| リアルタイムUI | Livewire（v4） | Hotwire（Turbo/Stimulus） | **LiveView（標準）** |
| プレゼンス管理 | — | — | Phoenix.Presence（標準） |

### バックグラウンドジョブ: Oban は「Redis のいらない Sidekiq」

Elixir はここが本領です。Sidekiq が Redis を要求するのに対し、**Oban はアプリのデータベースだけで動きます**（PostgreSQL が第一級。SQLite3 は Lite エンジン、MySQL 8.4+ は v2.19 で追加された Dolphin エンジンで対応）。BEAM の軽量プロセスのおかげでポーリングやジョブごとのプロセス起動が安く、「ジョブ基盤のためだけにミドルウェアを増やさない」構成が成立します。Rails 8 の Solid Queue（DB ベースをデフォルト化）も同じ方向を向いており、**「ジョブキューは Redis から DB へ」は3エコシステム共通のトレンド**といえます。

ジョブの定義と投入を3つ並べます。

```php
// Laravel
class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    public function __construct(public Podcast $podcast) {}

    public function handle(): void
    {
        // 処理本体
    }
}

ProcessPodcast::dispatch($podcast);
```

```ruby
# Rails (Sidekiq を直接使う場合)
class ProcessPodcastJob
  include Sidekiq::Job
  sidekiq_options queue: :default, retry: 3

  def perform(podcast_id)
    # 処理本体
  end
end

ProcessPodcastJob.perform_async(podcast.id)
```

```elixir
# Phoenix (Oban)
defmodule Blog.Workers.ProcessPodcast do
  use Oban.Worker, queue: :default, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"podcast_id" => id}}) do
    # 処理本体
    :ok
  end
end

%{podcast_id: podcast.id}
|> Blog.Workers.ProcessPodcast.new()
|> Oban.insert()
```

構造はほぼ一対一で対応しますが、注意点が2つあります。

- **args は JSON 経由**: Oban のジョブ引数は DB に JSON で保存されるため、`perform` に届くときはキーが**文字列**になります（`%{"podcast_id" => id}`）。Sidekiq の「引数はプリミティブだけにする」流儀と同じで、Laravel のようにモデルごと渡す（SerializesModels）ことはしません
- **ジョブ投入がトランザクションに乗る**: ジョブが DB レコードなので、`Ecto.Multi` で「レコード作成とジョブ投入を同一トランザクション」にできます。「コミット前にジョブが走ってレコードが見つからない」という Sidekiq あるあるが構造的に起きません

### リアルタイムUI: Livewire / Hotwire / LiveView の思想の違い

3つとも「SPA を作らずにリッチな UI を実現する」という目的は同じですが、**状態をどこに置き、何を配信するか**が根本的に違います。

**Livewire は「ステートフルなコンポーネントを HTTP で再現する」**アプローチです。コンポーネントの状態はリクエストごとにシリアライズしてクライアントに埋め込み、操作のたびに AJAX でサーバーに送り返して再構築・再レンダリングします。つまり本質はステートレスな HTTP の往復ですが、開発者からは「サーバー側に生きているコンポーネント」に見える、という抽象です。v4（2026年1月リリース）ではロジックとテンプレートを1ファイルにまとめる単一ファイルコンポーネントが導入され、コンポーネント指向がさらに強まりました。

**Hotwire は「HTML を配信する」ことに徹し、ステートフルなコンポーネントという概念を持ちません**。Turbo がページ遷移・部分置換（Frames）・サーバープッシュ（Streams）を HTML 断片の配信で実現し、こぼれた分だけ Stimulus で最小限の JS を書きます。サーバーは従来どおりのステートレスな MVC のままで、コントローラとビューの書き方がほぼ変わらないのが最大の美点です。3つの中では最も「疎結合で薄い」選択肢といえます。

**LiveView は「本当にステートフル」**です。クライアントごとに BEAM 上のプロセスが1つ立ち、状態はそのプロセスのメモリに生き続け、WebSocket 越しに**差分だけ**が push されます。Livewire が HTTP 往復でエミュレートしているものを、LiveView はランタイムの性質（数十万の軽量プロセスを平気で維持できる）を使って文字どおり実装している、という関係です。状態の再構築コストがないぶんインタラクションは軽く、サーバー起点の更新（他ユーザーの操作の反映など）も自然に書けます。一方で「接続が切れると状態も消える」「ステートフルなプロセス群を運用する」という LiveView 固有の考慮も生まれます。

Phoenix.Presence（「誰がオンラインか」の分散管理）に相当する標準機能が他の2つにないのも、この延長です。接続＝プロセスというモデルだからこそ、在席管理がフレームワーク標準になっています。

---

## 6. メール・その他

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| メール送信 | Mailable + Mail | Action Mailer | **Swoosh**（標準同梱） |
| 開発時のメール確認 | Mailpit など | letter_opener | Swoosh Local Adapter（`/dev/mailbox` 標準） |
| 画像処理 | Intervention Image | image_processing（libvips） | image（Vix / libvips） |
| i18n | 標準（lang/） | rails-i18n（標準） | Gettext（標準同梱） |
| 環境変数 | .env（標準） | dotenv-rails / credentials | `config/runtime.exs` + dotenvy |
| 監視ダッシュボード | Telescope | rack-mini-profiler 等 | **Phoenix LiveDashboard（標準）** |
| エラートラッキング | Sentry / Bugsnag | Sentry / Bugsnag | Sentry（sentry-elixir） |

いくつか補足します。

- **メール**: Swoosh は Phoenix の新規プロジェクトに最初から入っており、開発環境では追加ツールなしで `http://localhost:4000/dev/mailbox` に送信メールが溜まります。Mailpit や letter_opener を入れる手間が省略されている格好です
- **環境変数**: Elixir には「.env を読む標準機構」がなく、代わりに**起動時に評価される `config/runtime.exs`** で `System.get_env/2` を呼ぶのが公式の作法です。ローカルで .env ファイルが欲しい場合に dotenvy を足します。Rails の credentials のような暗号化シークレット管理は標準にはありません
- **監視**: LiveDashboard は LiveView 製のダッシュボードで、リクエストログ・プロセス一覧・Ecto クエリの統計・BEAM のメモリ状況までブラウザで見られます。Telescope 相当が標準で付いてくる、と考えてください

---

## 7. コード品質

| 用途 | Laravel | Rails | Phoenix |
|---|---|---|---|
| フォーマッタ | Pint（公式） | RuboCop（-a）/ standard | **`mix format`（言語標準）** |
| リンタ / スタイル | PHP_CodeSniffer | RuboCop | **Credo** |
| 静的解析 / 型 | PHPStan（Larastan）/ Psalm | Sorbet / Steep + RBS | Dialyzer（Dialyxir） |
| pre-commit 一括 | 各自構成 | 各自構成 | `mix precommit` エイリアス（Phoenix 1.8 生成） |

RuboCop 的な立ち位置に一番近いのは **Credo**（コードスメル・スタイル・リファクタリング機会の検出）ですが、整形は言語公式の `mix format` が担うため、**「フォーマット論争がそもそも存在しない」**のが Elixir の特徴です。gofmt と同じ思想で、設定項目は行幅くらいしかありません。

型まわりは過渡期です。Dialyzer（成功型付けによる実行前チェック）が従来の定番ですが、Elixir 1.17 以降**言語本体に漸進的な型システム（set-theoretic types）が段階的に組み込まれており**、1.20 時点ではコンパイラ自身がかなりの型エラーを警告として検出します。「Sorbet / PHPStan を導入する」に相当する作業が、将来的には「コンパイラを最新にする」に置き換わっていく見込みです。

---

## 8. ライブラリの探し方

対応表にない用途のライブラリを探すときの定番ルートも並べておきます。

| | PHP | Ruby | Elixir |
|---|---|---|---|
| レジストリ検索 | [Packagist](https://packagist.org) | [RubyGems](https://rubygems.org) | [Hex.pm](https://hex.pm) |
| 比較・目利きサイト | — | [Ruby Toolbox](https://www.ruby-toolbox.com) | [Elixir Toolbox](https://elixir-toolbox.dev) |
| キュレーションリスト | Awesome PHP / Awesome Laravel | Awesome Ruby | [Awesome Elixir](https://github.com/h4cc/awesome-elixir) |
| CLI から情報を見る | `composer show foo/bar` | `gem info foo` | `mix hex.info foo` |

Elixir でライブラリを目利きするときのポイントを3つ。

- **Hex.pm の数字を見る**: パッケージページに最近のダウンロード数と依存されている数が出ます。Ruby Toolbox 的な「カテゴリ内の勢力図」は Elixir Toolbox や Awesome Elixir で掴めます
- **hexdocs の充実度がそのまま品質シグナル**: Hex は全パッケージのドキュメントを [hexdocs.pm](https://hexdocs.pm) に自動ホストするため、「ドキュメントサイトが別途あるか探す」必要がありません。逆に hexdocs がスカスカなパッケージは避ける判断材料になります
- **「更新が止まっている＝死んでいる」とは限らない**: エコシステムが小さく安定している分、完成して更新が不要になったパッケージも多いです。最終コミット日だけでなく、issue への反応と Elixir Forum での言及を見るのがおすすめです

---

## まとめ

- **ORM が最大の思想差**。Eloquent / ActiveRecord の「モデル中心」から、Ecto の「changeset でデータ変換を明示する」世界へ。preload 必須の設計により N+1 は例外で顕在化します
- **認証は3者とも「生成型」へ収束中**。`phx.gen.auth`（1.8 で magic link + Scope がデフォルト）、Rails 8 標準認証ジェネレータ、Laravel の公式スターターキット
- **ジョブキューは「Redis から DB へ」が共通トレンド**。Oban はその最右翼で、トランザクションにジョブ投入を同居できるのが強み
- **リアルタイム UI は状態の置き場所で三者三様**。HTTP でステートフルを再現する Livewire、あくまで HTML 配信に徹する Hotwire、プロセスとして本当にステートフルな LiveView
- Phoenix は Swoosh・LiveDashboard・Channels など**標準同梱の守備範囲が広く、外部ミドルウェアとライブラリ選定の両方が減る**方向に設計されています

「あれって Phoenix だと何だっけ」の検索時間を、この表が減らせれば幸いです。

---

## 参考リンク

- [Ecto](https://hexdocs.pm/ecto/Ecto.html) / [Ecto.Changeset](https://hexdocs.pm/ecto/Ecto.Changeset.html) / [Flop](https://hexdocs.pm/flop/readme.html)
- [Req](https://hexdocs.pm/req/readme.html) / [Mox](https://hexdocs.pm/mox/Mox.html) / [ExMachina](https://hexdocs.pm/ex_machina/readme.html)
- [mix phx.gen.auth](https://hexdocs.pm/phoenix/mix_phx_gen_auth.html) / [Phoenix 1.8.0 リリースノート](https://phoenixframework.org/blog/phoenix-1-8-released)
- [Rails 8 認証ジェネレータ（Rails Guides: Securing Rails Applications）](https://guides.rubyonrails.org/security.html) / [Devise](https://github.com/heartcombo/devise)
- [Oban](https://hexdocs.pm/oban/Oban.html) / [Sidekiq](https://github.com/sidekiq/sidekiq) / [Laravel Queues](https://laravel.com/docs/13.x/queues)
- [Phoenix LiveView](https://hexdocs.pm/phoenix_live_view/welcome.html) / [Hotwire](https://hotwired.dev/) / [Livewire](https://livewire.laravel.com/)
- [Swoosh](https://hexdocs.pm/swoosh/Swoosh.html) / [Phoenix LiveDashboard](https://hexdocs.pm/phoenix_live_dashboard/Phoenix.LiveDashboard.html)
- [Credo](https://hexdocs.pm/credo/overview.html) / [Laravel Pint](https://laravel.com/docs/13.x/pint) / [RuboCop](https://rubocop.org/)
- [Hex.pm](https://hex.pm) / [Ruby Toolbox](https://www.ruby-toolbox.com) / [Packagist](https://packagist.org)
