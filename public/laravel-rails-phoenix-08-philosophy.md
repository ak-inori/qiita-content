---
title: Laravel・Rails・Phoenix 対応表（8/8）3つのエコシステムの思想の違い
tags:
  - Laravel
  - Rails
  - Phoenix
  - Elixir
  - 設計
private: true
updated_at: '2026-08-13T13:42:26+09:00'
id: e716638c92c3fba4e10e
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

コマンドやライブラリの対応表を眺めていると、「対応する行はあるのに、なぜか同じものに見えない」箇所がいくつも出てきます。`bundle exec` に相当するコマンドが Elixir に存在しない。Sidekiq に相当する Oban が Redis を要求しない。`$user->save()` に相当するはずの操作が `Repo.insert(changeset)` という別の形をしている。

こうした差は個別の設計判断ではなく、**エコシステムの根っこにある思想の違いが表面に染み出したもの**です。本記事では Laravel・Rails・Phoenix の「感触」の違いを、①ツールチェーンの構成、②ランタイムのプロセスモデル、③データの扱い方、の3つの軸で掘り下げます。本記事は Laravel・Rails・Phoenix 対応表シリーズ（全8回）の第8回です。

対象バージョン（執筆時点）:

| | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 言語 | PHP 8.5 | Ruby 4.0（Rails 8.1 の必須要件は 3.2+） | Elixir 1.20 |
| フレームワーク | Laravel 13 | Rails 8.1 | Phoenix 1.8 |

---

## 1. ツールチェーンが一枚岩であるということ

### 3エコシステムの役割分担

同じ「日常の開発作業」を支えるツールを役割ごとに並べると、構成の違いがはっきり見えます。

| 役割 | PHP / Laravel | Ruby / Rails | Elixir / Phoenix |
|---|---|---|---|
| 依存管理 | Composer | Bundler（+ RubyGems） | **Mix** |
| タスクランナー | artisan（+ composer scripts） | Rake / `bin/rails` | **Mix** |
| コード生成 | `artisan make:*` | `rails g` | `mix phx.gen.*` |
| テスト実行 | `artisan test` | `rails test` / `rspec` | `mix test` |
| フォーマッタ | Pint（公式別ツール） | RuboCop / standard（gem） | **`mix format`（言語標準）** |
| リリースビルド | —（デプロイツール任せ） | —（同左） | `mix release`（言語標準） |
| ロック準拠の実行 | `vendor/bin/xxx` | `bundle exec xxx` | **不要（常に lock 準拠）** |
| ツールの供給元 | 言語コミュニティ + フレームワーク | gem + フレームワーク | **すべて言語本体** |

PHP と Ruby では、依存管理（Composer / Bundler）は言語本体と別プロジェクトとして発展し、タスクランナー（artisan / Rake）はフレームワークや gem が担う、という**分業体制**です。歴史的にツールが後付けで積み重なってきたため、それぞれに設定ファイル・実行方法・流儀があります。`bundle exec` という「Gemfile.lock に準拠した環境でコマンドを実行するためのラッパー」が必要なのは、RubyGems（グローバル）と Bundler（プロジェクトローカル）という2層が併存しているからです。

Elixir では **Mix ひとつがビルド・依存管理・タスク実行・テスト・整形・リリースまで全部を担います**。Mix は言語本体に同梱されているため、「どのフォーマッタを使うか」「タスクランナーは何にするか」という選定の余地が意図的に消されています。`mix xxx` は常に `mix.lock` 準拠で動くので `bundle exec` 相当も不要です。整形ルールも言語公式の `mix format` が一元管理するため、RuboCop の `.rubocop.yml` を巡るチーム内論争のようなものが**そもそも発生しません**。

