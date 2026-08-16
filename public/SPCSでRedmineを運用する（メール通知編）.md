---
title: SPCSでRedmineを運用する（メール通知編）
tags:
  - Snowflake
  - SPCS
  - Redmine
  - Docker
  - SMTP
private: false
updated_at: '2026-08-17T00:48:21+09:00'
id: ab7ee0e3164072ad3169
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

[構築編](https://qiita.com/)では SPCS 上に Redmine を稼働させました。本記事（メール通知編）では、Redmine のチケット更新時にメール通知を送信する仕組みを構築します。

SPCSコンテナから外部SMTPサーバーへ直接通信することはできないため、Snowflake の `SYSTEM$SEND_EMAIL` を利用します。具体的には、SPCS上にSMTPリレーサーバーを別コンテナとして構築し、Redmine → SMTPリレー → `SYSTEM$SEND_EMAIL` という経路でメールを配信します。

なお、本記事の作成には以下のAIツールを活用しています:

1. **Kiro** — 記事の作成支援
2. **CoCo (Cortex Code)** — SQLや設定ファイルの作成支援

## アーキテクチャ

```
[Redmine Service (SPCS)]
  redmine container (port 3000)
       │
       │ SMTP (port 2525, プライベートエンドポイント / TCP)
       ▼
[SMTP Service (SPCS)]
  smtp-relay container (aiosmtpd + asyncio)
       │
       │ Snowflake Connector (OAuth token + warehouse)
       ▼
  SYSTEM$SEND_EMAIL (via REDMINE_EMAIL_INT)
       │
       ▼
  [受信者のメールボックス]
```

### 処理フロー

1. Redmine がチケット更新時に SMTP リレーへメール送信
2. SMTP リレーが即座に `250 OK` を返却（Redmine 側のタイムアウトを防止）
3. `asyncio.ensure_future` で非同期配送タスクを登録（fire-and-forget）
4. `run_in_executor` で Snowflake Connector 経由で `SYSTEM$SEND_EMAIL` を呼び出し
5. 失敗時は最大3回リトライ（指数バックオフ）

### コンポーネント一覧

| リソース | 名前 | 説明 |
|---------|------|------|
| Service | `SMTP_SERVICE_STAGING` | SMTPリレーコンテナサービス |
| Notification Integration | `REDMINE_EMAIL_INT` | `SYSTEM$SEND_EMAIL` 用のメール統合 |
| Stored Procedure | `SEND_REDMINE_EMAIL` | メール送信用プロシージャ（動作確認用） |
| Image | `smtp-relay:latest` | Python 3.12 + aiosmtpd |

## 前提条件

- [構築編](https://qiita.com/)のセットアップが完了していること
- `REDMINE_COMPUTEPOOL_STAGING` が稼働中
- `REDMINE_SERVICE_ROLE_STAGING` が存在
- `COMPUTE_WH` ウェアハウスが存在
- Docker CLI（イメージのビルド/push 用）

## 設計上の注意点

### なぜSMTPリレーが必要か

SPCSコンテナから外部の SMTP サーバー（Gmail, SendGrid 等）へ直接通信するには、External Access Integration で許可する必要がありますが、Snowflake には `SYSTEM$SEND_EMAIL` というメール送信機能が組み込まれています。しかし `SYSTEM$SEND_EMAIL` は SQL 経由でしか呼び出せないため、Redmine が期待する SMTP インターフェースとの間にリレーサーバーを配置する構成としました。

### 非同期配送の理由

Redmine は SMTP 送信完了を待ってからレスポンスを返します。`SYSTEM$SEND_EMAIL` の呼び出しには Snowflake Connector の接続確立を含めて数秒かかるため、同期的に処理すると Redmine のリクエストがタイムアウトするリスクがあります。そのため、SMTP リレーは即座に `250 OK` を返し、配送処理をバックグラウンドで行います。

Redmine の configuration.yml の中で delivery_method に async_smtp を設定すれば、リレーサーバ側で非同期処理を持たずに Redmine の設定のみで非同期配送を実現できるようですが未検証です。リレーサーバー側に非同期処理を持つことで、Redmine 以外の SPCS アプリからも、即座にレスポンスを返す、非同期配送機能を持った SMTP リレーサーバーとして利用可能です。

### ウェアハウスの必要性

`SYSTEM$SEND_EMAIL` はシステム関数ですが、実行にウェアハウスが必要です。SPCSコンテナから `/snowflake/session/token` で OAuth 接続する場合、サービスの `QUERY_WAREHOUSE` 設定はセッションに適用されません。そのため、Python Connector の接続パラメータで `warehouse` を明示的に指定しています。

## 構築手順

### 1. Notification Integration の作成

メール送信に使用する Notification Integration を作成します。

```sql
USE ROLE ACCOUNTADMIN;
USE DATABASE REDMINE_DB;
USE SCHEMA STAGING;

CREATE OR REPLACE NOTIFICATION INTEGRATION REDMINE_EMAIL_INT
  TYPE = EMAIL
  ENABLED = TRUE;
```

:::note info
`ALLOWED_RECIPIENTS` を省略すると、アカウント内のメール検証済み全ユーザーに送信可能になります。送信先を制限したい場合は明示的に指定してください。
:::

### 2. 権限付与

サービスロールに必要な権限を付与します。

```sql
USE ROLE ACCOUNTADMIN;

-- Notification Integrationの利用権限
GRANT USAGE ON INTEGRATION REDMINE_EMAIL_INT TO ROLE REDMINE_SERVICE_ROLE_STAGING;

-- ストアドプロシージャの作成権限
GRANT CREATE PROCEDURE ON SCHEMA REDMINE_DB.STAGING TO ROLE REDMINE_SERVICE_ROLE_STAGING;

-- SYSTEM$SEND_EMAIL の実行にウェアハウスが必須
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE REDMINE_SERVICE_ROLE_STAGING;
```

### 3. メール送信用ストアドプロシージャの作成（動作確認用）

SMTPリレーを使わずに直接メール送信をテストするためのプロシージャです。

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;
USE DATABASE REDMINE_DB;
USE SCHEMA STAGING;

CREATE OR REPLACE PROCEDURE SEND_REDMINE_EMAIL(
  RECIPIENTS VARCHAR,
  SUBJECT VARCHAR,
  BODY VARCHAR,
  MIME_TYPE VARCHAR DEFAULT 'text/plain'
)
RETURNS VARCHAR
LANGUAGE SQL
AS
BEGIN
  IF (:MIME_TYPE = 'text/html') THEN
    CALL SYSTEM$SEND_EMAIL(
      'REDMINE_EMAIL_INT',
      :RECIPIENTS,
      :SUBJECT,
      :BODY,
      :MIME_TYPE
    );
  ELSE
    CALL SYSTEM$SEND_EMAIL(
      'REDMINE_EMAIL_INT',
      :RECIPIENTS,
      :SUBJECT,
      :BODY
    );
  END IF;
  RETURN 'OK';
END;
```

### 4. SMTP リレーサーバーの実装

以下の3ファイルを作成します。

#### ディレクトリ構成

```
smtp-relay/
├── smtp_server.py       # SMTP リレーサーバー本体
├── requirements.txt     # Python 依存パッケージ
└── Dockerfile           # コンテナイメージ定義
```

#### requirements.txt

```text:requirements.txt
aiosmtpd==1.4.6
snowflake-connector-python==3.12.3
```

#### Dockerfile

```dockerfile:Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY smtp_server.py .
CMD ["python", "smtp_server.py"]
```

#### smtp_server.py

```python:smtp_server.py
"""
SPCS SMTP Relay Server
Receives SMTP messages from Redmine and forwards via SYSTEM$SEND_EMAIL.
Uses asyncio.create_task for non-blocking delivery within the Controller's event loop.
"""
import asyncio
import logging
import os
import time
from concurrent.futures import ThreadPoolExecutor
from email import policy
from email.parser import BytesParser

from aiosmtpd.controller import Controller
from aiosmtpd.smtp import SMTP as SMTPServer
import snowflake.connector

logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')
logger = logging.getLogger(__name__)

SNOWFLAKE_ACCOUNT = os.environ.get('SNOWFLAKE_ACCOUNT', '')
SNOWFLAKE_HOST = os.environ.get('SNOWFLAKE_HOST', '')
SNOWFLAKE_DATABASE = 'REDMINE_DB'
SNOWFLAKE_SCHEMA = 'STAGING'
TOKEN_FILE = '/snowflake/session/token'
MAX_RETRIES = 3
BACKOFF_BASE = 2

executor = ThreadPoolExecutor(max_workers=2)


def get_snowflake_connection():
    token = open(TOKEN_FILE).read().strip()
    params = {
        'account': SNOWFLAKE_ACCOUNT,
        'authenticator': 'oauth',
        'token': token,
        'database': SNOWFLAKE_DATABASE,
        'schema': SNOWFLAKE_SCHEMA,
        'warehouse': os.environ.get('SNOWFLAKE_WAREHOUSE', 'COMPUTE_WH'),
    }
    if SNOWFLAKE_HOST:
        params['host'] = SNOWFLAKE_HOST
    logger.info(f"Connecting to Snowflake account={SNOWFLAKE_ACCOUNT}")
    conn = snowflake.connector.connect(**params)
    logger.info("Snowflake connection established")
    return conn


def _send_email_sync(recipients, subject, body, mime_type):
    """Send email via SYSTEM$SEND_EMAIL directly (no warehouse required)."""
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            conn = get_snowflake_connection()
            cur = conn.cursor()
            logger.info(f"Calling SYSTEM$SEND_EMAIL to={recipients}")
            if mime_type == 'text/html':
                cur.execute(
                    "CALL SYSTEM$SEND_EMAIL(%s, %s, %s, %s, %s)",
                    ('REDMINE_EMAIL_INT', recipients, subject, body, mime_type)
                )
            else:
                cur.execute(
                    "CALL SYSTEM$SEND_EMAIL(%s, %s, %s, %s)",
                    ('REDMINE_EMAIL_INT', recipients, subject, body)
                )
            cur.close()
            conn.close()
            logger.info(f"Sent email to={recipients} subject={subject}")
            return
        except Exception as e:
            logger.error(f"Attempt {attempt}/{MAX_RETRIES} failed: {type(e).__name__}: {e}")
            if attempt < MAX_RETRIES:
                time.sleep(BACKOFF_BASE ** attempt)
            else:
                logger.error(f"Giving up on email to={recipients}")


async def _deliver_email(recipients, subject, body, mime_type):
    """Async wrapper: dispatches blocking send to executor thread."""
    loop = asyncio.get_event_loop()
    try:
        await loop.run_in_executor(
            executor, _send_email_sync, recipients, subject, body, mime_type
        )
    except Exception as e:
        logger.error(f"Delivery task error: {type(e).__name__}: {e}")


class RelayHandler:
    async def handle_RCPT(self, server, session, envelope, address, rcpt_options):
        envelope.rcpt_tos.append(address)
        return '250 OK'

    async def handle_DATA(self, server, session, envelope):
        msg = BytesParser(policy=policy.default).parsebytes(envelope.content)
        recipients = ', '.join(envelope.rcpt_tos)
        subject = str(msg.get('Subject', '(no subject)'))
        body = msg.get_body(preferencelist=('html', 'plain'))
        content = str(body.get_content()) if body else ''
        mime_type = 'text/html' if body and body.get_content_type() == 'text/html' else 'text/plain'

        # Fire-and-forget: schedule delivery in the same event loop
        asyncio.ensure_future(_deliver_email(recipients, subject, content, mime_type))
        logger.info(f"Accepted email to={recipients} subject={subject}")

        return '250 Message accepted for delivery'


def main():
    logger.info("Starting SMTP relay server on port 2525 (no auth required)")

    handler = RelayHandler()

    class NoAuthController(Controller):
        def factory(self):
            return SMTPServer(
                self.handler,
                hostname=self.hostname,
                auth_required=False,
                auth_exclude_mechanism=['LOGIN', 'PLAIN'],
            )

    controller = NoAuthController(handler, hostname='0.0.0.0', port=2525)
    controller.start()

    logger.info("SMTP server listening on 0.0.0.0:2525")
    try:
        while True:
            time.sleep(3600)
    except KeyboardInterrupt:
        controller.stop()


if __name__ == '__main__':
    main()
```

### 5. Docker イメージのビルドとプッシュ

```bash
cd smtp-relay

# linux/amd64 向けにビルド（Apple Silicon の場合は --platform 指定が必要）
docker build --platform linux/amd64 -t smtp-relay:latest ./

# タグ付け（SHOW IMAGE REPOSITORIES の repository_url を使用）
docker tag smtp-relay:latest \
  <YOUR_SNOWFLAKE_ACCOUNT>.registry.snowflakecomputing.com/redmine_db/staging/docker_repo/smtp-relay:latest

# プッシュ
docker push \
  <YOUR_SNOWFLAKE_ACCOUNT>.registry.snowflakecomputing.com/redmine_db/staging/docker_repo/smtp-relay:latest
```

### 6. SMTP サービスの作成

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;
USE DATABASE REDMINE_DB;
USE SCHEMA STAGING;

CREATE SERVICE SMTP_SERVICE_STAGING
  IN COMPUTE POOL REDMINE_COMPUTEPOOL_STAGING
  EXTERNAL_ACCESS_INTEGRATIONS = (REDMINE_EAI_STAGING)
  QUERY_WAREHOUSE = COMPUTE_WH
  MIN_INSTANCES = 1
  MAX_INSTANCES = 1
  FROM SPECIFICATION $$
  spec:
    containers:
    - name: smtp-relay
      image: /REDMINE_DB/STAGING/DOCKER_REPO/smtp-relay:latest
      env:
        SNOWFLAKE_ACCOUNT: "<YOUR_SNOWFLAKE_ACCOUNT>"
        SNOWFLAKE_HOST: "<YOUR_SNOWFLAKE_ACCOUNT>.<REGION>.snowflakecomputing.com"
        SNOWFLAKE_WAREHOUSE: "COMPUTE_WH"
      resources:
        requests:
          cpu: 0.25
          memory: 512Mi
        limits:
          cpu: 1
          memory: 1Gi
    endpoints:
    - name: smtp
      port: 2525
      protocol: TCP
    platformMonitor:
      metricConfig:
        groups:
          - system
  $$;
```

`<YOUR_SNOWFLAKE_ACCOUNT>` と `<REGION>` は自身の環境に合わせて置き換えてください。

:::note info
エンドポイントの `protocol: TCP` により、同一 SPCS 内のサービス間でのみアクセス可能なプライベートエンドポイントになります。外部からはアクセスできません。
:::

### 7. DNS 名の確認

サービス作成後、Redmine から接続するための DNS 名を確認します。

```sql
SHOW SERVICES IN SCHEMA REDMINE_DB.STAGING;
```

`dns_name` 列に表示される値（例: `smtp-service-staging.staging.redmine-db.snowflakecomputing.internal`）を、次のステップで Redmine の SMTP 設定に使用します。

### 8. Redmine 側のメール設定

`configuration.yml` に以下を追加し、`@REDMINE_DATASTORE/CONFIG/` にアップロードします。

```yaml:configuration.yml
production:
  email_delivery:
    delivery_method: :smtp
    smtp_settings:
      address: "smtp-service-staging.staging.redmine-db.snowflakecomputing.internal"
      port: 2525
      enable_starttls_auto: false
```

:::note info
同一 SPCS 内のプライベートエンドポイント通信のため、SMTP 認証と STARTTLS は不要です。
:::

設定ファイルのアップロード後、Redmine サービスを再起動して反映します。

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;
ALTER SERVICE REDMINE_SERVICE_STAGING SUSPEND;
ALTER SERVICE REDMINE_SERVICE_STAGING RESUME;
```

## 動作確認

### ストアドプロシージャの直接テスト

まずは `SYSTEM$SEND_EMAIL` が正常に動作するか確認します。

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;
CALL SEND_REDMINE_EMAIL(
  '<YOUR_EMAIL_ADDRESS>',
  'テストメール',
  'これはSPCS SMTPリレー経由のテストメールです。'
);
```

:::note warn
送信先のメールアドレスは、Snowflake アカウントのユーザーに紐づいた検証済みのメールアドレスである必要があります。
:::

### SMTP サービスのログ確認

```sql
SELECT SYSTEM$GET_SERVICE_LOGS('SMTP_SERVICE_STAGING', 0, 'smtp-relay', 100);
```

正常に動作している場合、以下のようなログが確認できます:

```
Starting SMTP relay server on port 2525 (no auth required)
SMTP server listening on 0.0.0.0:2525
Accepted email to=user@example.com subject=テスト
Connecting to Snowflake account=...
Snowflake connection established
Sent email to=user@example.com subject=テスト
```

### Redmine からの送信テスト

1. Redmine の管理画面 → 設定 → メール通知 を開く
2. 画面右下の「テストメールを送信」をクリック
3. SMTP サービスのログに `Accepted email` と `Sent email` が出力されることを確認

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| `Connection refused` (Redmine側) | SMTP サービスが起動していない | `CALL SYSTEM$GET_SERVICE_STATUS('SMTP_SERVICE_STAGING')` で確認し、必要なら RESUME |
| `Snowflake connection failed` | OAuth トークンの期限切れ or アカウント情報の誤り | 環境変数 `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_HOST` を確認。サービスを再起動 |
| `Warehouse 'COMPUTE_WH' does not exist` | ウェアハウス名の誤り or 権限不足 | `GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE REDMINE_SERVICE_ROLE_STAGING` を実行 |
| メールが届かない（ログに `Sent email` あり） | 受信者のメールアドレスが未検証 | Snowflake で対象ユーザーのメールアドレスが検証済みか確認 |
| `REDMINE_EMAIL_INT does not exist` | Integration が未作成 or 権限不足 | `GRANT USAGE ON INTEGRATION REDMINE_EMAIL_INT TO ROLE REDMINE_SERVICE_ROLE_STAGING` を実行 |

## メンテナンス

### サービスの再起動

```sql
USE ROLE REDMINE_SERVICE_ROLE_STAGING;
ALTER SERVICE SMTP_SERVICE_STAGING SUSPEND;
ALTER SERVICE SMTP_SERVICE_STAGING RESUME;
```

### パフォーマンスモニタリング

```sql
SELECT
  TIMESTAMP,
  METRIC_NAME,
  VALUE,
  UNIT,
  CONTAINER_NAME
FROM TABLE(REDMINE_DB.STAGING.SMTP_SERVICE_STAGING!SPCS_GET_METRICS(
  START_TIME => DATEADD('day', -1, CURRENT_TIMESTAMP())
))
WHERE METRIC_NAME IN ('container.cpu.usage', 'container.cpu.limit',
                      'container.memory.usage', 'container.memory.limit')
ORDER BY TIMESTAMP DESC;
```

## まとめ

本記事では、SPCS 上に SMTP リレーサーバーを構築し、Redmine からメール通知を送信する仕組みを解説しました。

- **SPCSの制約を回避** — 外部 SMTP サーバーへの直接通信の代わりに、`SYSTEM$SEND_EMAIL` を活用
- **非同期配送** — Redmine 側のタイムアウトを防止しつつ、リトライによる配送信頼性を確保
- **プライベート通信** — Service-to-Service の TCP エンドポイントにより、SMTP 認証なしでセキュアに通信
- **同一 Compute Pool** — 既存の `REDMINE_COMPUTEPOOL_STAGING` を共有し、追加のインフラコストを最小化
