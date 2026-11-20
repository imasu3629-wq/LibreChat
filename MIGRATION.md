# Zeabur → Google Cloud Run 移行計画

このドキュメントは、このフォークの LibreChat を Zeabur から Google Cloud Run へ移行するための計画です。
**APIキーや秘密情報はこのリポジトリに一切コミットしません。** 値はすべて Secret Manager / GitHub Secrets で管理します。

## 1. 現状

- Zeabur の **公式 LibreChat テンプレート** で運用中。
- テンプレートが構成しているサービス:
  - `LibreChat`（Node.js バックエンド + フロントエンド、ポート 3080）
  - `MongoDB`（会話・ユーザーデータ）
  - `Meilisearch`（会話検索）
  - （利用している場合）`rag_api` + `vectordb`（pgvector）
- 環境変数は Zeabur のサービス設定画面で管理。
- デプロイは Zeabur 側のテンプレート更新・再デプロイで実施。

## 2. 移行後の構成（目標）

| コンポーネント | 移行先 | 備考 |
| --- | --- | --- |
| LibreChat 本体 | Cloud Run（サービス `librechat`） | Artifact Registry のイメージ。最小インスタンス 1 推奨（コールドスタート・SSE 対策） |
| MongoDB | MongoDB Atlas（GCP 同リージョン） | Cloud Run からは VPC ピアリング or IP 許可リスト + Cloud NAT 固定IP |
| Meilisearch | Compute Engine（e2-small + 永続ディスク）または Meilisearch Cloud | Cloud Run はディスクが揮発のため Meilisearch 本体の常駐には不向き |
| RAG API / pgvector | Cloud Run（rag_api）+ Cloud SQL for PostgreSQL（pgvector 拡張） | 利用している場合のみ |
| ファイルアップロード | Cloud Storage（`CDN_PROVIDER` / S3 互換設定） | コンテナのローカル保存は揮発するため必須 |
| 秘密情報 | Secret Manager | Cloud Run に `--set-secrets` でマウント |
| イメージ | Artifact Registry | GitHub Actions からビルド & push |

## 3. 移行手順

### 3.1 事前準備

1. GCP プロジェクト作成、課金有効化、リージョン決定（例: `asia-northeast1`）。
2. API 有効化: Cloud Run, Artifact Registry, Secret Manager, Cloud Build, Cloud Monitoring, Cloud Logging, (必要なら) Compute Engine / Cloud SQL / Serverless VPC Access。
3. Artifact Registry に Docker リポジトリ `librechat` を作成。
4. デプロイ用サービスアカウント作成（`roles/run.admin`, `roles/artifactregistry.writer`, `roles/iam.serviceAccountUser`）。
5. GitHub Actions 用に **Workload Identity Federation** を設定（サービスアカウントキー JSON は発行しない）。
6. 実行用サービスアカウント作成（`roles/secretmanager.secretAccessor`, Cloud Storage へのアクセス権）。

### 3.2 MongoDB の移行

1. MongoDB Atlas に GCP・同リージョンでクラスタ作成。
2. Zeabur の MongoDB から `mongodump` でダンプ取得（Zeabur の公開ポート or 一時的なネットワーク公開を利用）:
   ```bash
   mongodump --uri "$ZEABUR_MONGO_URI" --archive=librechat.archive --gzip
   ```
3. Atlas へリストア:
   ```bash
   mongorestore --uri "$ATLAS_MONGO_URI" --archive=librechat.archive --gzip --nsInclude 'LibreChat.*'
   ```
4. ドキュメント件数（`users`, `conversations`, `messages`, `files` など）を比較して検証。
5. 新しい接続文字列を Secret Manager の `MONGO_URI` に登録。

### 3.3 Meilisearch の移行

Meilisearch の索引は MongoDB から再構築できるため、**データ移行は不要**（再同期で足りる）。

1. Compute Engine（または Meilisearch Cloud）に、Zeabur と**同じバージョン**の Meilisearch を立てる。
2. 新しい `MEILI_MASTER_KEY` を生成し Secret Manager に登録。
3. Cloud Run からは内部 IP（Serverless VPC Access コネクタ / Direct VPC egress 経由）で接続し、`MEILI_HOST` を設定。
4. LibreChat 起動時の同期（`SEARCH=true`）で索引が再構築されることを確認。必要なら `MEILI_NO_SYNC` を外して再同期させる。

### 3.4 RAG API / ファイル（利用している場合）

1. Cloud SQL for PostgreSQL を作成し `CREATE EXTENSION vector;`。
2. Zeabur の vectordb から `pg_dump` → Cloud SQL へ `pg_restore`。
3. `rag_api` を別 Cloud Run サービスとしてデプロイし、`RAG_API_URL` を設定。
4. アップロード済みファイルを Cloud Storage へコピーし、ファイル保存先を Cloud Storage に切り替え。

### 3.5 環境変数・秘密情報の扱い

- Zeabur の環境変数を一覧化し、**秘密情報**と**非秘密設定**に分類する。
  - 秘密情報（Secret Manager）: `MONGO_URI`, `MEILI_MASTER_KEY`, `CREDS_KEY`, `CREDS_IV`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, 各プロバイダの API キー（`OPENAI_API_KEY`, `ANTHROPIC_API_KEY` 等）, OAuth クライアントシークレット, `SENTRY_DSN`。
  - 非秘密設定（Cloud Run 環境変数）: `HOST=0.0.0.0`, `DOMAIN_CLIENT`, `DOMAIN_SERVER`, `MEILI_HOST`, `SEARCH`, `ALLOW_REGISTRATION` 等。
