---
title: SPCSでRedmineを運用する（構築編）
tags:
  - Snowflake
  - SPCS
  - Redmine
  - Docker
  - PostgreSQL
private: false
updated_at: '2026-08-17T00:48:20+09:00'
id: 0167331f12d46d6cd2c5
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

Snowflake の Snowpark Container Services (SPCS) 上で Redmine を稼働させてみました。データベースは Snowflake Managed PostgreSQL を利用しています。

SPCSを使えば、Snowflakeのインフラ上でコンテナアプリケーションをホストでき、Snowflake OAuth によるアクセス制御やManaged PostgreSQL との連携がシームレスに行えます。

本記事（構築編）では Redmine をSPCS上で稼働させるまでの手順を解説します。続編として [メール通知編](https://qiita.com/) と [SSO編](https://qiita.com/) も公開しています。

稼働させるまでに色々と詰まったのでトラブルシュートの備忘も兼ねて記載していきます。

なお、本記事の作成には以下のAIツールを活用しています:

1. **Kiro** — 記事の作成支援
2. **CoCo (Cortex Code)** — SQLや設定ファイルの作成支援

## アーキテクチャ

```
[ユーザー] → (Snowflake OAuth) → [SPCS Public Endpoint :3000]
                                        │
                                  [Redmine Container]
                                   ├── /usr/src/redmine/files   ← @REDMINE_DATASTORE/FILES
                                   ├── /usr/src/redmine/plugins ← @REDMINE_DATASTORE/PLUGINS
                                   ├── /usr/src/redmine/themes  ← @REDMINE_DATASTORE/THEMES
                                   └── /opt/redmine-config      ← @REDMINE_DATASTORE/CONFIG
                                        │
                                  (EAI: NWRULE_PGEGRESS :5432, NWRULE_RGEGRESS :443)
                                        │
                                  [Snowflake Managed PostgreSQL]
                                   REDMINE_POSTGRESQL_STAGING (BURST_S, PG18)
                                        │
                                  (Network Policy: NWRULE_PGINGRESS 0.0.0.0/0)
```

### コンポーネント一覧

| リソース | 名前 | 説明 |
|---------|------|------|
| Database | `REDMINE_DB.STAGING` | 各種オブジェクトを格納するスキーマ |
| Compute Pool | `REDMINE_COMPUTEPOOL_STAGING` | CPU_X64_XS, 1ノード |
| PostgreSQL | `REDMINE_POSTGRESQL_STAGING` | Snowflake Managed PostgreSQL (BURST_S, PG18) |
| Service | `REDMINE_SERVICE_STAGING` | Redmine コンテナサービス |
| Image Repository | `DOCKER_REPO` | `redmine:latest` を格納 |
| Stage | `REDMINE_DATASTORE` | files/, plugins/, themes/, specs/, config/ |
| Network Policy | `REDMINE_NWPOLICY_STAGING` | Compute Pool → PostgreSQL への Ingress 許可 |
| Network Rule | `NWRULE_PGINGRESS` | PostgreSQL Ingress (0.0.0.0/0) |
| Network Rule | `NWRULE_PGEGRESS` | PostgreSQL への Egress (:5432) |
| Network Rule | `NWRULE_RGEGRESS` | redmine.org / rubygems.org への Egress (:443) |
| Secret | `SECRET_PG_APPLICATION` | PostgreSQL application ロールの認証情報 |
| EAI | `REDMINE_EAI_STAGING` | Egress ルールと認証情報を統合 |
| Role | `REDMINE_SERVICE_ROLE_STAGING` | サービス所有者ロール |

## 前提条件

- Snowflake アカウント（ACCOUNTADMIN / SYSADMIN ロールへのアクセス）
- 仮想ウェアハウス（COMPUTE_WH）
- Docker CLI（イメージの pull/push 用）
- SnowSQL または Snowsight へのアクセス

## 構築手順

### 1. Database・スキーマの作成

専用のデータベース、スキーマを作成します。

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;

CREATE OR ALTER DATABASE REDMINE_DB;
CREATE OR ALTER SCHEMA REDMINE_DB.STAGING;

USE DATABASE REDMINE_DB;
USE SCHEMA REDMINE_DB.STAGING;
```

### 2. ステージの作成

Redmineのファイル・プラグイン・テーマ・設定ファイルを格納するためのステージを作成します。

```sql
CREATE STAGE IF NOT EXISTS REDMINE_DATASTORE
  DIRECTORY = ( ENABLE = true )
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
```

作成後、Snowsight からGUIで以下のフォルダ構造を作成します。（空ファイルをアップロードすることでフォルダが生成されます。）筆者はTouchコマンドで空ファイルを作成してアップロードしました。:

- `files/` — Redmineの添付ファイル保存先
- `plugins/` — プラグイン格納先
- `themes/` — テーマ格納先
- `specs/` — サービス仕様ファイル格納先
- `config/` — 設定ファイル格納先

### 3. Snowflake Managed PostgreSQL インスタンスの作成

Redmineのデータベースとして、Snowflake Managed PostgreSQL を使用します。
日本語ドキュメントでは COMPUTE_FAMILY として安価な BURST_XS の記載がありますが、実は利用できません。（英語ドキュメントにはXSの記載はありません。）

ちなみに筆者の環境では当初、インスタンス作成に失敗しました。2026年2月のGAから半年程度と、まだ若いサービスなので、参考となる先行事例も少なく途方に暮れていましたが、サポートの方の迅速なご対応に助けられました。

```sql
USE ROLE ACCOUNTADMIN;
CREATE POSTGRES INSTANCE IF NOT EXISTS REDMINE_POSTGRESQL_STAGING
  COMPUTE_FAMILY = 'BURST_S'
  STORAGE_SIZE_GB = 10
  AUTHENTICATION_AUTHORITY = POSTGRES
  POSTGRES_VERSION = 18;
```

:::note warn
作成時に `application` ロールと `snowflake_admin` ロールのパスワードが表示されます。**パスワードは作成時のみ表示される**ため、必ず控えてください。
:::

### 4. Compute Pool の作成

最小ノード、最小インスタンスサイズでコンピュートプールを作成します。

```sql
USE ROLE SYSADMIN;
CREATE COMPUTE POOL IF NOT EXISTS REDMINE_COMPUTEPOOL_STAGING
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = CPU_X64_XS;
```

### 5. External Access Integration（外部アクセス設定）

SPCSコンテナからManaged PostgreSQLへ接続するためのネットワーク設定を行います。

#### 5-1. Network Policy（Ingress許可）

Compute PoolからManaged PostgreSQLへのIngress通信を許可します。

```sql
CREATE OR REPLACE NETWORK RULE NWRULE_PGINGRESS
  MODE = POSTGRES_INGRESS
  TYPE = IPV4
  VALUE_LIST = ('0.0.0.0/0');

USE ROLE ACCOUNTADMIN;
CREATE OR REPLACE NETWORK POLICY REDMINE_NWPOLICY_STAGING
  ALLOWED_NETWORK_RULE_LIST = (REDMINE_DB.STAGING.NWRULE_PGINGRESS);

USE ROLE SYSADMIN;
ALTER POSTGRES INSTANCE REDMINE_POSTGRESQL_STAGING
  SET NETWORK_POLICY = REDMINE_NWPOLICY_STAGING;
```

:::note info
Compute Pool の IP 範囲が特定できないため、Ingressは全許可 (`0.0.0.0/0`) としています。PostgreSQL自体の認証で保護されます。
:::

#### 5-2. Network Rule（PostgreSQLへのEgress許可）

まず、PostgreSQLインスタンスのホスト名を確認します。

```sql
DESCRIBE POSTGRES INSTANCE REDMINE_POSTGRESQL_STAGING;
```

取得した `host` 列の値を使用して、Egressルールを作成します。

```sql
CREATE OR REPLACE NETWORK RULE NWRULE_PGEGRESS
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('<YOUR_PG_HOST>:5432');
```

`<YOUR_PG_HOST>` は `DESCRIBE POSTGRES INSTANCE` で取得したホスト名に置き換えてください。

#### 5-3. Network Rule（redmine.org, rubygems.org へのEgress許可）

プラグインのGem依存関係を解決するため、rubygems.org への通信を許可します。

```sql
CREATE OR REPLACE NETWORK RULE NWRULE_RGEGRESS
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('redmine.org:443', 'index.rubygems.org:443', 'rubygems.org:443');
```

#### 5-4. Secret（PostgreSQL接続用の認証情報）

PostgreSQL作成時に取得したパスワードを使用してSecretを作成します。

```sql
CREATE OR REPLACE SECRET SECRET_PG_APPLICATION
  TYPE = PASSWORD
  USERNAME = 'application'
  PASSWORD = '<YOUR_PG_APPLICATION_PASSWORD>';
```

`<YOUR_PG_APPLICATION_PASSWORD>` はPostgreSQLインスタンス作成時に表示された `application` ロールのパスワードに置き換えてください。

#### 5-5. External Access Integration の作成

ネットワークルールを適用したネットワークポリシーを作成します。

```sql
USE ROLE ACCOUNTADMIN;
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION REDMINE_EAI_STAGING
  ALLOWED_NETWORK_RULES = (NWRULE_PGEGRESS, NWRULE_RGEGRESS)
  ALLOWED_AUTHENTICATION_SECRETS = (SECRET_PG_APPLICATION)
  ENABLED = true;
```

### 6. Image Repository の作成

Redmine の Docker イメージを格納するリポジトリを作成します。

```sql
USE ROLE SYSADMIN;
CREATE IMAGE REPOSITORY IF NOT EXISTS DOCKER_REPO;

-- リポジトリURLの確認（docker push する際に必要）
SHOW IMAGE REPOSITORIES;
```

### 7. Redmine コンテナイメージの準備

ローカルマシンで以下のコマンドを実行し、RedmineイメージをSnowflakeレジストリにpushします。

```bash
# Snowflakeレジストリへログイン
docker login <YOUR_SNOWFLAKE_ACCOUNT>.registry.snowflakecomputing.com

# amd64イメージのdigestを確認
docker manifest inspect --verbose redmine:latest
# 出力結果から architecture が amd64 のエントリの digest を確認

# Redmineイメージをpull（amd64版を指定）
docker pull redmine:latest@sha256:<DIGEST>

# タグ付け（SHOW IMAGE REPOSITORIES の repository_url を使用）
docker tag redmine:latest@sha256:<DIGEST> \
  <YOUR_SNOWFLAKE_ACCOUNT>.registry.snowflakecomputing.com/redmine_db/staging/docker_repo/redmine:latest

# push
docker push \
  <YOUR_SNOWFLAKE_ACCOUNT>.registry.snowflakecomputing.com/redmine_db/staging/docker_repo/redmine:latest
```

:::note warn
SPCSはamd64アーキテクチャのみサポートしています。Apple Silicon (M1/M2/M3) 環境の場合は `docker manifest inspect` で amd64 版の digest を特定し、明示的に指定してpullしてください。
:::

### 8. サービス用ロールの作成と権限付与

```sql
CREATE ROLE IF NOT EXISTS REDMINE_SERVICE_ROLE_STAGING;

GRANT OWNERSHIP ON DATABASE REDMINE_DB TO ROLE REDMINE_SERVICE_ROLE_STAGING COPY CURRENT GRANTS;
GRANT USAGE ON SCHEMA REDMINE_DB.STAGING TO ROLE REDMINE_SERVICE_ROLE_STAGING;
GRANT USAGE, MONITOR ON COMPUTE POOL REDMINE_COMPUTEPOOL_STAGING TO ROLE REDMINE_SERVICE_ROLE_STAGING;
GRANT USAGE ON INTEGRATION REDMINE_EAI_STAGING TO ROLE REDMINE_SERVICE_ROLE_STAGING;
GRANT READ ON SECRET SECRET_PG_APPLICATION TO ROLE REDMINE_SERVICE_ROLE_STAGING;
GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE REDMINE_SERVICE_ROLE_STAGING;
GRANT READ, WRITE ON STAGE REDMINE_DATASTORE TO ROLE REDMINE_SERVICE_ROLE_STAGING;
GRANT READ, WRITE ON IMAGE REPOSITORY DOCKER_REPO TO ROLE REDMINE_SERVICE_ROLE_STAGING;
GRANT CREATE SERVICE ON SCHEMA REDMINE_DB.STAGING TO ROLE REDMINE_SERVICE_ROLE_STAGING;
```

:::note alert
`GRANT USAGE ON INTEGRATION REDMINE_EAI_STAGING TO ROLE REDMINE_SERVICE_ROLE_STAGING` が漏れると、サービス起動直後は動作しますが、約2時間後にバックグラウンドの権限検証でEgress設定が無効化され、PostgreSQLへの接続が失敗します。エラーメッセージは `could not translate host name ... to address: Name or service not known` として現れます。
:::

### 9. サービス仕様ファイルの準備

以下の内容で `redmine_service_spec.yaml` を作成し、`@REDMINE_DATASTORE/SPECS/` にアップロードします。

```yaml:redmine_service_spec.yaml
spec:
  containers:
  - name: redmine
    image: /REDMINE_DB/STAGING/DOCKER_REPO/redmine:latest
    command:
    - /bin/sh
    - -c
    - |
      cp /opt/redmine-config/configuration.yml /usr/src/redmine/config/configuration.yml
      exec /docker-entrypoint.sh rails server -b 0.0.0.0
    env:
      REDMINE_DB_POSTGRES: <YOUR_PG_HOST>
      REDMINE_DB_PORT: "5432"
      REDMINE_DB_DATABASE: "postgres"
      REDMINE_DB_USERNAME: "application"
      REDMINE_PLUGINS_MIGRATE: "true"
      SNOWFLAKE_HOST: "<YOUR_ACCOUNT>.snowflakecomputing.com"
    secrets:
    - snowflakeSecret: REDMINE_DB.STAGING.SECRET_PG_APPLICATION
      secretKeyRef: password
      envVarName: REDMINE_DB_PASSWORD
    resources:
      requests:
        cpu: 1
        memory: 2Gi
      limits:
        cpu: 2
        memory: 4Gi
    volumeMounts:
    - name: files
      mountPath: /usr/src/redmine/files
    - name: plugins
      mountPath: /usr/src/redmine/plugins
    - name: themes
      mountPath: /usr/src/redmine/themes
    - name: config
      mountPath: /opt/redmine-config
  endpoints:
  - name: redmine-web
    port: 3000
    protocol: HTTP
    public: true
  platformMonitor:
    metricConfig:
      groups:
        - system
  volumes:
  - name: files
    source: "@REDMINE_DB.STAGING.REDMINE_DATASTORE/FILES"
    uid: 999
    gid: 999
  - name: plugins
    source: "@REDMINE_DB.STAGING.REDMINE_DATASTORE/PLUGINS"
    uid: 999
    gid: 999
  - name: themes
    source: "@REDMINE_DB.STAGING.REDMINE_DATASTORE/THEMES"
    uid: 999
    gid: 999
  - name: config
    source: "@REDMINE_DB.STAGING.REDMINE_DATASTORE/CONFIG"
    uid: 999
    gid: 999
```

`<YOUR_PG_HOST>` は PostgreSQL インスタンスのホスト名に置き換えてください。

**ファイルマウントのポイント:**

- `/usr/src/redmine/config` を直接マウントすると、Rails が必要とする `database.yml` や `routes.rb` 等が上書きされてしまいます。そのため、`/opt/redmine-config` にマウントし、コンテナ起動時に `cp` で配置する構成にしています。
- `REDMINE_PLUGINS_MIGRATE: "true"` により、起動時にプラグインのDBマイグレーションが自動実行されます。
- `uid: 999` / `gid: 999` はRedmineコンテナ内の実行ユーザーに合わせた設定です。

### 10. Redmine サービスの作成

最大、最小1インスタンスで作成します。

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;

CREATE SERVICE REDMINE_SERVICE_STAGING
  IN COMPUTE POOL REDMINE_COMPUTEPOOL_STAGING
  FROM @REDMINE_DB.STAGING.REDMINE_DATASTORE
  SPECIFICATION_FILE = 'SPECS/redmine_service_spec.yaml'
  EXTERNAL_ACCESS_INTEGRATIONS = (REDMINE_EAI_STAGING)
  MIN_INSTANCES = 1
  MAX_INSTANCES = 1;
```

### 11. 動作確認

サービスの起動状態を確認します。

```sql
CALL SYSTEM$GET_SERVICE_STATUS('REDMINE_SERVICE_STAGING');
```

`status` が `READY` になったら、エンドポイントURLを確認します。

```sql
SHOW ENDPOINTS IN SERVICE REDMINE_SERVICE_STAGING;
```

ユーザーへのロール付与は以下で行います:

```sql
USE ROLE USERADMIN;
GRANT ROLE REDMINE_SERVICE_ROLE_STAGING TO USER <YOUR_USER>;
ALTER USER <YOUR_USER> SET DEFAULT_ROLE = 'REDMINE_SERVICE_ROLE_STAGING';
```

## プラグインの追加

Redmineにプラグインを追加する手順です。

### 1. ZIP ファイルのアップロード

GitHubからプラグインのZIPをダウンロードし、Snowsightで `@REDMINE_DATASTORE/PLUGINS/` にアップロードします。

### 2. ZIP の展開

以下のストアドプロシージャを作成・実行することで、ステージ上のZIPファイルを展開できます。GitHubのZIPに付与される `-main`、`-master` 等のサフィックスは自動除去されます。

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;
USE DATABASE REDMINE_DB;
USE SCHEMA REDMINE_DB.STAGING;

CREATE OR REPLACE PROCEDURE EXTRACT_ZIP_FILES_ON_STAGE(
    stage_path STRING DEFAULT '@REDMINE_DATASTORE/PLUGINS/'
)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python')
HANDLER = 'run'
EXECUTE AS CALLER
AS
$$
import zipfile
import io
import os
import re
from snowflake.snowpark.files import SnowflakeFile

def run(session, stage_path):
    list_result = session.sql(f"LIST {stage_path}").collect()
    zip_files = [row['name'] for row in list_result if row['name'].lower().endswith('.zip')]

    if not zip_files:
        return "No ZIP files found on stage."

    extracted_count = 0

    stage_path_stripped = stage_path.rstrip('/')
    if '/' in stage_path_stripped.lstrip('@'):
        at_prefix = '@' if stage_path_stripped.startswith('@') else ''
        without_at = stage_path_stripped.lstrip('@')
        parts = without_at.split('/', 1)
        stage_only = at_prefix + parts[0]
        path_prefix = parts[1] + '/' if len(parts) > 1 else ''
    else:
        stage_only = stage_path_stripped
        path_prefix = ''

    for zip_file_path in zip_files:
        name_parts = zip_file_path.split('/', 1)
        relative_path = name_parts[1] if len(name_parts) > 1 else zip_file_path

        scoped_url = session.sql(
            f"SELECT BUILD_SCOPED_FILE_URL('{stage_only}', '{relative_path}') AS url"
        ).collect()[0]['URL']

        with SnowflakeFile.open(scoped_url, 'rb') as f:
            zip_data = f.read()

        zip_basename = os.path.splitext(os.path.basename(zip_file_path))[0]
        plugin_name = re.sub(r'-(main|master|develop|dev)$', '', zip_basename)

        with zipfile.ZipFile(io.BytesIO(zip_data), 'r') as zf:
            for member in zf.namelist():
                if member.endswith('/'):
                    continue

                file_data = zf.read(member)

                member_parts = member.split('/')
                if member_parts[0] == zip_basename:
                    member_rel = '/'.join(member_parts[1:])
                else:
                    member_rel = member

                member_dir = os.path.dirname(member_rel)
                member_filename = os.path.basename(member_rel)

                tmp_path = f"/tmp/{member_filename}"
                with open(tmp_path, 'wb') as tmp_file:
                    tmp_file.write(file_data)

                if member_dir:
                    dest_path = f"{stage_only}/{path_prefix}{plugin_name}/{member_dir}"
                else:
                    dest_path = f"{stage_only}/{path_prefix}{plugin_name}"
                session.file.put(
                    tmp_path,
                    dest_path,
                    auto_compress=False,
                    overwrite=True
                )

                os.remove(tmp_path)
                extracted_count += 1

        session.sql(f"REMOVE {stage_only}/{relative_path}").collect()

    return f"Extracted {extracted_count} files from {len(zip_files)} ZIP archive(s). ZIP files removed."
$$;
```

実行:

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;
CALL EXTRACT_ZIP_FILES_ON_STAGE('@REDMINE_DATASTORE/PLUGINS/');
```

### 3. サービスの再起動

プラグインを反映するためにサービスを再起動します。

```sql
ALTER SERVICE REDMINE_SERVICE_STAGING SUSPEND;
ALTER SERVICE REDMINE_SERVICE_STAGING RESUME;
```

## メンテナンス

### サービスの状態確認

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;

-- ステータス確認
CALL SYSTEM$GET_SERVICE_STATUS('REDMINE_SERVICE_STAGING');

-- ログ確認
CALL SYSTEM$GET_SERVICE_LOGS('REDMINE_SERVICE_STAGING', '0', 'redmine', 200);

-- エンドポイント確認
SHOW ENDPOINTS IN SERVICE REDMINE_SERVICE_STAGING;
```

### サービス仕様の更新

`redmine_service_spec.yaml` を編集後、ステージにアップロードして以下を実行します。

```sql
ALTER SERVICE REDMINE_SERVICE_STAGING
  FROM @REDMINE_DB.STAGING.REDMINE_DATASTORE
  SPECIFICATION_FILE = 'SPECS/redmine_service_spec.yaml';
```

### パフォーマンスモニタリング

```sql
SELECT
  TIMESTAMP,
  METRIC_NAME,
  VALUE,
  UNIT,
  CONTAINER_NAME
FROM TABLE(REDMINE_DB.STAGING.REDMINE_SERVICE_STAGING!SPCS_GET_METRICS(
  START_TIME => DATEADD('day', -1, CURRENT_TIMESTAMP())
))
WHERE METRIC_NAME IN ('container.cpu.usage', 'container.cpu.limit',
                      'container.memory.usage', 'container.memory.limit')
ORDER BY TIMESTAMP DESC;
```

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| `Could not reach host index.rubygems.org` | EAI に rubygems.org Egress ルールがない | `NWRULE_RGEGRESS` を作成し EAI に追加 |
| `Plugin not found` | ZIP 展開時のディレクトリ名に `-main` サフィックス | `EXTRACT_ZIP_FILES_ON_STAGE` が自動除去。手動時はステージ上で再配置 |
| `cannot load such file` (LoadError) | プラグインが Rails 8 / Zeitwerk 非対応 | プラグイン更新または削除 |
| `could not translate host name` | サービスロールに EAI の USAGE 権限不足 | `GRANT USAGE ON INTEGRATION ... TO ROLE ...` を実行 |
| `Unschedulable due to insufficient CPU` | Compute Pool ノード起動中 | 数分待機 |

### DNS解決失敗の復旧手順

`could not translate host name` エラーが発生した場合の復旧手順:

1. PostgreSQLインスタンスの状態を確認
   ```sql
   SHOW POSTGRES INSTANCES LIKE 'REDMINE_POSTGRESQL_STAGING';
   -- state = READY であること
   ```

2. EAIのUSAGE権限を確認（ACCOUNTADMINロールで実行）
   ```sql
   SHOW GRANTS ON INTEGRATION REDMINE_EAI_STAGING;
   -- REDMINE_SERVICE_ROLE_STAGING に USAGE が付与されていること
   -- 不足している場合:
   GRANT USAGE ON INTEGRATION REDMINE_EAI_STAGING TO ROLE REDMINE_SERVICE_ROLE_STAGING;
   ```

3. ネットワークルールの設定を確認
   ```sql
   DESCRIBE NETWORK RULE REDMINE_DB.STAGING.NWRULE_PGEGRESS;
   -- VALUE_LIST が "<PG_HOST>:5432" であること（":0" は不可）
   ```

4. サービスを再起動
   ```sql
   USE ROLE REDMINE_SERVICE_ROLE_STAGING;
   ALTER SERVICE REDMINE_SERVICE_STAGING SUSPEND;
   ALTER SERVICE REDMINE_SERVICE_STAGING RESUME;
   ```

5. 復旧確認
   ```sql
   CALL SYSTEM$GET_SERVICE_STATUS('REDMINE_SERVICE_STAGING');
   -- status = READY を確認
   CALL SYSTEM$GET_SERVICE_LOGS('REDMINE_SERVICE_STAGING', '0', 'redmine', 30);
   -- "Completed 200 OK" が出力されていること
   ```

## まとめ

本記事では、SPCS上にRedmineを構築する手順を解説しました。SPCSを使うことで、以下のメリットが得られます:

- **インフラ管理の簡素化** — Compute Pool や Managed PostgreSQL により、サーバー管理が不要
- **認証の統合** — Snowflake OAuth によりアクセス制御が自動的に適用される
- **データの永続化** — Internal Stage にファイル・プラグイン・テーマを格納し、コンテナ再起動時もデータが保持される

続編では、Redmineからメール通知を送信するためのSMTPサーバをSPCS上に構築する方法を解説します。