トレードオフもあります。Mix は「コアは薄く」という方針のため、`bundle add` に相当する「依存を CLI から追加するコマンド」が長らく存在せず、`mix.exs` への手書きが基本です（[Igniter](https://hexdocs.pm/igniter/) で補完可能）。一枚岩ゆえに、コアが提供しない機能はコミュニティのタスクとして追加する文化です。

### 「設定より規約」の系譜

3つのフレームワークは思想的に一本の系譜でつながっています。

- **Rails（2004年〜）** が「設定より規約（Convention over Configuration）」「おまかせ（omakase）」を打ち出し、フルスタックフレームワークの原型を作りました
- **Laravel（2011年〜）** の Taylor Otwell 氏は Rails などの影響を公言しており、Eloquent・マイグレーション・artisan といった構成は Rails の設計を PHP 流に再解釈したものです
- **Phoenix** に至っては、Elixir の作者 José Valim 氏が **元 Rails コアチームのメンバー**であり、Phoenix 作者の Chris McCord 氏も Ruby 出身です。`mix phx.new` が生成するディレクトリ構造、マイグレーション、ジェネレータ文化は Rails の直系です

つまり3つとも「Rails 的な開発体験」という共通言語を持っています。だからこそ対応表が成立するのですが、Phoenix は Rails の規約文化を継承しつつ、**「マジック（暗黙の挙動）」を意図的に減らす**方向に舵を切りました。ルーティングからコントローラまでのリクエスト処理は `plug` の明示的なパイプラインで書き下されますし、ActiveSupport のようなコア拡張（モンキーパッチ）は言語仕様上そもそも不可能です。「Rails の生産性は好きだが、どこで何が起きているか追えなくなるのは嫌だ」という10年分の経験が反映された設計と言えます。

---

## 2. プロセスモデルの違い: BEAM の並行性が開発体験に効く

「Oban は Redis 不要」「テストがデフォルト並列」といった Phoenix の特徴は、すべてランタイムのプロセスモデルに由来します。3つの言語は「1リクエストをどう処理するか」のレベルで根本的に異なります。

| | PHP | Ruby | Elixir（BEAM） |
|---|---|---|---|
| 伝統的モデル | リクエスト毎にプロセスが状態を破棄（PHP-FPM） | プロセス（worker）+ スレッドのプール（Puma） | VM 内の軽量プロセスを大量生成 |
| 並行の単位 | OS プロセス | OS スレッド | **BEAM プロセス（数KB〜）** |
| 並列実行の制約 | プロセス毎に独立 | **GVL**: 1プロセス内で Ruby コードを同時実行できるのは1スレッド | 制約なし（CPU コア毎のスケジューラで並列実行） |
| スケジューリング | OS 任せ | OS 任せ（GVL の奪い合い） | **VM がプリエンプティブに制御** |
| 状態の共有 | 共有しない（shared nothing） | スレッド間でメモリ共有（要排他制御） | **共有しない（メッセージパッシング）** |

### PHP: 「毎回忘れる」モデルと、その変化

PHP の伝統的な実行モデルは「1リクエスト = 1実行コンテキスト」で、リクエストが終わるとメモリ上の状態はすべて破棄されます。この **shared nothing** 設計はメモリリークや状態汚染に強く、PHP が長年「雑に書いてもなんとかなる」と言われた安定性の源泉です。代償として、フレームワークの起動（ブートストラップ）を毎リクエスト繰り返すオーバーヘッドがあります。

近年はここが変わりつつあります。**FrankenPHP** の worker モードや **Laravel Octane**（FrankenPHP / Swoole / RoadRunner 対応）は、アプリケーションを一度起動してメモリに常駐させ、リクエストを使い回すモデルです。ブートストラップが消える分、大幅なスループット向上が見込めますが、引き換えに「static プロパティやシングルトンの状態が次のリクエストに漏れる」という、PHP 開発者がこれまで考えずに済んだ問題への注意が必要になります。つまり PHP は今、**Ruby や Elixir が昔から向き合ってきた「常駐プロセスの状態管理」の世界に合流しつつある**段階です。

### Ruby: プロセス + スレッド、そして GVL

Rails の標準サーバー Puma は「複数プロセス（worker）× 各プロセス内の複数スレッド」構成です。ただし CRuby には **GVL（Global VM Lock）** があり、1プロセス内で Ruby コードを同時に実行できるスレッドは1つだけです。スレッドが効くのは DB やAPIの I/O 待ちの間だけで、CPU を使う並列処理はプロセスを増やして実現します（プロセス数 × メモリ消費のトレードオフ）。

GVL を超える並列実行の仕組みとして **Ractor** が Ruby 3.0 から入っており、Ruby 4.0 では通信 API の `Ractor::Port` 追加や内部ロック競合の削減など改善が続いています。ただし Ractor 上では多くの gem がそのまま動かないため、**Rails アプリの実運用で Ractor が主役になる段階にはまだ達していない**、というのが現状の認識です。

### Elixir: 軽量プロセスとプリエンプティブスケジューリング

BEAM（Erlang VM）のプロセスは OS のプロセスやスレッドとは別物で、1つ数KBから始まる軽量な実行単位です。1つの VM 内に数十万〜数百万個を平気で生成でき、Phoenix では **1リクエスト = 1プロセス、1 WebSocket 接続 = 1プロセス**が基本です。プロセス同士はメモリを共有せず、メッセージパッシングで通信します。GC もプロセス単位なので、どこかの巨大な GC が全体を止める「stop the world」がありません。

もう1つ重要なのが**プリエンプティブスケジューリング**です。BEAM は CPU コアごとにスケジューラを持ち、各プロセスを一定の命令数（リダクション）ごとに強制的に切り替えます。協調的スケジューリング（Node.js のイベントループなど）と違い、**1つの重い処理が他のリクエストを巻き添えにしない**ことを VM が保証します。

この性質が開発体験に直結します。

- **`iex -S mix phx.server`**: HTTP サーバーは VM 内の1プロセス群にすぎないので、同じ VM に REPL プロセスで同居できます
- **`async: true` のテスト並列実行**: プロセスが状態を共有しないので、DB を触るテストも SQL Sandbox と組み合わせて安全に並列化できます
- **Redis 不要の Oban**: バックグラウンドワーカーは BEAM プロセスとして常駐し、DB ポーリングも並行性が安いため専用ミドルウェアが要りません
- **Channels / LiveView / Presence が標準**: 接続毎のプロセスが状態を持てるため、WebSocket のための外部インフラが不要です

**「インフラの部品数が減る」のは機能の多さではなく、ランタイムの性質の帰結**です。逆に言うと、この恩恵は BEAM の上でしか得られないため、CPU バウンドな数値計算などが主戦場なら別の選択肢（NIF や外部サービス）を検討することになります。

---

## 3. モデル中心からデータ変換中心へ: 同じ要件を3通りに書く

書き味が一番変わるのはデータ層です。「ユーザー登録では email・name・パスワード（12文字以上）が必須。プロフィール更新では email・name のみ変更可能でパスワードは触らない」という定番の要件を、3つのフレームワークで書き比べます。

### Laravel: バリデーションは HTTP 層（FormRequest）へ

Laravel の Eloquent モデル自体はバリデーションを持ちません。検証は **FormRequest** としてリクエスト毎に定義するのが標準です。

```php
// app/Http/Requests/RegisterUserRequest.php
class RegisterUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email'    => ['required', 'email', 'unique:users'],
            'name'     => ['required', 'string', 'max:100'],
            'password' => ['required', Password::min(12)],
        ];
    }
}

// app/Http/Requests/UpdateProfileRequest.php（更新時はパスワードを扱わない）
class UpdateProfileRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => ['required', 'email',
                        Rule::unique('users')->ignore($this->user()->id)],
            'name'  => ['required', 'string', 'max:100'],
        ];
    }
}
```

```php
// コントローラは検証済みデータを受け取るだけ
public function store(RegisterUserRequest $request): RedirectResponse
{
    $user = User::create([
        ...$request->safe()->except('password'),
        'password' => Hash::make($request->validated('password')),
    ]);
    // ...
}
```

コンテキスト（登録/更新）の違いは**リクエストクラスの違い**として表現されます。HTTP 経由なら綺麗に解けますが、コンソールやジョブからモデルを直接操作する経路では FormRequest を通らないため、検証はサービス層などで別途担保する必要があります。

### Rails: モデルにバリデーション + コンテキスト指定

ActiveRecord は「モデルが検証を持つ」設計です。コンテキスト毎の差は `on:` オプションで表現できます。

```ruby
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true
  validates :name,  presence: true, length: { maximum: 100 }
  validates :password, length: { minimum: 12 }, on: :create
end
```

```ruby
user = User.new(email: "a@example.com", name: "Alice", password: "short")
user.save          # => false（:create コンテキストでパスワード長を検証）

user = User.find(1)
user.update(name: "Bob")  # => true（:update ではパスワード検証はスキップ）

user.save(context: :account_setup)  # 任意の名前付きコンテキストも定義可能
```

`on:` で足りなくなる（コンテキストが3つ4つに増えて条件分岐が絡み合う）と、**フォームオブジェクト**に切り出すのが Rails の定番リファクタリングです。

```ruby
class Registration
  include ActiveModel::Model

  attr_accessor :email, :name, :password
  validates :email, :name, presence: true
  validates :password, length: { minimum: 12 }

  def save
    return false unless valid?
    User.create!(email:, name:, password:)
  end
end
```

つまり Rails では「モデル中心で始めて、複雑になったらコンテキスト毎のオブジェクトに分離する」という**段階的な進化**を辿ります。

### Ecto: 最初からコンテキスト毎の changeset

Ecto はこの問題を最初から設計に織り込んでいます。スキーマ（データ構造の定義）とバリデーション（変換ルール）が分離しており、**「どの操作か」毎に changeset 関数を定義する**のが標準スタイルです。

```elixir
defmodule Blog.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :password, :string, virtual: true, redact: true
    field :hashed_password, :string, redact: true
    timestamps()
  end

  # 登録用: パスワード必須・12文字以上
  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :password])
    |> validate_required([:email, :name, :password])
    |> validate_length(:name, max: 100)
    |> validate_length(:password, min: 12)
    |> unique_constraint(:email)
    |> hash_password()
  end

  # プロフィール更新用: パスワードはそもそも cast しない
  def profile_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name])
    |> validate_required([:email, :name])
    |> validate_length(:name, max: 100)
    |> unique_constraint(:email)
  end

  defp hash_password(%{valid?: true, changes: %{password: pw}} = changeset) do
    change(changeset, hashed_password: Bcrypt.hash_pwd_salt(pw))
  end

  defp hash_password(changeset), do: changeset
end
```

```elixir
# コンテキストモジュール（lib/blog/accounts.ex）
def register_user(attrs) do
  %User{}
  |> User.registration_changeset(attrs)
  |> Repo.insert()
end

def update_profile(%User{} = user, attrs) do
  user
  |> User.profile_changeset(attrs)
  |> Repo.update()
end
```

3つ並べると設計思想の違いが見えます。

- Laravel は検証を **HTTP 層**に置く（モデルは素通し）
- Rails は検証を**モデル**に置き、コンテキスト差はオプションや別オブジェクトで吸収する
- Ecto は検証を**操作（changeset 関数）**に置く。「User の正しさ」という単一の真実は存在せず、「登録として正しいか」「プロフィール更新として正しいか」だけがある

`registration_changeset` を最初に見たときの「冗長では？」という感覚は、コンテキストが増えるにつれて「衝突しない」という安心感に変わります。Rails でフォームオブジェクトに辿り着いた経験がある人ほど、**Ecto は「最終形が最初から標準」**だと感じるはずです。もう1つの副産物として、changeset は `Repo` に渡すまで DB に一切触らない純粋なデータ構造なので、バリデーションのテストが DB なしで書けます。

なお `cast` の対象カラムを操作毎に明示する設計は、Rails 初期に頻発した Mass Assignment 脆弱性（Strong Parameters 導入の経緯）への回答にもなっています。「そもそも cast していないフィールドは、どんなパラメータを投げられても変わらない」が構造的に保証されます。

---

## 4. 補論: 例外で守るか、クラッシュさせるか

もう1つ、コードレビューで顕在化しやすい文化差がエラーハンドリングです。Laravel / Rails では例外（Exception）を投げて上位の `Handler` や `rescue_from` で受け止めるのが基本です。一方 Elixir では、**予期できる失敗は `{:ok, result}` / `{:error, reason}` のタプルで戻り値として返し**、予期できない失敗は捕まえずにプロセスごとクラッシュさせます（**let it crash**）。

```elixir
case Accounts.register_user(params) do
  {:ok, user} -> redirect(conn, to: ~p"/users/#{user}")
  {:error, %Ecto.Changeset{} = changeset} -> render(conn, :new, changeset: changeset)
end
```

let it crash が成立するのは、前述のプロセスモデルが前提にあるからです。1プロセスの死は他のリクエストに波及せず、スーパーバイザーが定義済みの初期状態で再起動してくれます。「異常系を握りつぶして中途半端な状態で走り続けるより、綺麗に死んで綺麗にやり直す方が安全」という思想で、防御的な `try/rescue` をほとんど書かないコードになります。Laravel / Rails 経験者が Elixir のコードベースを読むと「例外処理が全然ない」ことに驚きますが、書いていないのではなく**ランタイムとスーパーバイザーに委譲している**のです。

---

## 5. 2026年の言語動向

思想の違いは今も現在進行形で動いています。直近のリリースを軽く押さえておきます。

- **Elixir 1.20**（2026年6月）: 漸進的型付けの第1マイルストーンが完了し、**型注釈を一切書かずに全プログラムの型推論・型チェック**が行われるようになりました。報告されるのは「実行すれば必ず落ちる」ことが保証された違反のみで、誤検知を極小に抑える設計です。「動的言語の書き味のまま静的検査の恩恵を足す」というアプローチは Sorbet（Ruby）や PHPStan（PHP）の「注釈・別ファイルで型を足す」方式とは対照的です
- **Ruby 4.0**（2025年12月）: 新 JIT コンパイラ **ZJIT**（実験的・本番非推奨）、モンキーパッチや定義を隔離する実験的機能 **Ruby Box**、Ractor の通信 API `Ractor::Port` 追加など、内部再構築が中心のリリースです
- **PHP 8.5**（2025年11月）: **パイプ演算子 `|>`** が入りました。`$result = $input |> trim(...) |> strtoupper(...);` のように書けます。Elixir の `|>` に慣れた目には感慨深い輸入で、関数合成スタイルの影響が主流言語に波及している例と言えます

Elixir が型を、PHP がパイプラインを取り込み、Ruby が並列実行の基盤を作り直している。**3つのエコシステムは互いの良いところを吸収しながら収斂しつつある**、というのが2026年の風景です。

---

## 6. どういう人がどれを選ぶと幸せか（私見）

最後に、あくまで執筆者の私見として。3つとも成熟したフルスタックフレームワークであり、大半のWebアプリケーションはどれでも問題なく作れます。そのうえで:

- **Laravel が向いている場面**: PHP 人材の採用しやすさ・ホスティングの選択肢の広さを重視する場合。公式周辺ツール（Forge、Horizon、Octane など）のカバー範囲が広く、「公式のレールに乗り続ける」体験は3つの中で最も充実しています
- **Rails が向いている場面**: 少人数で立ち上げから運用まで走り切るプロダクト開発。「omakase」に乗ったときの初速と、20年分の蓄積（gem・情報・実践知）は依然として強力です。GVL やメモリのスケール特性は、多くのビジネスでは問題になる前に事業側の別のボトルネックが来ます
- **Phoenix が向いている場面**: WebSocket・リアルタイム性・多数の同時接続が要件の中心にある場合や、インフラ部品を増やさず1つのランタイムに寄せたい場合。関数型と changeset の学習コストは最初の数週間に集中しますが、その先の「暗黙の挙動が少なく、壊れ方が予測できる」感覚はチームの規模が大きくなるほど効いてきます

迷ったら、**チームが今持っている言語資産と、要件に占めるリアルタイム性の比重**で決めるのが現実的だと考えています。そしてどれを選んでも、他の2つの思想を知っていることは設計の引き出しとして必ず活きます。FormRequest に changeset の発想を、Rails のフォームオブジェクトに Ecto の割り切りを持ち込む、といった越境こそが対応表シリーズの本当の使い道かもしれません。

---

## 参考リンク

- [Mix (Hexdocs)](https://hexdocs.pm/mix/Mix.html) / [Composer](https://getcomposer.org/) / [Bundler](https://bundler.io/)
- [Ecto.Changeset](https://hexdocs.pm/ecto/Ecto.Changeset.html) / [Active Record バリデーション (Rails ガイド)](https://guides.rubyonrails.org/active_record_validations.html) / [Laravel Validation (FormRequest)](https://laravel.com/docs/13.x/validation#form-request-validation)
- [Elixir v1.20 released: now a gradually typed language](https://elixir-lang.org/blog/2026/06/03/elixir-v1-20-0-released/)
- [Ruby 4.0.0 Released](https://www.ruby-lang.org/en/news/2025/12/25/ruby-4-0-0-released/)
- [PHP 8.5 Release Announcement](https://www.php.net/releases/8.5/en.php)
- [FrankenPHP: Worker Mode](https://frankenphp.dev/docs/worker/) / [Laravel Octane](https://laravel.com/docs/13.x/octane)
- [Erlang: Processes (BEAM の軽量プロセス)](https://www.erlang.org/doc/system/ref_man_processes.html) / [Phoenix ドキュメント](https://hexdocs.pm/phoenix/overview.html)