- **`CREDS_KEY` / `CREDS_IV` / `JWT_SECRET` / `JWT_REFRESH_SECRET` は Zeabur と同じ値を引き継ぐ。** 変えるとユーザーが保存した API キーの復号やログインセッションが壊れる。
- `PORT` は Cloud Run が注入するため、コンテナは `PORT`（既定 3080）で待ち受ける。Cloud Run 側の `--port=3080` で揃える。
- `librechat.yaml` を使っている場合は Secret Manager にファイルとして登録し、ボリュームマウント + `CONFIG_PATH` で読み込む。
- `.env` はローカル専用で `.gitignore`（`.env*`）により git 管理外。リポジトリには `.env.example` 以外置かない。

### 3.6 デプロイ

1. `.github/workflows/deploy-cloudrun.yml` のプレースホルダ（`GCP_PROJECT_ID`, `GCP_REGION`, `GCP_WIF_PROVIDER`, `GCP_DEPLOY_SA`, `GCP_RUNTIME_SA` など）を GitHub の Secrets / Variables に登録。
2. 初回は `workflow_dispatch` で手動実行し、Cloud Run の自動生成 URL（`*.run.app`）で動作確認。
3. 確認項目: ログイン、会話作成・ストリーミング、会話検索、ファイルアップロード、既存ユーザーの保存済み API キーが使えること。

### 3.7 切り替え（カットオーバー）

1. **前日まで**: DNS の TTL を 300 秒程度に下げる。Cloud Run にカスタムドメイン（またはロードバランサ + マネージド証明書）を設定。
2. **メンテナンス開始**: Zeabur 側で新規登録を停止（`ALLOW_REGISTRATION=false`）、可能なら書き込み停止を告知。
3. 最終差分として MongoDB を再度 `mongodump` → `mongorestore --drop`。
4. Cloud Run の `DOMAIN_CLIENT` / `DOMAIN_SERVER` を本番ドメインに更新して再デプロイ。
5. DNS を Cloud Run に向ける。OAuth プロバイダのリダイレクト URI も更新。
6. スモークテスト（3.6 の確認項目）。
7. **ロールバック手順**: 問題があれば DNS を Zeabur に戻す。Zeabur 環境は最低 1〜2 週間停止せず保持。
8. 安定稼働を確認後、Zeabur のサービスとデータを削除。

## 4. 移行後の監視構成

```
LibreChat (Cloud Run)
 ├─ Sentry SDK (Node backend) ──► Sentry ──► Devin Bot ──► 仕分け / 修正 PR 作成
 │                                                              │
 │                                                              ▼
 │                                                     PR レビュー & マージ (main)
 │                                                              │
 │                     GitHub Actions (deploy-cloudrun.yml) ◄───┘
 │                                │
 │                                ▼ 自動再デプロイ
 └─ Cloud Logging / Cloud Monitoring
       ├─ Uptime check（死活監視）
       └─ ログベースアラート ──► Slack
```

### 4.1 Sentry（アプリケーションエラー）

- Node バックエンド（`api/`）に `@sentry/node` を導入し、サーバー起動の最初で `Sentry.init()` する。
- DSN は `SENTRY_DSN` として Secret Manager から注入（コードには書かない）。
- `environment`（production / staging）と `release`（Git SHA）を設定し、どのデプロイで発生したエラーか追えるようにする。
- 送信前に `beforeSend` でリクエストボディ・Authorization ヘッダ・会話本文などの個人情報/秘密情報を除去する。

### 4.2 Devin Bot（仕分けと PR 作成）

- Sentry のアラート（Issue Alert / Webhook または Slack 経由）を Devin Bot に連携。
- Devin Bot が新規エラーを仕分け（重複・既知・ノイズ・要対応）し、要対応のものは原因調査して **このフォークに修正 PR を作成**。
- PR は必ず人がレビューしてからマージ（自動マージはしない）。Devin に渡す権限はこのリポジトリへの PR 作成に限定する。

### 4.3 GCP の死活監視とログアラート → Slack

- **Uptime check**: Cloud Run の公開 URL の `/health`（または `/`）を 1 分間隔で監視。失敗時にアラート。
- **ログベースアラート**（Cloud Logging）:
  - `severity>=ERROR` の急増
  - HTTP 5xx の割合
  - コンテナの OOM / 起動失敗（`Container terminated`, `Memory limit exceeded`）
  - MongoDB / Meilisearch への接続エラー
- **メトリクスアラート**: リクエストレイテンシ p95、インスタンス数の上限到達、CPU/メモリ使用率。
- 通知チャネルは **Slack**（Cloud Monitoring の Slack 通知チャネル）。

### 4.4 マージ後の自動再デプロイ

- `main` への push（= PR のマージ）で `.github/workflows/deploy-cloudrun.yml` が起動し、イメージをビルド → Artifact Registry へ push → Cloud Run に新リビジョンをデプロイ。
- 失敗時は Cloud Run の前リビジョンにトラフィックを戻す（`gcloud run services update-traffic librechat --to-revisions=<prev>=100`）。
- フォークを上流（danny-avila/LibreChat）に追従させる際も、上流の変更を取り込む PR をマージすれば同じ流れでデプロイされる。
