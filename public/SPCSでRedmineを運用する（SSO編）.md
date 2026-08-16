---
title: SPCSでRedmineを運用する（SSO編）
tags:
  - Snowflake
  - SPCS
  - Redmine
  - SSO
  - OAuth
private: false
updated_at: '2026-08-17T00:48:20+09:00'
id: 6eda59d65cd4717e5843
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

[構築編](https://qiita.com/)では SPCS 上に Redmine を稼働させ、[メール通知編](https://qiita.com/)ではメール配信の仕組みを構築しました。本記事（SSO編）では、Snowflake OAuth と Redmine のシングルサインオン（SSO）を実現するプラグインを導入します。

SPCS のパブリックエンドポイントには Snowflake OAuth 認証が自動的に適用されます。つまり、ユーザーは Redmine にアクセスする時点で既に Snowflake に認証済みです。しかし、Redmine 自体にも独自のログイン画面があり、デフォルトでは Snowflake 認証後に再度 Redmine のログインが求められます。

SPCS 上で Redmine を運用してしばらく経ったのですが、やはりログインが二度手間というのは利用者にとっては煩わしいので、SSOを導入することにしました。当初、Snowflake の認証をバイパスして Redmine の認証に一本化することも考えたのですが、SPCS の仕様上、外部からのアクセスにはSnowflake の認証が必須とわかりました。そこで　Snowflake の認証後に SSO で Redmine にログインする方針に変更しました。（Redmine にたまたま SSO を受け入れる機能があって良かったです。）

本記事で導入するプラグインは、SPCS が付与するヘッダー情報を使って Redmine のログインを自動化し、ユーザーから見て Snowflake にログインするだけで Redmine が使える状態にします。

なお、本記事の作成には以下のAIツールを活用しています:

1. **Kiro** — 記事の作成支援
2. **CoCo (Cortex Code)** — プラグインのコード作成支援

## アーキテクチャ

```
[ユーザー] → (Snowflake OAuth 認証)
                    │
                    │ ヘッダー付与: Sf-Context-Current-User
                    ▼
            [SPCS Public Endpoint :3000]
                    │
            [Redmine Container]
             └── plugins/redmine_spcs_sso/init.rb
                    │
                    ├─ ヘッダーからログイン名を取得
                    ├─ Redmine ユーザー検索
                    │    └─ 存在しない場合 → 自動作成
                    │         │
                    │         ▼
                    │    Snowflake SQL API (DESCRIBE USER)
                    │    → メールアドレス取得
                    │
                    └─ セッション設定 → Redmine 認証バイパス
```

### 認証フロー

1. ユーザーが SPCS パブリックエンドポイントにアクセス
2. Snowflake OAuth でユーザーが認証される
3. SPCS がリクエストヘッダー `Sf-Context-Current-User` にログイン名を付与
4. プラグインがヘッダーを読み取り、Redmine ユーザーを検索
5. ユーザーが存在しない場合は Snowflake API でメールアドレスを取得し自動作成
6. Redmine セッションを設定し、標準のログイン画面をバイパス

### 設計ポイント

| 項目 | 説明 |
|------|------|
| 認証方式 | SPCS ヘッダー `Sf-Context-Current-User` を信頼 |
| ユーザー自動作成 | Snowflake SQL API (`DESCRIBE USER`) でメールアドレスを取得 |
| セッション管理 | SSO ユーザーのセッション期限切れをスキップ（Snowflake OAuth に委任） |
| sudo モード | 無効化が必要（SSO 環境ではパスワード入力不可） |

## 前提条件

- [構築編](https://qiita.com/)のセットアップが完了していること
- `REDMINE_SERVICE_STAGING` が稼働中であること
- admin ユーザーで Redmine にログインできること

## 構築手順

### 1. Snowflake 側の権限付与

プラグインが `DESCRIBE USER` を実行するために、サービスロールに `MONITOR` 権限が必要です。

```sql
USE ROLE ACCOUNTADMIN;
GRANT MONITOR ON ACCOUNT TO ROLE REDMINE_SERVICE_ROLE_STAGING;
```

:::note warn
`MONITOR ON ACCOUNT` はアカウントレベルの権限です。これにより、サービスロールがアカウント内のユーザー情報（メールアドレス等）を参照できるようになります。
:::

### 2. サービス仕様への環境変数追加

プラグインが Snowflake API に接続するために、`SNOWFLAKE_HOST` 環境変数をサービス仕様に追加します。

`redmine_service_spec.yaml` の `env` セクションに以下を追加:

```yaml:redmine_service_spec.yaml
    env:
      REDMINE_DB_POSTGRES: <YOUR_PG_HOST>
      REDMINE_DB_PORT: "5432"
      REDMINE_DB_DATABASE: "postgres"
      REDMINE_DB_USERNAME: "application"
      REDMINE_PLUGINS_MIGRATE: "true"
      SNOWFLAKE_HOST: "<YOUR_ACCOUNT>.snowflakecomputing.com"  # 追加
```

`<YOUR_ACCOUNT>` は自身の Snowflake アカウント識別子に置き換えてください。

### 3. configuration.yml の変更

SSO 環境ではパスワード入力ができないため、sudo モードを無効化します。

`configuration.yml` に以下を追加:

```yaml:configuration.yml
production:
  sudo_mode: false
```

:::note info
Redmine の sudo モードは、管理操作時に再度パスワード入力を求める機能です。以前から存在している機能ですが、Redmine 7.0.0 以降、デフォルトで有効化されています。SSO 環境ではパスワードが存在しないため、無効化しないと管理操作ができなくなります。
:::

変更後、`@REDMINE_DATASTORE/CONFIG/` にアップロードします。

### 4. Redmine 管理画面での事前設定

プラグインをデプロイする前に、Redmine 管理画面で以下の設定を行います。Snowflake OAuth 認証後に表示される Redmine のログイン画面に `admin` / `admin` でログインしてください。

#### 4-1. admin ユーザーのログイン名変更

**管理 > ユーザー > admin** を開き、ログイン名を変更します。

| 項目 | 変更前 | 変更後 |
|------|--------|--------|
| ログイン | `admin` | あなたの Snowflake ユーザー名（大文字） |

これにより、プラグインデプロイ後も管理者権限でログインできます。

:::note alert
この手順を忘れると、プラグインデプロイ後に admin 権限でアクセスできなくなります。必ずプラグインデプロイ前に実施してください。
:::

#### 4-2. 既存ユーザーのログイン名変更

他のユーザーも同様に、**管理 > ユーザー > 対象ユーザー** でログイン名を Snowflake のユーザー名（大文字）に変更します。

#### 4-3. 認証設定（任意）

**管理 > 設定 > 認証タブ** で以下を設定します:

| 項目 | 推奨設定 | 理由 |
|------|----------|------|
| 認証が必要 | ON | 匿名アクセスを禁止し、必ず SSO 経由でログインさせる |
| ユーザーによるアカウント登録 | 無効 | プラグインが自動作成するため不要 |
| ユーザーによるアカウント削除を許可 | OFF | 意図しないアカウント削除を防止 |

### 5. プラグインの配置

プラグインのファイルを `@REDMINE_DATASTORE/PLUGINS/redmine_spcs_sso/` に配置します。

プラグインは単一ファイル（`init.rb`）で構成されています:

```
redmine_spcs_sso/
└── init.rb
```

#### init.rb

```ruby:init.rb
# Snowflake SPCS OAuth header-based SSO plugin for Redmine

require 'net/http'
require 'json'
require 'uri'

module RedmineSpcsSSO
  @email_cache = {}

  class << self
    attr_accessor :email_cache
  end

  def self.find_or_create_user(login)
    user = User.find_by_login(login)
    return user if user

    email = fetch_snowflake_email(login)

    user = User.new
    user.login = login
    user.firstname = login.split('_').last.capitalize
    user.lastname = login.split('_')[-2]&.capitalize || 'User'
    user.mail = email || "#{login.downcase}@spcs.local"
    user.admin = false
    user.status = User::STATUS_ACTIVE
    user.language = Setting.default_language

    if user.save(validate: false)
      Rails.logger.info "[SPCS SSO] Created user: #{login} (email: #{user.mail})"
      user
    else
      Rails.logger.warn "[SPCS SSO] Failed to create user: #{login} - #{user.errors.full_messages.join(', ')}"
      nil
    end
  end

  def self.fetch_snowflake_email(login)
    return @email_cache[login] if @email_cache.key?(login)

    email = query_snowflake_user_email(login)
    @email_cache[login] = email
    email
  rescue => e
    Rails.logger.warn "[SPCS SSO] Failed to fetch email for #{login}: #{e.message}"
    nil
  end

  def self.query_snowflake_user_email(login)
    token_path = '/snowflake/session/token'
    return nil unless File.exist?(token_path)

    token = File.read(token_path).strip

    account_host = ENV['SNOWFLAKE_HOST']
    return nil if account_host.nil? || account_host.empty?

    uri = URI("https://#{account_host}/api/v2/statements")
    http = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl = true
    http.open_timeout = 5
    http.read_timeout = 10

    req = Net::HTTP::Post.new(uri)
    req['Authorization'] = "Snowflake Token=\"#{token}\""
    req['Content-Type'] = 'application/json'
    req['Accept'] = 'application/json'
    req['X-Snowflake-Authorization-Token-Type'] = 'KEYPAIR_JWT'

    req.body = {
      statement: "DESCRIBE USER #{login.gsub(/[^A-Za-z0-9_]/, '')}",
      timeout: 10,
      resultSetMetaData: { format: "jsonv2" }
    }.to_json

    response = http.request(req)
    return nil unless response.is_a?(Net::HTTPSuccess)

    result = JSON.parse(response.body)
    data = result['data']
    return nil unless data.is_a?(Array)

    data.each do |row|
      if row.is_a?(Array) && row[0].to_s.upcase == 'EMAIL'
        email = row[1].to_s.strip
        return email.empty? ? nil : email
      end
    end
    nil
  end
end

Redmine::Plugin.register :redmine_spcs_sso do
  name 'Snowflake SPCS SSO'
  description 'Automatically authenticates users via Snowflake SPCS OAuth header (Sf-Context-Current-User)'
  version '1.0.0'
  author 'CoCo'
end

# Wrap session_expiration to inject SSO session before expiry check
module RedmineSpcsSSO
  module SessionExpirationPatch
    private

    def session_expiration
      remote_user = request.env['HTTP_SF_CONTEXT_CURRENT_USER']
      if remote_user.present?
        login = remote_user.strip
        unless session[:user_id] && User.where(id: session[:user_id], login: login).exists?
          user = RedmineSpcsSSO.find_or_create_user(login)
          if user&.active?
            session[:user_id] = user.id
            session[:ctime] = Time.now.utc.to_i
            session[:atime] = Time.now.utc.to_i
            user.update_column(:last_login_on, Time.now) unless user.last_login_on&.today?
          end
        else
          session[:atime] = Time.now.utc.to_i
        end
        # Skip original session_expiration for SSO users
        # Authentication is handled by Snowflake OAuth
        return
      end
      super
    end
  end
end

Rails.application.config.after_initialize do
  ApplicationController.prepend(RedmineSpcsSSO::SessionExpirationPatch)
end
```

構築編で紹介したプラグイン追加手順と同様に、ZIPファイルとしてアップロードするか、Snowsight から直接ファイルをアップロードしてください。

### 6. サービスの再起動

設定ファイルとプラグインを反映するためにサービスを再起動します。

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;

ALTER SERVICE REDMINE_SERVICE_STAGING
  FROM @REDMINE_DB.STAGING.REDMINE_DATASTORE
  SPECIFICATION_FILE = 'SPECS/redmine_service_spec.yaml';
```

:::note info
`SNOWFLAKE_HOST` 環境変数の追加を含むサービス仕様の変更があるため、`ALTER SERVICE ... FROM ... SPECIFICATION_FILE` で仕様ごと更新します。単にプラグインファイルの追加だけであれば SUSPEND → RESUME でも反映されます。
:::

## 動作確認

### 1. サービスの起動確認

```sql
CALL SYSTEM$GET_SERVICE_STATUS('REDMINE_SERVICE_STAGING');
```

`status` が `READY` になったことを確認します。

### 2. ログでプラグインの読み込みを確認

```sql
CALL SYSTEM$GET_SERVICE_LOGS('REDMINE_SERVICE_STAGING', '0', 'redmine', 50);
```

プラグインが正常に読み込まれていれば、エラーなく起動ログが表示されます。

### 3. ブラウザでアクセス

SPCS パブリックエンドポイントにアクセスすると:

1. Snowflake のログイン画面が表示される
2. Snowflake に認証すると、直接 Redmine のトップページが表示される（Redmine のログイン画面は表示されない）
3. 右上にログイン中のユーザー名が表示される

### 4. ユーザー自動作成の確認

Snowflake に存在するが Redmine に未登録のユーザーでアクセスした場合:

1. 自動的にアカウントが作成される
2. Snowflake に登録されたメールアドレスが Redmine ユーザーに設定される
3. ログで `[SPCS SSO] Created user: <LOGIN>` が確認できる

## プラグインの仕組み（詳細）

### ヘッダーの信頼性

`Sf-Context-Current-User` ヘッダーは SPCS インフラが付与するもので、クライアントからは偽装できません。SPCS パブリックエンドポイントでは Snowflake OAuth が必須（無効化不可）であるため、このヘッダーが存在する時点でユーザーは確実に Snowflake 認証済みです。

### SessionExpirationPatch

Redmine の `ApplicationController#session_expiration` メソッドを `prepend` でオーバーライドしています:

- SSO ヘッダーが存在する場合: セッションを設定し、Redmine 標準のセッション期限切れチェックをスキップ
- SSO ヘッダーが存在しない場合: 通常の `session_expiration` を実行（`super`）

これにより、Snowflake OAuth のセッションが有効な限り Redmine にもアクセスし続けられます。

### メールアドレスの取得

Snowflake SQL API (`/api/v2/statements`) を使って `DESCRIBE USER <LOGIN>` を実行し、`EMAIL` プロパティを取得します。取得したメールアドレスはインメモリキャッシュに保存され、同一ユーザーへの重複 API コールを防ぎます。

メールアドレスが取得できない場合は `<login>@spcs.local` がフォールバックとして設定されます。

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| SSO 後も Redmine のログイン画面が表示される | プラグインが読み込まれていない | ログ確認。`@REDMINE_DATASTORE/PLUGINS/redmine_spcs_sso/init.rb` の配置を確認 |
| `[SPCS SSO] Failed to fetch email` | `SNOWFLAKE_HOST` が未設定 or `MONITOR` 権限不足 | 環境変数と `GRANT MONITOR ON ACCOUNT` を確認 |
| 管理画面にアクセスできない（sudo モード） | `sudo_mode: false` が未設定 | `configuration.yml` に追加してサービス再起動 |
| admin 権限でログインできない | ログイン名の変更忘れ | PostgreSQL に直接接続してログイン名を更新するか、プラグインを一時削除して admin でログイン |
| ユーザーが自動作成されない | `handle_DATA` でエラー | ログに `Failed to create user` がないか確認。Redmine のバリデーション問題の可能性 |

### admin 権限のリカバリ手順

プラグインデプロイ後に admin でアクセスできなくなった場合:

1. プラグインを一時的に削除
   ```sql
   USE ROLE REDMINE_SERVICE_ROLE_STAGING;
   REMOVE @REDMINE_DATASTORE/PLUGINS/redmine_spcs_sso/init.rb;
   ```

2. サービスを再起動
   ```sql
   ALTER SERVICE REDMINE_SERVICE_STAGING SUSPEND;
   ALTER SERVICE REDMINE_SERVICE_STAGING RESUME;
   ```

3. Redmine に `admin` / `admin` でログインし、ログイン名を修正

4. プラグインを再デプロイしてサービスを再起動

## まとめ

本記事では、Snowflake SPCS の OAuth 認証と Redmine を統合する SSO プラグインを導入しました。

- **シームレスな認証** — Snowflake にログインするだけで Redmine にもアクセス可能
- **ユーザー自動作成** — Snowflake ユーザーが初回アクセス時に自動的に Redmine アカウントを取得
- **メールアドレス連携** — Snowflake に登録されたメールアドレスが Redmine に自動設定される
- **セッション管理の委任** — Snowflake OAuth にセッション管理を委任し、二重認証の手間を排除
- **単一ファイル構成** — `init.rb` のみでマイグレーション不要、デプロイと削除が容易
